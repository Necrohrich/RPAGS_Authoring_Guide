---
layout: default
title: Game System Authoring Guide
nav_order: 1
---

# Game System Authoring Guide
{: .no_toc }

Руководство для авторов игровых систем движка настольных RPG.
{: .fs-5 .fw-300 }

---

## Содержание
{: .no_toc .text-delta }

1. TOC
{:toc}

---



## 1. Структура ZIP-архива

Движок принимает ZIP-архив с файлами `.jsonc` (JSON with Comments — `//` и `/* */` стрипаются перед парсингом). Единственный обязательный файл — `manifest.jsonc` в корне архива.

```
my_system/
  manifest.jsonc          ← обязателен, корневой файл
  sheet/
    attributes.jsonc
    skills.jsonc
  rolls/
    checks.jsonc
  rules/
    conditions.jsonc
  actions/
    actions.jsonc
```

### manifest.jsonc

```jsonc
{
  "version": "1.0.0",
  "changelog": "Первая версия",

  // Подключаемые файлы — ключи из них мерджатся в общую схему.
  // При конфликте ключей manifest.jsonc побеждает.
  "includes": [
    "sheet/attributes.jsonc",
    "rolls/checks.jsonc",
    "rules/conditions.jsonc",
    "actions/actions.jsonc"
  ],

  // Активные модули системы — движок проверяет их через ManifestEngine
  "manifest": {
    "has_sheet":       true,
    "has_rolls":       true,
    "has_creation":    false,
    "has_progression": false,
    "has_actions":     false
  }
}
```

**Правила загрузки:**

- Путь traversal (`../evil.jsonc`) запрещён — загрузчик бросает `UnsafePathError`.
- Циклические includes (`a → b → a`) запрещены — `CircularIncludeError`.
- Максимальный размер одного файла — 2 МБ — `FileTooLargeError`.
- Если файл из `includes` не найден в архиве — `IncludeNotFoundError`.

### Альтернатива ZIP — PATCH API

Если архив не нужен, схемы можно загрузить напрямую через `PATCH /admin/game-systems/{id}/schemas` с телом:

```json
{
  "version": "1.0.0",
  "sheet_schema":   { "sections": [] },
  "rolls_schema":   {},
  "rules_schema":   {},
  "actions_schema": {}
}
```

Оба пути проходят через один `SchemaValidator`.

---

## 2. sheet\_schema — анкета персонажа

`sheet_schema` описывает поля анкеты персонажа. Верхний уровень — список секций, каждая секция содержит поля.

```jsonc
{
  "sections": [
    {
      "id": "attributes",      // [a-z][a-z0-9_]* — обязательно
      "label": "Атрибуты",     // строка или { "en": "Attributes", "ru": "Атрибуты" }
      "display": { ... },      // опционально
      "fields": [ ... ]
    }
  ]
}
```

### 2.1. Правила именования id

`id` используется в формулах как переменная Python — отсюда ограничения:

| Правило | Верно | Неверно |
|---------|-------|---------|
| Только латиница и цифры | `strength`, `dex_mod` | `сила`, `ловкость` |
| Начинается с буквы | `level`, `hp_max` | `1level`, `_hp` |
| Нижний регистр + underscore | `base_attack` | `BaseAttack`, `spellDC` |
| Уникален в пределах всей системы | — | два поля с id `strength` |

Транслитерация допустима: `sila`, `lovkost`.

### 2.2. Типы полей

| Тип | Описание | Специфичные параметры |
|-----|----------|-----------------------|
| `integer` | Целое число | `min`, `max`, `default` |
| `number` | Дробное число | `min`, `max`, `default` |
| `text` | Строка | `max_length`, `required` |
| `textarea` | Многострочный текст | `max_length` |
| `boolean` | Чекбокс | `default` |
| `select` | Выбор одного значения из списка | `options: string[]` — обязателен |
| `multiselect` | Выбор нескольких значений | `options: string[]` — обязателен |
| `skill` | Навык с базовым атрибутом | `base_attribute` — обязателен |
| `computed` | Вычисляется по формуле, readonly | `formula` — обязателен |
| `resource` | Пул ресурсов (HP, ячейки заклинаний) | `max_formula`, `recharge` |
| `list` | Список объектов (инвентарь) | `item_schema` |
| `dice` | Тип кубика (`d6`, `d8`...) | `options` — если не задан, доступны d2/d4/d6/d8/d10/d12/d20 |

