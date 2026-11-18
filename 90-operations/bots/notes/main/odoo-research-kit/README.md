# Набор для исследования Odoo Community (research kit)

Готовые файлы для **отдельного** рабочего пространства, где лежит клон Odoo Community 20.0. Здесь агенты Cursor изучают модель данных и флоу Odoo и пишут справочники своими словами. Код Studio Desk здесь не пишется.

Источник подхода и решений: [`../2026-11-18-odoo-as-reference-approach.md`](../2026-11-18-odoo-as-reference-approach.md), записи 2026-11-18 в [`../../../decisions.md`](../../../decisions.md).

## Состав

```
odoo-research-kit/
├── README.md                          ← этот файл (как развернуть)
├── .cursor/rules/00-research-core.mdc ← правила для агентов (alwaysApply)
├── docs/
│   ├── README.md                      ← индекс исследований: цель, список, статусы
│   └── _template.md                   ← шаблон справочника
└── prompts/
    └── base-research-prompt.md        ← базовый промпт + пример для ACC-01
```

## Как развернуть (один раз)

1. **Клон Odoo — отдельно от Studio Desk.** Например:
   ```bash
   mkdir -p ~/Research && cd ~/Research
   git clone --branch 20.0 --single-branch https://github.com/odoo/odoo.git odoo-src
   cd odoo-src && git rev-parse --short HEAD   # запомни коммит, он нужен в каждом справочнике
   ```
   Только `odoo/odoo` (Community, LGPLv3). Репозиторий `odoo/enterprise` не клонировать и не открывать. OCA-репозитории (AGPL) не клонировать в эту папку.
   Если хочется частичный клон для экономии места: `git clone --depth 1 --branch 20.0 …` тоже подходит, коммит всё равно фиксируется.
2. **Скопировать набор в корень папки исследования** (рядом с клоном, не внутрь него, чтобы не смешивать с git Odoo):
   ```
   ~/Research/odoo-research/        ← открываешь в Cursor
   ├── .cursor/rules/00-research-core.mdc
   ├── docs/README.md, docs/_template.md
   ├── prompts/base-research-prompt.md
   └── odoo-src/                    ← клон Odoo (или симлинк на него)
   ```
   Команды:
   ```bash
   mkdir -p ~/Research/odoo-research && cd ~/Research/odoo-research
   cp -r <studio-desk-docs>/90-operations/bots/notes/main/odoo-research-kit/{.cursor,docs,prompts} .
   mv ~/Research/odoo-src ./odoo-src        # или: ln -s ~/Research/odoo-src odoo-src
   git init && git add .cursor docs prompts && git commit -m "research kit"   # своя история справочников
   echo "odoo-src/" >> .gitignore
   ```
   Свой git для `docs/` удобен: видно, что агент изменил в справочниках.
3. **Открывать `~/Research/odoo-research` в отдельном окне Cursor.** Не добавлять её в multi-root вместе со studio-desk-frontend / backend / docs, и наоборот: в окне Studio Desk этой папки быть не должно. Так агенты, пишущие наш код, не видят код Odoo.
4. Проверить, что правило подхватилось: в новом чате агента спросить «какие правила действуют в этом workspace» — он должен пересказать `00-research-core`.

## Как работать

1. В `docs/README.md` выбрать следующий справочник по приоритету (статус `planned`).
2. Открыть новый чат агента, вставить `prompts/base-research-prompt.md`, заполнить плейсхолдеры (для ACC-01 есть готовый пример внизу файла).
3. Агент пишет `docs/<ID>-<slug>.md` по шаблону и обновляет статус в индексе.
4. Ты читаешь, правишь, ставишь статус `reviewed`.
5. **Перенос:** готовый справочник копируешь в `studio-desk-docs/research/odoo/` (эту папку ведёшь ты; боты туда не пишут). Перед переносом проверь: нет кусков кода/XML/CSV Odoo, есть коммит и пути, есть раздел про Studio Desk. После переноса статус `moved`.
6. Выводы «берём/меняем» ты переносишь в канон Studio Desk (01–03). Агенты-исполнители пишут модуль `accounting` только по канону, без доступа к этой папке.

Справочник пишется тогда, когда до его темы доходит канон; собирать всё сразу не нужно. Одна тема — один справочник.
