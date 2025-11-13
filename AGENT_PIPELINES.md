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