**integer:**
```json
{
  "id": "strength",
  "label": "Сила",
  "type": "integer",
  "min": 1,
  "max": 20,
  "default": 10,
  "required": true
}
```

**select:**
```json
{
  "id": "character_class",
  "label": "Класс",
  "type": "select",
  "options": ["fighter", "wizard", "rogue"]
}
```

**skill** — движок автоматически создаёт дополнительное поле `{id}_proficient: boolean` для отметки владения:
```json
{
  "id": "athletics",
  "label": "Атлетика",
  "type": "skill",
  "base_attribute": "strength"
}
```

### 2.3. Computed поля и derived

**computed** — отдельное поле, значение которого вычисляется из других полей. Нельзя записать через PATCH `/sheet` — только читать.

```json
{
  "id": "strength_modifier",
  "label": "Модификатор силы",
  "type": "computed",
  "formula": "floor((strength - 10) / 2)"
}
```

**derived** — вычисляемые поля, объявленные внутри другого поля. Семантически то же самое, что `computed`, но привязаны к родительскому полю в схеме:

```json
{
  "id": "hit_die",
  "label": "Кость хитов",
  "type": "dice",
  "derived": [
    {
      "id": "max_hp",
      "label": "Максимум HP",
      "formula": "level * 8 + constitution_modifier"
    }
  ]
}
```

Порядок вычисления топологический — если `max_hp` зависит от `constitution_modifier`, а тот от `constitution`, движок вычислит их в правильном порядке. Циклические зависимости (`a` зависит от `b`, `b` зависит от `a`) блокируются валидатором с кодом `CIRCULAR_DEPENDENCY`.

### 2.4. display — конфигурация отображения

Опциональный объект для секции или поля. Все ключи опциональны.

```jsonc
// display для секции
{
  "display": {
    "icon": "⚔️",          // эмодзи перед заголовком
    "embed_color": "#e74c3c", // цвет Embed в Discord (hex)
    "collapsed": false,      // секция свёрнута по умолчанию на сайте
    "order": 1,              // порядок отображения секций
    "show_in_summary": true  // показывать в кратком просмотре
  }
}

// display для поля
{
  "display": {
    "icon": "💪",
    "color": "#3498db",      // цвет поля в веб-UI (hex)
    "position": "left",      // left | right | center
    "size": "normal",        // small | normal | large
    "hint": "От 1 до 20",   // подсказка под полем
    "hidden": false,         // скрыть из анкеты и модалки редактирования
    "group": "physical"      // визуальная группировка внутри секции
  }
}
```

`embed_color` применяется только к Discord Embed. `color`, `position`, `size`, `hint`, `group` — только веб-UI. `icon`, `hidden`, `order`, `show_in_summary` — оба рендерера.

### 2.5. Локализация label

```json
{ "id": "strength", "label": { "en": "Strength", "ru": "Сила", "de": "Stärke" } }
```

Если запрошенный язык отсутствует — берётся первый доступный ключ.

---

## 3. rolls\_schema — броски

`rolls_schema` — словарь `{roll_id: roll_definition}`. Каждый бросок описывает формулу, опциональные параметры, правила кубиков и интерпретацию результата.

```jsonc
{
  "rolls_schema": {
    "attack_roll": {
      "id": "attack_roll",        // должен совпадать с ключом словаря
      "label": "Бросок атаки",
      "formula": "1d20 + strength_modifier",
      "parameters": [ ... ],     // опционально
      "dice_rules": { ... },     // опционально
      "result": { ... },         // обязателен
      "on_hit": { ... },         // опционально — следующий бросок при успехе
      "roll_modifier": null      // "advantage_successes" | "disadvantage_successes"
    }
  }
}
```

### 3.1. Нотация формулы

Формула кубиков — строка, разбираемая `DiceEngine`:

| Нотация | Описание |
|---------|----------|
| `1d20` | 1 кубик d20 |
| `3d6` | 3 кубика d6 |
| `1d20+5` | d20 плюс константа |
| `1d20 + strength_modifier` | d20 плюс computed/обычное поле из контекста |
| `{pool_size}d10` | количество кубиков из переменной контекста |
| `{hit_die}` | если `hit_die` содержит строку типа `"d8"` — парсится как `1d8` |
| `4d6kH3` | бросить 4d6, оставить 3 наибольших |
| `4d6kL3` | бросить 4d6, оставить 3 наименьших |
| `1d6!` | взрывающийся кубик (нотация) |

