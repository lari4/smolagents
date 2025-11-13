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
