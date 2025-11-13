# Документация пайплайнов агентов smolagents

## Содержание

1. [Введение](#введение)
2. [Архитектура агентов](#архитектура-агентов)
3. [Базовый пайплайн выполнения](#базовый-пайплайн-выполнения)
4. [Пайплайн CodeAgent](#пайплайн-codeagent)
5. [Пайплайн ToolCallingAgent](#пайплайн-toolcallingagent)
6. [Пайплайн с планированием](#пайплайн-с-планированием)
7. [Пайплайн с управляемыми агентами](#пайплайн-с-управляемыми-агентами)
8. [Обработка ошибок](#обработка-ошибок)

---

## Введение

Smolagents использует модульную архитектуру с несколькими типами агентов и режимами работы. Этот документ описывает все возможные пайплайны выполнения, последовательность вызовов промтов и передачу данных между компонентами.

### Типы агентов

1. **CodeAgent** - решает задачи через написание и выполнение Python кода
2. **ToolCallingAgent** - решает задачи через JSON-based вызовы инструментов
3. **MultiStepAgent** - абстрактный базовый класс для обоих типов

### Режимы работы

- **Базовый режим** - простое выполнение шагов без планирования
- **С планированием** - агент создает и обновляет план решения задачи
- **С управляемыми агентами** - делегирование подзадач другим агентам
- **Streaming режим** - генерация результатов в реальном времени

---

## Архитектура агентов

### Иерархия классов

```
MultiStepAgent (ABC)
├── properties:
│   ├── tools: dict[str, Tool]
│   ├── managed_agents: dict[str, ManagedAgent]
│   ├── model: Model
│   ├── memory: AgentMemory
│   ├── max_steps: int
│   ├── planning_interval: int | None
│   └── prompt_templates: PromptTemplates
│
├── ToolCallingAgent
│   ├── использует model.generate() с tools_to_call_from
│   ├── парсит JSON tool calls
│   └── поддерживает параллельное выполнение инструментов
│
└── CodeAgent
    ├── генерирует Python код
    ├── парсит код через parse_code_blobs()
    ├── выполняет через PythonExecutor
    └── опционально: use_structured_outputs_internally=True
```

### Ключевые компоненты

**Файл**: `src/smolagents/agents.py`

| Класс/Компонент | Назначение | Строки |
|-----------------|-----------|---------|
| `MultiStepAgent` | Базовый класс с основным циклом | 266-1188 |
| `ToolCallingAgent` | Tool calling реализация | 1190-1477 |
| `CodeAgent` | Code-based реализация | 1479-1778 |
| `AgentMemory` | Хранилище истории выполнения | memory.py |
| `ActionStep` | Шаг выполнения действия | memory.py:51-150 |
| `PlanningStep` | Шаг планирования | memory.py:153-183 |

---

## Базовый пайплайн выполнения

Этот пайплайн используется всеми типами агентов как основа.

### Диаграмма потока

```
┌─────────────────────────────────────────────────────────────────┐
│                    agent.run(task, ...)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ЭТАП 1: ИНИЦИАЛИЗАЦИЯ                                          │
│  ├─ Установить task в memory                                   │
│  ├─ Сбросить память (если reset=True)                          │
│  ├─ Добавить system_prompt в память                            │
│  ├─ Добавить TaskStep(task, images) в memory.steps             │
│  └─ Если CodeAgent: инициализировать PythonExecutor            │
│                                                                 │
│  ЭТАП 2: ВЫБОР РЕЖИМА ВЫПОЛНЕНИЯ                               │
│  ├─ Если stream=True:                                          │
│  │  └─ return _run_stream() → генератор событий               │
│  └─ Если stream=False:                                         │
│     └─ steps = list(_run_stream())                             │
│        └─ return output или RunResult                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              _run_stream() - ОСНОВНОЙ ЦИКЛ                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  for step_number in range(1, max_steps + 1):                   │
│                                                                 │
│    ┌──────────────────────────────────────────────┐            │
│    │ ПРОВЕРКА ПРЕРЫВАНИЯ                          │            │
│    │ if self.interrupt_switch:                    │            │
│    │   raise AgentError("Agent interrupted")      │            │
│    └──────────────────────────────────────────────┘            │
│                    │                                            │
│                    ▼                                            │
│    ┌──────────────────────────────────────────────┐            │
│    │ ПЛАНИРОВАНИЕ (опционально)                   │            │
│    │ if planning_interval и                       │            │
│    │    (step=1 или (step-1) % interval == 0):   │            │
│    │                                              │            │
│    │   planning_step = _generate_planning_step() │            │
│    │   Yield PlanningStep                         │            │
│    └──────────────────────────────────────────────┘            │
│                    │                                            │
│                    ▼                                            │
│    ┌──────────────────────────────────────────────┐            │
│    │ СОЗДАНИЕ ActionStep                          │            │
│    │ action_step = ActionStep(                    │            │
│    │   step_number, timing, observations_images)  │            │
│    └──────────────────────────────────────────────┘            │
│                    │                                            │
│                    ▼                                            │
│    ┌──────────────────────────────────────────────┐            │
│    │ ВЫПОЛНЕНИЕ ШАГА                              │            │
│    │ try:                                         │            │
│    │   for output in _step_stream(action_step):  │            │
│    │     Yield output (deltas, tool calls, etc)  │            │
│    │     if ActionOutput.is_final_answer:        │            │
│    │       returned_final_answer = True          │            │
│    │       break                                 │            │
│    │                                              │            │
│    │ except AgentGenerationError:                │            │
│    │   raise (критическая ошибка)                │            │
│    │ except AgentError as e:                     │            │
│    │   action_step.error = e                     │            │
│    │                                              │            │
│    │ finally:                                    │            │
│    │   memory.steps.append(action_step)          │            │
│    │   Yield action_step                         │            │
│    └──────────────────────────────────────────────┘            │
│                    │                                            │
│                    ▼                                            │
│    if returned_final_answer:                                   │
│      break  ◄── выход из цикла                                 │
│                                                                 │
│  ┌──────────────────────────────────────────────┐              │
│  │ ОБРАБОТКА МАКСИМУМА ШАГОВ                    │              │
│  │ if not returned_final_answer:                │              │
│  │   final_answer = _handle_max_steps_reached() │              │
│  │   Yield FinalAnswerStep(final_answer)        │              │
│  └──────────────────────────────────────────────┘              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Используемые промты

| Этап | Промт | Откуда |
|------|-------|--------|
| Инициализация | `system_prompt` | Из YAML файла агента |
| Планирование | `planning.initial_plan` | Если step == 1 и planning_interval |
| Планирование | `planning.update_plan_*` | Если нужно обновить план |
| Макс. шагов | `final_answer.pre_messages`, `final_answer.post_messages` | При достижении max_steps |

### Передаваемые данные

**Вход метода `run()`**:
```python
{
  "task": str,              # Задача для выполнения
  "stream": bool,           # Режим потоковой передачи
  "reset": bool,            # Сбросить память
  "images": list,           # Изображения для задачи
  "additional_args": dict,  # Дополнительные переменные состояния
  "max_steps": int          # Переопределить max_steps
}
```

**Выход метода `run()`** (если stream=False):
```python
# Вариант 1: return_full_result=False (по умолчанию)
output: Any  # Финальный ответ агента

# Вариант 2: return_full_result=True
RunResult {
  "output": Any,
  "token_usage": TokenUsage,
  "state": "success" | "max_steps_error"
}
```

---

## Пайплайн CodeAgent

CodeAgent решает задачи через генерацию и выполнение Python кода. Использует цикл Thought → Code → Observation.

### Диаграмма потока CodeAgent._step_stream()

```
┌──────────────────────────────────────────────────────────────────┐
│              _step_stream(action_step) - CodeAgent               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ЭТАП 1: ПОДГОТОВКА СООБЩЕНИЙ ДЛЯ LLM                           │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ memory_messages = write_memory_to_messages()           │     │
│  │                                                        │     │
│  │ Преобразование memory.steps → ChatMessage[]           │     │
│  │ [SystemPromptStep] → ChatMessage(SYSTEM, system_prompt)│     │
│  │ [TaskStep]         → ChatMessage(USER, task)          │     │
│  │ [ActionStep]       → ChatMessage(ASSISTANT, thought+code) │  │
│  │                      ChatMessage(USER, "Observation: output")│
│  │ [PlanningStep]     → ChatMessage(USER, plan)          │     │
│  │                                                        │     │
│  │ memory_step.model_input_messages = messages            │     │
│  └────────────────────────────────────────────────────────┘     │
│                         │                                       │
│                         ▼                                       │
│  ЭТАП 2: ГЕНЕРАЦИЯ ВЫВОДА ОТ LLM                                │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ ПРОМТ: system_prompt из code_agent.yaml               │     │
│  │ (содержит примеры Thought/Code/Observation)           │     │
│  │                                                        │     │
│  │ Переменные в промте:                                  │     │
│  │ - {{tools}}: сгенерированные описания через          │     │
│  │              tool.to_code_prompt()                    │     │
│  │ - {{authorized_imports}}: список разрешенных модулей │     │
│  │ - {{code_block_opening_tag}}: "```py" или "```python"│     │
│  │ - {{code_block_closing_tag}}: "```"                  │     │
│  │                                                        │     │
│  │ if stream_outputs:                                    │     │
│  │   for event in model.generate_stream(messages):       │     │
│  │     Yield ChatMessageStreamDelta                      │     │
│  │   chat_message = agglomerate_deltas()                 │     │
│  │ else:                                                 │     │
│  │   chat_message = model.generate(messages)            │     │
│  │                                                        │     │
│  │ memory_step.model_output = chat_message.content       │     │
│  │ memory_step.token_usage = chat_message.token_usage   │     │
│  └────────────────────────────────────────────────────────┘     │
│                         │                                       │
│                         ▼                                       │
│  ЭТАП 3: ПАРСИНГ КОДА ИЗ ВЫВОДА                                │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ code_blobs = parse_code_blobs(                        │     │
│  │   chat_message.content,                               │     │
│  │   code_block_opening_tag,                             │     │
│  │   code_block_closing_tag                              │     │
│  │ )                                                      │     │
│  │                                                        │     │
│  │ if not code_blobs:                                    │     │
│  │   raise AgentParsingError(                            │     │
│  │     "No code found in output"                         │     │
│  │   )                                                    │     │
│  │                                                        │     │
│  │ code_to_execute = "\n".join(code_blobs)               │     │
│  │ memory_step.code = code_to_execute                    │     │
│  └────────────────────────────────────────────────────────┘     │
│                         │                                       │
│                         ▼                                       │
│  ЭТАП 4: ВЫПОЛНЕНИЕ КОДА ЧЕРЕЗ PythonExecutor                  │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ result = python_executor(code_to_execute)             │     │
│  │                                                        │     │
│  │ PythonExecutor:                                       │     │
│  │ ├─ Имеет персистентное состояние (self.state)        │     │
│  │ ├─ Содержит инструменты как функции                  │     │
│  │ ├─ Выполняет exec(code, state)                        │     │
│  │ ├─ Перехватывает print() → observations              │     │
│  │ ├─ Перехватывает final_answer() → is_final_answer    │     │
│  │ └─ Применяет authorized_imports                       │     │
│  │                                                        │     │
│  │ if "final_answer" вызвана:                            │     │
│  │   is_final_answer = True                              │     │
│  │   output = результат final_answer                     │     │
│  │ else:                                                 │     │
│  │   is_final_answer = False                             │     │
│  │   output = observations (все print())                 │     │
│  │                                                        │     │
│  │ memory_step.observations = output                     │     │
│  │                                                        │     │
│  │ Yield ActionOutput(                                   │     │
│  │   output=output,                                      │     │
│  │   is_final_answer=is_final_answer                     │     │
│  │ )                                                      │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Пример выполнения CodeAgent

**Задача**: "What is 15 multiplied by 7?"

**Шаг 1: Инициализация**
```
memory.steps = [
  SystemPromptStep(system_prompt),
  TaskStep("What is 15 multiplied by 7?")
]
```

**Шаг 2: Генерация (step 1)**

Отправляемые сообщения:
```
[
  ChatMessage(SYSTEM, "You are an expert assistant who can solve any task using code blobs..."),
  ChatMessage(USER, "What is 15 multiplied by 7?")
]
```

Ответ LLM:
```
Thought: I need to calculate 15 * 7 using Python.
```py
result = 15 * 7
print(result)
```
```

Парсинг → `code = "result = 15 * 7\nprint(result)"`

**Шаг 3: Выполнение кода**
```python
python_executor.execute("result = 15 * 7\nprint(result)")
# → observations = "105"
# → is_final_answer = False
```

**Шаг 4: Observation добавлена в память**
```
memory.steps = [
  SystemPromptStep(...),
  TaskStep(...),
  ActionStep(
    model_output="Thought: I need...\n```py\n...",
    code="result = 15 * 7\nprint(result)",
    observations="105"
  )
]
```

**Шаг 5: Генерация финального ответа (step 2)**

Отправляемые сообщения:
```
[
  ChatMessage(SYSTEM, "..."),
  ChatMessage(USER, "What is 15 multiplied by 7?"),
  ChatMessage(ASSISTANT, "Thought: I need...\n```py\n..."),
  ChatMessage(USER, "Observation: 105")
]
```

Ответ LLM:
```
Thought: I have the result, let me return it.
```py
final_answer(105)
```
```

Выполнение → `is_final_answer = True`, `output = 105`

**Финальный результат**: `105`

### StructuredCodeAgent (JSON mode)

При `use_structured_outputs_internally=True`, CodeAgent использует structured outputs:

```
┌────────────────────────────────────────────────────┐
│ Генерация в JSON формате:                         │
│                                                    │
│ {                                                  │
│   "thought": "I need to calculate...",            │
│   "code": "result = 15 * 7\nprint(result)"        │
│ }                                                  │
│                                                    │
│ Промт: structured_code_agent.yaml                 │
│ Схема JSON:                                       │
│   {                                                │
│     "type": "object",                             │
│     "properties": {                               │
│       "thought": {"type": "string"},              │
│       "code": {"type": "string"}                  │
│     },                                             │
│     "required": ["thought", "code"]               │
│   }                                                │
└────────────────────────────────────────────────────┘
```

### Используемые промты в CodeAgent

| Этап | Промт | Файл | Переменные |
|------|-------|------|-----------|
| Генерация кода | `system_prompt` | code_agent.yaml или structured_code_agent.yaml | `{{tools}}`, `{{authorized_imports}}`, `{{code_block_opening_tag}}`, `{{code_block_closing_tag}}` |
| Описание инструмента | Динамический | tools.py:to_code_prompt() | Для каждого инструмента |

### Передаваемые данные между этапами

**Из LLM**:
```python
{
  "content": "Thought: ...\n```py\ncode\n```",
  "token_usage": TokenUsage
}
```

**После парсинга**:
```python
{
  "code": "extracted code",
  "code_blobs": ["blob1", "blob2", ...]
}
```

**Из PythonExecutor**:
```python
{
  "observations": "print() outputs",
  "is_final_answer": bool,
  "output": Any  # Если final_answer вызван
}
```

**В память (ActionStep)**:
```python
ActionStep {
  "step_number": int,
  "model_input_messages": ChatMessage[],
  "model_output": str,
  "model_output_message": ChatMessage,
  "code": str,
  "observations": str,
  "is_final_answer": bool,
  "token_usage": TokenUsage,
  "timing": {"start": ..., "end": ...}
}
```

---

## Пайплайн ToolCallingAgent

ToolCallingAgent решает задачи через вызов инструментов в JSON формате. Использует цикл Action → Observation.

### Диаграмма потока ToolCallingAgent._step_stream()

```
┌──────────────────────────────────────────────────────────────────┐
│           _step_stream(action_step) - ToolCallingAgent           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ЭТАП 1: ПОДГОТОВКА СООБЩЕНИЙ ДЛЯ LLM                           │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ memory_messages = write_memory_to_messages()           │     │
│  │                                                        │     │
│  │ Преобразование аналогично CodeAgent:                  │     │
│  │ [SystemPromptStep] → ChatMessage(SYSTEM)              │     │
│  │ [TaskStep]         → ChatMessage(USER, task)          │     │
│  │ [ActionStep]       → ChatMessage(ASSISTANT, tool_calls) │   │
│  │                      ChatMessage(TOOL_RESPONSE, results)│   │
│  │                                                        │     │
│  │ memory_step.model_input_messages = messages            │     │
│  └────────────────────────────────────────────────────────┘     │
│                         │                                       │
│                         ▼                                       │
│  ЭТАП 2: ГЕНЕРАЦИЯ С TOOL CALLING                              │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ ПРОМТ: system_prompt из toolcalling_agent.yaml        │     │
│  │                                                        │     │
│  │ Переменные в промте:                                  │     │
│  │ - {{tools}}: описания через tool.to_tool_calling_prompt() │ │
│  │ - {{managed_agents}}: список управляемых агентов      │     │
│  │                                                        │     │
│  │ if stream_outputs:                                    │     │
│  │   for event in model.generate_stream(                 │     │
│  │     messages,                                         │     │
│  │     stop_sequences=["Observation:", "Calling tools:"], │    │
│  │     tools_to_call_from=self.tools_and_managed_agents  │     │
│  │   ):                                                  │     │
│  │     Yield ChatMessageStreamDelta                      │     │
│  │   chat_message = agglomerate_deltas()                 │     │
│  │ else:                                                 │     │
│  │   chat_message = model.generate(...)                 │     │
│  │                                                        │     │
│  │ memory_step.model_output = chat_message.content       │     │
│  │ memory_step.token_usage = chat_message.token_usage   │     │
│  └────────────────────────────────────────────────────────┘     │
│                         │                                       │
│                         ▼                                       │
│  ЭТАП 3: ПАРСИНГ TOOL CALLS                                    │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ if chat_message.tool_calls is None или пусто:         │     │
│  │   # LLM не вернула tool_calls напрямую               │     │
│  │   chat_message = model.parse_tool_calls(chat_message) │     │
│  │   # Парсинг из текста вывода                          │     │
│  │                                                        │     │
│  │ # Парсинг JSON arguments для каждого tool_call        │     │
│  │ for tool_call in chat_message.tool_calls:             │     │
│  │   if isinstance(tool_call.function.arguments, str):   │     │
│  │     try:                                              │     │
│  │       tool_call.function.arguments = json.loads(...)  │     │
│  │     except JSONDecodeError:                           │     │
│  │       raise AgentParsingError                         │     │
│  └────────────────────────────────────────────────────────┘     │
│                         │                                       │
│                         ▼                                       │
│  ЭТАП 4: ВЫПОЛНЕНИЕ TOOL CALLS                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ for output in process_tool_calls(                     │     │
│  │   chat_message, memory_step                           │     │
│  │ ):                                                     │     │
│  │                                                        │     │
│  │   ┌──────────────────────────────────────┐            │     │
│  │   │ ФАЗА 1: Yield всех ToolCall          │            │     │
│  │   │ for tool_call in tool_calls:          │            │     │
│  │   │   Yield ToolCall(name, args, id)      │            │     │
│  │   └──────────────────────────────────────┘            │     │
│  │                                                        │     │
│  │   ┌──────────────────────────────────────┐            │     │
│  │   │ ФАЗА 2: Выполнение                   │            │     │
│  │   │                                      │            │     │
│  │   │ if len(tool_calls) == 1:             │            │     │
│  │   │   # Последовательное выполнение      │            │     │
│  │   │   result = execute_tool_call(...)    │            │     │
│  │   │   Yield ToolOutput                   │            │     │
│  │   │                                      │            │     │
│  │   │ else:                                │            │     │
│  │   │   # Параллельное выполнение          │            │     │
│  │   │   with ThreadPoolExecutor() as pool: │            │     │
│  │   │     futures = [                      │            │     │
│  │   │       pool.submit(execute_tool_call, tc) │       │     │
│  │   │       for tc in tool_calls           │            │     │
│  │   │     ]                                │            │     │
│  │   │     for future in as_completed():    │            │     │
│  │   │       result = future.result()       │            │     │
│  │   │       Yield ToolOutput               │            │     │
│  │   └──────────────────────────────────────┘            │     │
│  │                                                        │     │
│  │   execute_tool_call():                                │     │
│  │   ├─ Валидация имени и аргументов                    │     │
│  │   ├─ Замена state_variables в аргументах             │     │
│  │   ├─ Если обычный инструмент:                        │     │
│  │   │  └─ result = tool(**arguments)                   │     │
│  │   └─ Если managed_agent:                             │     │
│  │      └─ result = managed_agent.run(                  │     │
│  │           task=arguments["task"],                    │     │
│  │           additional_args=arguments.get("additional_args") │ │
│  │         )                                             │     │
│  │                                                        │     │
│  │   ToolOutput(                                         │     │
│  │     id=tool_call.id,                                  │     │
│  │     output=result,                                    │     │
│  │     is_final_answer=(tool_name == "final_answer"),   │     │
│  │     observation=truncate(str(result)),                │     │
│  │     tool_call=tool_call                               │     │
│  │   )                                                    │     │
│  │                                                        │     │
│  │   memory_step.tool_calls.append(tool_call)            │     │
│  │   memory_step.observations.append(observation)        │     │
│  │                                                        │     │
│  │   if is_final_answer:                                 │     │
│  │     Yield ActionOutput(output, is_final_answer=True)  │     │
│  │     break                                             │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Пример выполнения ToolCallingAgent

**Задача**: "Search for Python tutorials and summarize the top result"

**Шаг 1: Генерация первого action**

Ответ LLM:
```json
{
  "tool_calls": [
    {
      "id": "call_1",
      "type": "function",
      "function": {
        "name": "web_search",
        "arguments": "{\"query\": \"Python tutorials\"}"
      }
    }
  ]
}
```

**Шаг 2: Выполнение web_search**
```
Observation: "Found: 1. Learn Python - Tutorial for Beginners at python.org..."
```

**Шаг 3: Генерация второго action**

Входящие сообщения включают предыдущий tool call и observation.

Ответ LLM:
```json
{
  "tool_calls": [
    {
      "id": "call_2",
      "type": "function",
      "function": {
        "name": "final_answer",
        "arguments": "{\"answer\": \"The top Python tutorial is...\"}"
      }
    }
  ]
}
```

### Параллельное выполнение инструментов

Если LLM возвращает несколько tool_calls одновременно:

```
tool_calls = [
  ToolCall(name="web_search", args={"query": "Python"}),
  ToolCall(name="web_search", args={"query": "JavaScript"}),
  ToolCall(name="web_search", args={"query": "Rust"})
]

# Выполняются параллельно через ThreadPoolExecutor
Results:
  - ToolOutput(id="call_1", output="Python results...")
  - ToolOutput(id="call_2", output="JavaScript results...")
  - ToolOutput(id="call_3", output="Rust results...")
```

### Используемые промты в ToolCallingAgent

| Этап | Промт | Файл | Переменные |
|------|-------|------|-----------|
| Генерация actions | `system_prompt` | toolcalling_agent.yaml | `{{tools}}`, `{{managed_agents}}` |
| Описание инструмента | Динамический | tools.py:to_tool_calling_prompt() | Для каждого инструмента |

---

## Пайплайн с планированием

Когда `planning_interval > 0`, агент создает и обновляет планы решения задачи.

### Диаграмма потока планирования

```
┌─────────────────────────────────────────────────────────────────┐
│         _generate_planning_step(task, is_first_step, step)      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  if is_first_step (step_number == 1):                          │
│                                                                 │
│    ┌──────────────────────────────────────────────────┐        │
│    │ НАЧАЛЬНОЕ ПЛАНИРОВАНИЕ                           │        │
│    │                                                  │        │
│    │ prompt = prompt_templates["planning"]           │        │
│    │          ["initial_plan"]                       │        │
│    │                                                  │        │
│    │ filled_prompt = populate_template(prompt, {     │        │
│    │   "task": task,                                 │        │
│    │   "tools": tools,                               │        │
│    │   "managed_agents": managed_agents              │        │
│    │ })                                              │        │
│    │                                                  │        │
│    │ input_messages = [                              │        │
│    │   ChatMessage(USER, filled_prompt)              │        │
│    │ ]                                               │        │
│    │                                                  │        │
│    │ plan_message = model.generate(                  │        │
│    │   input_messages,                               │        │
│    │   stop_sequences=["<end_plan>"]                │        │
│    │ )                                               │        │
│    │                                                  │        │
│    │ ОЖИДАЕМЫЙ ФОРМАТ ОТВЕТА:                        │        │
│    │ ## 1. Facts survey                              │        │
│    │ ### 1.1. Facts given in the task                │        │
│    │ - Fact 1                                        │        │
│    │ ### 1.2. Facts to look up                       │        │
│    │ - Where to search for X                         │        │
│    │ ### 1.3. Facts to derive                        │        │
│    │ - Calculation needed                            │        │
│    │                                                  │        │
│    │ ## 2. Plan                                      │        │
│    │ 1. First step                                   │        │
│    │ 2. Second step                                  │        │
│    │ 3. Final step                                   │        │
│    │ <end_plan>                                      │        │
│    │                                                  │        │
│    │ plan = f"Here are the facts I know...\n{content}"│       │
│    └──────────────────────────────────────────────────┘        │
│                                                                 │
│  else:  # NOT first_step - обновление плана                    │
│                                                                 │
│    ┌──────────────────────────────────────────────────┐        │
│    │ ОБНОВЛЕНИЕ ПЛАНА                                 │        │
│    │                                                  │        │
│    │ # Получить память БЕЗ system_prompt и планов    │        │
│    │ memory_messages =                               │        │
│    │   write_memory_to_messages(summary_mode=True)    │        │
│    │   # summary_mode удаляет ненужное               │        │
│    │                                                  │        │
│    │ pre_msg = ChatMessage(SYSTEM,                   │        │
│    │   prompt_templates["planning"]                  │        │
│    │     ["update_plan_pre_messages"]                │        │
│    │   populated with {"task": task}                 │        │
│    │ )                                               │        │
│    │                                                  │        │
│    │ post_msg = ChatMessage(USER,                    │        │
│    │   prompt_templates["planning"]                  │        │
│    │     ["update_plan_post_messages"]               │        │
│    │   populated with {                              │        │
│    │     "task": task,                               │        │
│    │     "tools": tools,                             │        │
│    │     "managed_agents": managed_agents,           │        │
│    │     "remaining_steps": max_steps - step         │        │
│    │   }                                              │        │
│    │ )                                               │        │
│    │                                                  │        │
│    │ input_messages = [pre_msg] +                    │        │
│    │                  memory_messages +               │        │
│    │                  [post_msg]                      │        │
│    │                                                  │        │
│    │ plan_message = model.generate(...)              │        │
│    │                                                  │        │
│    │ plan = f"I still need to solve...\n{content}"   │        │
│    └──────────────────────────────────────────────────┘        │
│                                                                 │
│  PlanningStep(                                                 │
│    model_input_messages=input_messages,                        │
│    plan=plan,                                                  │
│    model_output_message=plan_message,                          │
│    token_usage=...,                                            │
│    timing=...                                                  │
│  )                                                             │
│                                                                 │
│  Yield PlanningStep                                            │
│  memory.steps.append(PlanningStep)                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Интеграция планирования в основной цикл

```
for step_number in range(1, max_steps + 1):

  # ПРОВЕРКА НЕОБХОДИМОСТИ ПЛАНИРОВАНИЯ
  if planning_interval is not None:
    if step_number == 1 or (step_number - 1) % planning_interval == 0:

      ┌─────────────────────────────────────┐
      │ planning_step = _generate_planning_step( │
      │   task=self.task,                   │
      │   is_first_step=(step_number == 1), │
      │   step=step_number                  │
      │ )                                   │
      │                                     │
      │ memory.steps.append(planning_step)  │
      │ Yield planning_step                 │
      └─────────────────────────────────────┘

  # ОБЫЧНЫЙ ACTION STEP
  action_step = ActionStep(...)
  ...
```

### Пример с планированием

**Параметры**: `planning_interval=3`

**Последовательность шагов**:
```
Step 0: TaskStep("Complex multi-step task")

Step 1: PlanningStep (initial_plan)
        → Creates initial plan with facts survey

Step 1: ActionStep
        → Executes first action based on plan

Step 2: ActionStep
        → Continues execution

Step 3: ActionStep
        → Continues execution

Step 4: PlanningStep (update_plan)
        → Reviews progress, updates plan
        → Considers remaining_steps

Step 4: ActionStep
        → Executes based on updated plan

...
```

---

## Пайплайн с управляемыми агентами

Managed Agents позволяют основному агенту делегировать подзадачи другим агентам.

### Диаграмма вызова managed agent

```
┌──────────────────────────────────────────────────────────────────┐
│                  ОСНОВНОЙ АГЕНТ (Manager)                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ЭТАП 1: ГЕНЕРАЦИЯ TOOL CALL ДЛЯ MANAGED AGENT                  │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ LLM видит managed_agents в system_prompt:              │     │
│  │                                                        │     │
│  │ def research_assistant(                               │     │
│  │   task: str,                                          │     │
│  │   additional_args: dict[str, Any]                     │     │
│  │ ) -> str:                                             │     │
│  │   """Expert at researching topics"""                 │     │
│  │                                                        │     │
│  │ LLM генерирует:                                       │     │
│  │ {                                                      │     │
│  │   "name": "research_assistant",                       │     │
│  │   "arguments": {                                      │     │
│  │     "task": "Research quantum computing basics",     │     │
│  │     "additional_args": {"depth": "detailed"}         │     │
│  │   }                                                    │     │
│  │ }                                                      │     │
│  └────────────────────────────────────────────────────────┘     │
│                         │                                       │
│                         ▼                                       │
│  ЭТАП 2: ВЫПОЛНЕНИЕ MANAGED AGENT CALL                         │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ execute_tool_call() распознает managed_agent:         │     │
│  │                                                        │     │
│  │ if tool_name in self.managed_agents:                  │     │
│  │   managed_agent = self.managed_agents[tool_name]      │     │
│  │                                                        │     │
│  │   # Подготовка промта для managed agent              │     │
│  │   task_prompt = populate_template(                    │     │
│  │     prompt_templates["managed_agent"]["task"], {     │     │
│  │       "name": managed_agent.name,                     │     │
│  │       "task": arguments["task"]                       │     │
│  │     }                                                  │     │
│  │   )                                                    │     │
│  │                                                        │     │
│  │   ┌──────────────────────────────────────┐            │     │
│  │   │ РЕКУРСИВНЫЙ ВЫЗОВ АГЕНТА             │            │     │
│  │   │                                      │            │     │
│  │   │ result = managed_agent.run(          │            │     │
│  │   │   task=task_prompt,                  │            │     │
│  │   │   additional_args=arguments.get(     │            │     │
│  │   │     "additional_args", {}            │            │     │
│  │   │   )                                  │            │     │
│  │   │ )                                    │            │     │
│  │   │                                      │            │     │
│  │   │ # Managed agent выполняет свои шаги │            │     │
│  │   │ # Имеет собственную память          │            │     │
│  │   │ # Возвращает final_answer            │            │     │
│  │   └──────────────────────────────────────┘            │     │
│  │                                                        │     │
│  │   # Форматирование отчета                             │     │
│  │   report = populate_template(                         │     │
│  │     prompt_templates["managed_agent"]["report"], {   │     │
│  │       "name": managed_agent.name,                     │     │
│  │       "final_answer": result                          │     │
│  │     }                                                  │     │
│  │   )                                                    │     │
│  │                                                        │     │
│  │   return report                                       │     │
│  └────────────────────────────────────────────────────────┘     │
│                         │                                       │
│                         ▼                                       │
│  ЭТАП 3: ИСПОЛЬЗОВАНИЕ РЕЗУЛЬТАТА                              │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ ToolOutput(                                            │     │
│  │   output=report,                                       │     │
│  │   observation="Here is the final answer from your     │     │
│  │                managed agent 'research_assistant':     │     │
│  │                ### 1. Task outcome (short): ...        │     │
│  │                ### 2. Task outcome (detailed): ...     │     │
│  │                ### 3. Additional context: ..."         │     │
│  │ )                                                      │     │
│  │                                                        │     │
│  │ Основной агент видит этот результат и продолжает      │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Иерархия агентов

Managed agents могут сами иметь managed agents:

```
Main Agent (Manager)
├── Research Agent (Managed)
│   ├── Web Search Agent (Sub-managed)
│   └── Summarization Agent (Sub-managed)
├── Code Agent (Managed)
└── Visualization Agent (Managed)
```

Каждый уровень выполняется рекурсивно через `agent.run()`.

### Используемые промты для managed agents

| Этап | Промт | Назначение |
|------|-------|-----------|
| Передача задачи | `managed_agent.task` | Форматирует задачу с инструкциями для managed agent |
| Возврат результата | `managed_agent.report` | Форматирует отчет от managed agent обратно к менеджеру |

---

## Обработка ошибок

### Типы ошибок

1. **AgentGenerationError** - критическая ошибка генерации (пробрасывается)
2. **AgentParsingError** - ошибка парсинга вывода (сохраняется в step.error)
3. **AgentExecutionError** - ошибка выполнения кода/инструмента (сохраняется в step.error)
4. **AgentMaxStepsError** - достигнут лимит шагов (специальная обработка)

### Обработка достижения max_steps

```
┌─────────────────────────────────────────────────────────────────┐
│            _handle_max_steps_reached(task)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ЭТАП 1: ПОДГОТОВКА КОНТЕКСТА                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ memory_messages = write_memory_to_messages()           │    │
│  │ # Вся история выполнения                               │    │
│  └────────────────────────────────────────────────────────┘    │
│                         │                                      │
│                         ▼                                      │
│  ЭТАП 2: ПРОМТЫ ДЛЯ ФИНАЛЬНОГО ОТВЕТА                         │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ pre_msg = ChatMessage(SYSTEM,                          │    │
│  │   prompt_templates["final_answer"]["pre_messages"]    │    │
│  │ )                                                      │    │
│  │ # "An agent tried to answer a user query but          │    │
│  │ #  it got stuck..."                                    │    │
│  │                                                        │    │
│  │ post_msg = ChatMessage(USER,                          │    │
│  │   prompt_templates["final_answer"]["post_messages"]   │    │
│  │   populated with {"task": task}                       │    │
│  │ )                                                      │    │
│  │ # "Based on the above, please provide an answer..."   │    │
│  │                                                        │    │
│  │ input_messages = [pre_msg] +                          │    │
│  │                  memory_messages +                     │    │
│  │                  [post_msg]                            │    │
│  └────────────────────────────────────────────────────────┘    │
│                         │                                      │
│                         ▼                                      │
│  ЭТАП 3: ГЕНЕРАЦИЯ ФИНАЛЬНОГО ОТВЕТА                          │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ final_answer = model.generate(input_messages)          │    │
│  │                                                        │    │
│  │ # LLM анализирует всю историю и пытается             │    │
│  │ # сформулировать ответ на основе собранной информации │    │
│  │                                                        │    │
│  │ return final_answer.content                            │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Обработка ошибок в цикле

```python
try:
  for output in _step_stream(action_step):
    Yield output
    if ActionOutput.is_final_answer:
      returned_final_answer = True
      break

except AgentGenerationError:
  # Критическая ошибка - пробросить
  raise

except AgentError as e:
  # Ошибка парсинга или выполнения
  action_step.error = e
  # Продолжить на следующий шаг
  # LLM увидит ошибку в истории и может исправить

finally:
  # Всегда финализировать шаг
  _finalize_step(action_step)
  memory.steps.append(action_step)
  Yield action_step
```

---