Допустимые грани: `d2`, `d3`, `d4`, `d6`, `d8`, `d10`, `d12`, `d20`, `d100`.

### 3.2. dice\_rules

Модификаторы броска применяются после парсинга нотации. Все поля опциональны.

```json
{
  "dice_rules": {
    "exploding": true,
    "explode_on": "max",
    "explode_limit": 10,
    "reroll_ones": false,
    "reroll_condition": "<= 2",
    "cap": 5,
    "min_die": 2
  }
}
```

| Поле | Тип | По умолчанию | Описание |
|------|-----|--------------|----------|
| `exploding` | bool | `false` | Максимум на грани → бросить ещё один кубик |
| `explode_on` | string | `"max"` | Условие взрыва: `"max"`, `"value:6"`, `">= 5"` |
| `explode_limit` | int | `10` | Максимум цепочек взрыва. Обязателен при `exploding: true`, иначе — предупреждение |
| `reroll_ones` | bool | `false` | Автоматически перебросить единицы (один раз) |
| `reroll_condition` | string | null | Произвольное условие переброса: `"<= 2"`, `"== 1"` |
| `cap` | int | null | Максимальное значение одного кубика |
| `min_die` | int | null | Минимальное значение одного кубика |

`reroll` срабатывает один раз — второй подряд неудачный результат засчитывается.

### 3.3. parameters

Параметры, которые нужны для броска. Бывают двух видов: **статические** (берутся из анкеты или вычисляются заранее) и **runtime** (запрашиваются у игрока в момент броска через UI).

```json
{
  "parameters": [
    {
      "id": "target",
      "type": "reference",
      "label": "Сложность из поля",
      "source_field": "spell_save_dc"
    },
    {
      "id": "bonus",
      "type": "computed",
      "label": "Бонус мастерства",
      "formula": "proficiency_bonus"
    },
    {
      "id": "target",
      "type": "runtime_integer",
      "label": "Сложность броска",
      "min": 1,
      "max": 30,
      "default": 15
    },
    {
      "id": "skill_choice",
      "type": "runtime_select",
      "label": "Выберите навык",
      "source": "sheet_fields",
      "filter": { "section": "skills" }
    },
    {
      "id": "use_bonus",
      "type": "runtime_boolean",
      "label": "Использовать бонус мастерства?"
    }
  ]
}
```

| Тип | Описание | Специфичные поля |
|-----|----------|-----------------|
| `reference` | Берёт значение из поля анкеты | `source_field` — id поля |
| `computed` | Вычисляет формулу через FormulaEngine | `formula` |
| `runtime_integer` | Игрок вводит число | `min`, `max`, `default` |
| `runtime_select` | Игрок выбирает из списка | `source`, `filter`, `options` |
| `runtime_boolean` | Игрок ставит галочку | `options: [{value, label, formula}]` |

Бросок с любым `runtime_*` параметром показывает UI перед броском. Бросок только с `reference`/`computed` — выполняется сразу.

### 3.4. Типы result

#### compare

Бросает формулу, сравнивает `total` с `target`. `target` должен прийти через параметр типа `runtime_integer` или `reference`.

Доступные переменные в `condition`: `total`, `target`, `raw_dice` (значение первого кубика до модификаторов).

```json
{
  "result": {
    "type": "compare",
    "outcomes": [
      { "condition": "raw_dice == 20", "label": "Критический успех" },
      { "condition": "raw_dice == 1",  "label": "Критический провал" },
      { "condition": "total >= target", "label": "Успех" },
      { "condition": "total < target",  "label": "Провал" }
    ]
  }
}
```

Условия проверяются по порядку — срабатывает первое истинное.

#### roll\_under

Успех = бросок **не превышает** значение поля анкеты. Используется в GURPS, Call of Cthulhu.

```json
{
  "result": {
    "type": "roll_under",
    "target_field": "skill_value",
    "critical_success_threshold": 5,
    "critical_fail_threshold": 96,
    "outcomes": [
      { "condition": "total <= critical_success_threshold", "label": "Критический успех" },
      { "condition": "total >= critical_fail_threshold",    "label": "Критический провал" },
      { "condition": "total <= target_field",               "label": "Успех" },
      { "condition": "total > target_field",                "label": "Провал" }
    ]
  }
}
```

Переменные в `condition`: `total`, `target_field` (значение поля анкеты), `critical_success_threshold`, `critical_fail_threshold`.

#### count\_successes

Подсчёт успехов по пулу кубиков. Используется в World of Darkness, Shadowrun.

```json
{
  "formula": "5d10",
  "result": {
    "type": "count_successes",
    "success_condition": "die >= difficulty",
    "botch_condition": "die == 1",
    "cancel_on_botch": true,
    "default_difficulty": 6,
    "outcomes": [
      { "condition": "successes >= 3", "label": "Выдающийся успех" },
      { "condition": "successes >= 1", "label": "Успех" },
      { "condition": "successes == 0 and botch", "label": "Провал с осложнением" },
      { "condition": "successes == 0", "label": "Провал" }
    ]
  }
}
```

- `success_condition` — проверяется для каждого кубика, переменные: `die`, `difficulty`.
- `botch_condition` — аналогично. Если не задан — botch невозможен.
- `cancel_on_botch: true` — каждая единица вычитает один успех.
- `difficulty` приходит через параметр типа `runtime_integer` или `reference`; если не передан — используется `default_difficulty`.
- Переменные в `outcomes`: `successes`, `botch` (bool).

#### opposed

Два броска сравниваются между собой. Переменные в `condition`: `total` (активный), `opponent_total`.

```json
{
  "result": {
    "type": "opposed",
    "outcomes": [
      { "condition": "total > opponent_total",  "label": "Победа" },
      { "condition": "total == opponent_total", "label": "Ничья" },
      { "condition": "total < opponent_total",  "label": "Поражение" }
    ]
  }
}
```

#### table\_lookup

Результат определяется по таблице из `rules_schema.tables`. Таблица должна существовать — валидатор проверяет ссылку.

```json
{
  "formula": "1d6",
  "result": {
    "type": "table_lookup",
    "table": "fumble_table",
    "outcomes": []
  }
}
```

#### multi\_pool

Пул разносимвольных кубиков с взаимоотменой. Используется в Genesys/Star Wars FFG.

```jsonc
{
  "result": {
    "type": "multi_pool",
    "pools": [
      {
        "id": "ability",
        "dice": "{ability_dice}d8",
        "symbols": ["success", "advantage"],
        // faces: грань → список символов
        "faces": {
          "1": [],
          "2": ["success"],
          "6": ["success", "advantage"],
          "8": ["triumph"]
        }
      },
      {
        "id": "difficulty",
        "dice": "{difficulty_dice}d8",
        "symbols": ["failure", "threat"],
        "faces": {
          "1": [],
          "2": ["failure"],
          "8": ["despair"]
        }
      }
    ],
    // пары символов которые аннулируют друг друга
    "cancel_pairs": [["success", "failure"], ["advantage", "threat"]],
    // поля для отображения итога
    "display_fields": ["success", "advantage", "failure", "threat"],
    "outcomes": [
      { "condition": "success > 0", "label": "Успех" },
      { "condition": "success == 0", "label": "Провал" }
    ]
  }
}
```

Если `faces` не задан — символы распределяются round-robin по значению грани.

#### custom

Движок не интерпретирует результат — возвращает сырые кубики. Используется для специфичных систем.

```json
{
  "result": {
    "type": "custom",
    "description": "Интерпретируйте результат самостоятельно",
    "display_fields": []
  }
}
```

### 3.5. on\_hit

Цепочка бросков: если основной бросок успешен (`success: true`), автоматически выполняется следующий.

```json
{
  "on_hit": {
    "roll_id": "damage_roll"
  }
}
```

---

## 4. rules\_schema — правила системы

`rules_schema` содержит метаданные и игровые правила: манифест, права, состояния, типы предметов, пулы ресурсов, группы опций, таблицы, прогрессию.

### 4.1. manifest

Определяет активные модули системы. Неизвестные ключи запрещены — опечатка даёт ошибку валидации.

```json
{
  "rules_schema": {
    "manifest": {
      "has_sheet":       true,
      "has_rolls":       true,
      "has_creation":    false,
      "has_progression": false,
      "has_actions":     false
    }
  }
}
```

Если `has_sheet: true`, но `sheet_schema.sections` пуст — валидатор выдаст предупреждение `MANIFEST_MODULE_EMPTY`.

### 4.2. permissions

Матрица прав доступа к анкете. Неизвестные ключи запрещены. Значения здесь — дефолты для всей системы; могут быть переопределены в настройках конкретной игры.

```json
{
  "permissions": {
    "player_can_edit_sheet":    false,
    "player_can_roll":          true,
    "player_can_add_items":     true,
    "player_can_remove_items":  false,
    "gm_can_edit_player_sheet": true,
    "gm_can_create_custom":     true,
    "locked_fields":            ["character_class", "race"],
    "public_fields":            ["name", "character_class"]
  }
}
```

- `locked_fields` — игрок не может редактировать эти поля даже при `player_can_edit_sheet: true`.
- `public_fields` — поля, видимые другим игрокам. Если список пуст — чужая анкета недоступна.

### 4.3. conditions — состояния и модификаторы

Временные состояния персонажа (отравлен, воодушевлён и т.д.). Активные состояния хранятся в `sheet_data.active_conditions` как список id.

```json
{
  "conditions": [
    {
      "id": "poisoned",
      "label": "Отравлен",
      "duration": { "count": 1, "type": "scene" },
      "modifiers": [
        {
          "target": "roll",
          "op": "disadvantage",
          "roll_ids": ["attack_roll", "ability_check"]
        },
        {
          "target": "field",
          "op": "multiply",
          "field": "speed",
          "value": 0.5
        }
      ]
    },
    {
      "id": "blessed",
      "label": "Благословлён",
      "modifiers": [
        {
          "target": "roll",
          "op": "bonus_dice",
          "dice": "1d4",
          "roll_ids": ["attack_roll", "saving_throw"]
        }
      ]
    }
  ]
}
```

**Модификаторы (target: roll):**

| op | Описание |
|----|----------|
| `advantage` | Бросить дважды, взять лучший |
| `disadvantage` | Бросить дважды, взять худший |
| `bonus_dice` | Добавить дополнительный кубик (поле `dice`: нотация) |

**Модификаторы (target: field):**

| op | Описание |
|----|----------|
| `add` | Прибавить `value` к полю |
| `multiply` | Умножить поле на `value` |

`roll_ids` — список id бросков из `rolls_schema`. Валидатор проверяет их существование.

### 4.4. item\_types

Типы предметов с полями и эффектами при экипировке.

```json
{
  "item_types": [
    {
      "id": "weapon",
      "label": "Оружие",
      "fields": [
        { "id": "damage_dice", "label": "Урон", "type": "dice" },
        { "id": "damage_bonus", "label": "Бонус урона", "type": "integer", "default": 0 }
      ],
      "on_equip": [
        {
          "item_key": "damage_bonus",
          "op": "add",
          "target": "attack_bonus"
        }
      ]
    }
  ]
}
```

`on_equip[].item_key` — поле из `fields` этого типа. Валидатор проверяет, что `item_key` существует в `fields`.

### 4.5. resource\_pools

Пулы для систем создания персонажа с point-buy (GURPS, HERO).

```json
{
  "resource_pools": [
    {
      "id": "character_points",
      "label": "Очки персонажа",
      "initial": 100,
      "initial_field": null,
      "thresholds": [50, 75, 100],
      "spend_on": [
        {
          "targets": ["strength", "dexterity", "constitution"],
          "rate": 10
        }
      ],
      "gain_on": null
    }
  ]
}
```

Если задан `initial_field` — начальный бюджет берётся из `sheet_data[initial_field]` вместо `initial`. `spend_on[].targets` ссылаются на поля `sheet_schema` — валидатор проверяет их существование.

### 4.6. option\_groups

Группы опций для выбора расы, класса, фита. Каждая опция применяет эффекты к `sheet_data`.

```json
{
  "option_groups": [
    {
      "id": "character_class",
      "label": "Класс",
      "required": true,
      "max_picks": 1,
      "options": [
        {
          "id": "fighter",
          "label": "Воин",
          "effects": [
            { "field": "hit_die",         "op": "set",     "value": "d10" },
            { "field": "proficiency_bonus","op": "formula", "value": "floor((level - 1) / 4) + 2" }
          ]
        },
        {
          "id": "wizard",
          "label": "Волшебник",
          "effects": [
            { "field": "hit_die", "op": "set", "value": "d6" },
            {
              "op": "unlock",
              "group": "wizard_spells"
            }
          ]
        }
      ]
    }
  ]
}
```

**Операции эффектов:**

| op | Описание |
|----|----------|
| `add` | Прибавить `value` к полю |
| `set` | Установить поле в `value` |
| `formula` | Вычислить `value` как формулу через FormulaEngine |
| `grant` | Автоматически выдать другую опцию |
| `unlock` | Открыть доступ к другой `option_group` (`group: id`) |

`effects[].field` для `op` не равных `grant`/`unlock` должен существовать в `sheet_schema` — валидатор проверяет. Лимит: 200 опций на группу (предупреждение при >100).

### 4.7. tables

Справочные таблицы для `table_lookup` бросков.

```json
{
  "tables": {
    "fumble_table": {
      "label": "Таблица критических провалов",
      "rows": [
        { "min": 1,  "max": 2,  "result": "Роняешь оружие" },
        { "min": 3,  "max": 4,  "result": "Получаешь урон 1d4" },
        { "min": 5,  "max": 6,  "result": "Падаешь ничком" }
      ]
    }
  }
}
```

### 4.8. progression

Механика развития персонажа — уровни, повышения. Stages — шаги wizard'а повышения уровня.

```json
{
  "progression": {
    "level_field": "level",
    "stages": [
      {
        "id": "choose_feat",
        "label": "Выберите фит",
        "type": "pick_option",
        "source": "feats",
        "required": true,
        "condition": "level % 4 == 0"
      },
      {
        "id": "hit_points",
        "label": "Повышение хитов",
        "type": "roll_and_assign",
        "roll": "{hit_die}",
        "modifier": "constitution_modifier",
        "op": "add",
        "target_field": "max_hp"
      }
    ]
  }
}
```

**Типы stages:**

| type | Описание | Специфичные поля |
|------|----------|-----------------|
| `pick_option` | Выбрать из `option_group` | `source: group_id` |
| `distribute_pool` | Потратить очки из `resource_pool` | `pool: pool_id` |
| `standard_array` | Расставить фиксированный набор значений | — |
| `roll_and_assign` | Бросить кубик и записать в поле | `roll`, `modifier`, `op`, `target_field` |
| `free_text` | Ввести произвольный текст | `target_field` |
| `compute_and_assign` | Вычислить формулу и записать | `formula`, `target_field` |
| `confirm` | Финальное подтверждение (конец wizard'а) | — |

`condition` — формула через ConditionEngine: шаг показывается только если условие истинно.

---

## 5. actions\_schema — действия

Действия персонажа — атаки, заклинания, способности. Могут тратить ресурсы и запускать броски.

```json
{
  "actions_schema": {
    "actions": [
      {
        "id": "melee_attack",
        "label": "Удар в ближнем бою",
        "description": "Атака оружием ближнего боя",
        "cost": {
          "resource": "action_points",
          "amount": 1
        },
        "trigger_roll": "attack_roll",
        "on_hit": {
          "roll_id": "damage_roll"
        },
        "effect": {
          "field": "enemy_hp",
          "op": "add",
          "formula": "-damage",
          "cap_at": "enemy_hp"
        },
        "recharge": "short_rest",
        "requires": {
          "field": "equipped_weapon",
          "condition": "equipped_weapon != 'unarmed'"
        }
      }
    ]
  }
}
```

| Поле | Описание |
|------|----------|
| `trigger_roll` | id броска из `rolls_schema` — запускается при активации. Валидатор проверяет ссылку. |
| `effect.op` | `add` или `set` |
| `effect.formula` | Формула через FormulaEngine |
| `effect.cap_at` | id поля — ограничитель результата |
| `cost.resource` | id ресурса из `resource_pools` |
| `recharge` | Условие восстановления: `"short_rest"`, `"long_rest"`, `"turn"` |

---

## 6. FormulaEngine — безопасные формулы

Формулы используются в `computed` полях, `derived`, параметрах бросков типа `computed`, эффектах опций, шагах прогрессии.

FormulaEngine разбирает формулу через AST и выполняет только разрешённые конструкции. Любой неизвестный узел → `UnsafeFormulaError`.

### 6.1. Разрешённые конструкции

**Арифметика:**

| Оператор | Пример |
|----------|--------|
| `+` `-` `*` `/` | `strength + 2` |
| `//` | `(dexterity - 10) // 2` — целочисленное деление |
| `%` | `level % 4` — остаток |
| `**` | `2 ** level` — возведение в степень |
| унарный `-` | `-strength_modifier` |

**Встроенные функции:**

| Функция | Описание |
|---------|----------|
| `floor(x)` | Округление вниз |
| `ceil(x)` | Округление вверх |
| `round(x)` | Стандартное округление |
| `max(a, b)` | Максимум из двух |
| `min(a, b)` | Минимум из двух |
| `abs(x)` | Абсолютное значение |
| `cond(c, a, b)` | Тернарный оператор: если `c` истинно — `a`, иначе `b` |

**Переменные** — только id полей из `sheet_schema`, существующие в контексте броска.

**Числовые литералы** — только `int` и `float`. Строки, списки, булевы — запрещены.

### 6.2. Запрещено

```python
# Всё нижеперечисленное вызывает UnsafeFormulaError:

__import__('os').system('rm -rf /')   # вызов функций не из whitelist
open('/etc/passwd').read()             # методы объектов
[x for x in range(10**9)]             # list comprehension
"строка"                               # строковые литералы
True or False                          # булевы литералы
lambda x: x                            # лямбды
```

### 6.3. Примеры

```python
# D&D 5e — модификатор характеристики
"floor((strength - 10) / 2)"

# Бонус мастерства по уровню
"floor((level - 1) / 4) + 2"

# Класс доспехов с ограничением
"max(10 + dexterity_modifier + armor_bonus, natural_armor)"

# Условное значение
"cond(has_shield, armor_class + 2, armor_class)"

# GURPS — базовый урон
"floor(strength / 4) - 2"

# WoD — сложность по умолчанию
"cond(animalism == 0, 10, animalism * 5)"
```

---

## 7. Минимальная рабочая система

Полная система с d20-чеком — минимальный набор для проверки загрузки.

### manifest.jsonc

```jsonc
{
  "version": "1.0.0",
  "includes": [
    "sheet/sheet.jsonc",
    "rolls/rolls.jsonc"
  ],
  "manifest": {
    "has_sheet": true,
    "has_rolls": true
  }
}
```

### sheet/sheet.jsonc

```json
{
  "sections": [
    {
      "id": "attributes",
      "label": "Атрибуты",
      "display": {
        "icon": "⚔️",
        "embed_color": "#2c3e50",
        "order": 1
      },
      "fields": [
        {
          "id": "strength",
          "label": { "en": "Strength", "ru": "Сила" },
          "type": "integer",
          "min": 1,
          "max": 20,
          "default": 10
        },
        {
          "id": "dexterity",
          "label": { "en": "Dexterity", "ru": "Ловкость" },
          "type": "integer",
          "min": 1,
          "max": 20,
          "default": 10
        },
        {
          "id": "dexterity_modifier",
          "label": { "en": "Dex Modifier", "ru": "Мод. Ловкости" },
          "type": "computed",
          "formula": "floor((dexterity - 10) / 2)"
        },
        {
          "id": "level",
          "label": "Уровень",
          "type": "integer",
          "min": 1,
          "max": 20,
          "default": 1,
          "required": true
        },
        {
          "id": "proficiency_bonus",
          "label": "Бонус мастерства",
          "type": "computed",
          "formula": "floor((level - 1) / 4) + 2"
        }
      ]
    }
  ]
}
```

### rolls/rolls.jsonc

```json
{
  "rolls_schema": {
    "ability_check": {
      "id": "ability_check",
      "label": "Проверка характеристики",
      "formula": "1d20 + dexterity_modifier",
      "parameters": [
        {
          "id": "target",
          "type": "runtime_integer",
          "label": "Сложность (DC)",
          "min": 1,
          "max": 30,
          "default": 15
        }
      ],
      "result": {
        "type": "compare",
        "outcomes": [
          { "condition": "raw_dice == 20", "label": "Критический успех" },
          { "condition": "raw_dice == 1",  "label": "Критический провал" },
          { "condition": "total >= target", "label": "Успех" },
          { "condition": "total < target",  "label": "Провал" }
        ]
      }
    }
  }
}
```

---

## 8. Ошибки валидатора

`SchemaValidator` возвращает `ValidationReport` с полями `valid`, `errors`, `warnings`. Ошибки блокируют сохранение, предупреждения — нет.

### Ошибки (errors)

| Код | Условие |
|-----|---------|
| `DUPLICATE_ID` | Два поля с одинаковым id |
| `INVALID_ID_FORMAT` | id содержит кириллицу или недопустимые символы |
| `COMPUTED_NO_FORMULA` | Поле типа `computed` без `formula` |
| `UNKNOWN_VARIABLE` | Формула ссылается на id, которого нет в `sheet_schema` |
| `SELECT_NO_OPTIONS` | Поле типа `select` или `multiselect` без `options` |
| `SKILL_NO_BASE` | Поле типа `skill` без `base_attribute` |
| `MIN_GREATER_MAX` | `min` больше `max` у поля |
| `CIRCULAR_DEPENDENCY` | Циклическая зависимость в `computed`/`derived` полях |
| `UNKNOWN_ROLL_RESULT_TYPE` | `result.type` не зарегистрирован в EngineRegistry |
| `ROLL_UNKNOWN_TABLE` | `table_lookup` ссылается на таблицу, которой нет в `rules_schema.tables` |
| `ROLL_UNKNOWN_REF` | Параметр броска `source_field` ссылается на несуществующее поле |
| `CONDITION_UNKNOWN_ROLL_REF` | Модификатор состояния `roll_ids` содержит несуществующий id броска |
| `ACTION_UNKNOWN_ROLL_REF` | `trigger_roll` действия ссылается на несуществующий бросок |
| `OPTION_EFFECT_UNKNOWN_FIELD` | Эффект опции `field` ссылается на несуществующее поле анкеты |
| `RESOURCE_POOL_UNKNOWN_TARGET` | `spend_on.targets` ссылается на несуществующее поле |
| `RESOURCE_POOL_UNKNOWN_INITIAL_FIELD` | `initial_field` ссылается на несуществующее поле |
| `ITEM_UNKNOWN_FIELD` | `on_equip.item_key` ссылается на несуществующее поле предмета |
| `TOO_MANY_SECTIONS` | Превышен лимит секций |
| `TOO_MANY_FIELDS` | Превышен лимит полей в секции |
| `TOO_MANY_ROLLS` | Превышен лимит бросков |
| `TOO_MANY_OPTIONS` | Превышен лимит опций в группе |
| `INVALID_EXPLODE_LIMIT` | `explode_limit <= 0` |

### Предупреждения (warnings)

| Код | Условие |
|-----|---------|
| `MANIFEST_MODULE_EMPTY` | `has_sheet: true`, но `sections` пуст (и аналогично для других модулей) |
| `MISSING_EXPLODE_LIMIT` | `exploding: true` без `explode_limit` — будет применён лимит 10 |
| `TOO_MANY_SECTIONS` | Количество секций превышает рекомендуемое |
| `TOO_MANY_FIELDS` | Количество полей в секции превышает рекомендуемое |
| `TOO_MANY_ROLLS` | Количество бросков превышает рекомендуемое |
| `TOO_MANY_OPTIONS` | Количество опций в группе превышает рекомендуемое |

### Hint от валидатора

При ошибках `UNKNOWN_VARIABLE`, `UNKNOWN_ROLL_RESULT_TYPE`, `ROLL_UNKNOWN_REF` и других ссылочных ошибках валидатор вычисляет расстояние Левенштейна и предлагает похожий id:

```
Формула ссылается на 'dex_modifier', которого нет в схеме.
Возможно, вы имели в виду 'dexterity_modifier'?
```

### Формат ответа API

```json
{
  "valid": false,
  "errors": [
    {
      "path": "sections.attributes.fields.strength_mod",
      "code": "UNKNOWN_VARIABLE",
      "message": "Формула ссылается на 'str', которого нет в схеме",
      "hint": "Возможно, вы имели в виду 'strength'?"
    }
  ],
  "warnings": [],
  "stats": {
    "sections_count": 1,
    "fields_count": 5,
    "computed_fields": 1,
    "rolls_count": 1,
    "has_display_config": true
  }
}
```
