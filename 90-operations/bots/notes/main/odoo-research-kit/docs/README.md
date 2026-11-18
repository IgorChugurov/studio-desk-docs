# Справочники по Odoo Community — индекс

## Цель проекта

Постепенно и полностью описать предметную и бизнес-область **Odoo Community 20.0** (LGPLv3): сначала бухгалтерию, затем другие открытые модули. Описываем модели данных, состояния, флоу, инварианты и крайние случаи **своими словами** и сразу сопоставляем со Studio Desk.

**Что получаем в итоге:** набор справочников в `studio-desk-docs/research/odoo/`. По ним Игорь:
- пишет канон модуля `accounting` (ядро платформы, схема `accounting`, NestJS) и других разделов;
- берёт идеи или берёт модель Odoo за основу и адаптирует её к нашим процессам (`Visit`, абонементы, выплаты, `accounting_events`, мультитенантность);
- видит, чего в Odoo нет и что придётся строить по закону (SK, затем UA).

Правила работы — `.cursor/rules/00-research-core.mdc`. Шаблон — `_template.md`. Промпт — `../prompts/base-research-prompt.md`.

## Статусы

`planned` → `in-progress` → `draft` (агент закончил) → `reviewed` (Игорь проверил) → `moved` (перенесено в `studio-desk-docs/research/odoo/`).

Приоритет: **P1** нужно для первого канона accounting (SK SZČO); **P2** для абонементов, кассы и e-faktúra; **P3** для других разделов платформы; **P4** позже.

## A. Ядро бухгалтерии (`account`)

| ID | Тема | Модули / файлы Odoo | Зачем нам | Приоритет | Статус |
|---|---|---|---|---|---|
| ACC-01 | Ядро проводок: `account.move` и `account.move.line` | `account` (`models/account_move.py`, `account_move_line.py`) | Шапка и строки, баланс = 0, типы документов, связь документа и проводки; основа журнала `accounting` | P1 | planned |
| ACC-02 | Журналы и план счетов | `account` (`account_journal.py`, `account_account.py`, `company.py`) | Журналы организации, типы счетов, что из этого задаёт пакет юрисдикции | P1 | planned |
| ACC-03 | Проведение, сторно, lock dates, нумерация, хеш неизменяемости | `account` (`account_move.py`, `sequence_mixin.py`, `res_company` lock dates, `account_move_reversal` wizard) | Неизменяемость, ±1 сторно, закрытие периода, нумерация документов по закону | P1 | planned |
| ACC-04 | Налоги: `account.tax`, распределение, теги отчёта, exigibility | `account` (`account_tax.py`, repartition lines, `account.account.tag`), `account_tax_python` (только идея) | Как налог раскладывается по счетам и строкам декларации; DPH для SK | P1 | planned |
| ACC-05 | Фискальные позиции | `account` (`account_fiscal_position.py`) | Плательщик / неплательщик DPH, клиент из ЕС, замена налогов и счетов | P1 | planned |
| ACC-06 | Условия оплаты и сроки | `account` (`account_payment_term.py`) | Сроки дебиторки, частичная оплата | P2 | planned |
| ACC-07 | Оплаты и сопоставление (reconciliation) | `account` (`account_payment.py`, `account_partial_reconcile.py`, `account_full_reconcile.py`), `account_payment` | Оплата → проводка, частичная оплата, переплата, связь оплаты с начислением и абонементом | P1 | planned |
| ACC-08 | Валюты и курсовые разницы | `base` (`res.currency`, `res.currency.rate`), `account` (exchange difference в `account_move_line.py`) | EUR/UAH, курс на дату, авто-проводка разницы | P2 | planned |
| ACC-09 | Банковские выписки и касса в `account` | `account` (`account_bank_statement*.py`) | Импорт выписки, ручная касса, сверка | P2 | planned |
| ACC-10 | Аналитика | `analytic` (`analytic_account`, `analytic_plan`, `analytic_distribution`) | Разрезы: филиал, специалист, услуга, раздел | P2 | planned |
| ACC-11 | Кредит-ноты, дебет-ноты, возвраты | `account` (`out_refund`/`in_refund`), `account_debit_note` | Возврат за визит или абонемент, корректировочные документы | P2 | planned |
| ACC-12 | Определения отчётов (`account.report`) | `account` (`account_report.py` и data отчётов — только структура) | Как устроены баланс, P&L, налоговые отчёты на уровне модели (рендер в Enterprise — не открываем) | P2 | planned |
| ACC-13 | Мультикомпания и права в бухгалтерии | `base` (`res.company`, record rules), `account` security | Чем это отличается от нашего `organization_id`, что не брать | P2 | planned |

## B. Соседние модули, важные для денег

| ID | Тема | Модули Odoo | Зачем нам | Приоритет | Статус |
|---|---|---|---|---|---|
| PRD-01 | Товары/услуги, варианты, единицы, прайс-листы | `product`, `uom` | Каталог услуг и цен, связь с налогом и счётом дохода | P2 | planned |
| PTN-01 | Контрагенты | `base` (`res.partner`), `contacts` | Клиент / поставщик / специалист, IČO, DIČ, IČ DPH, адреса | P2 | planned |
| LOY-01 | Loyalty как остаток со сроком сгорания | `loyalty` (`loyalty.program`, `loyalty.card`, `loyalty.history`), `sale_loyalty`, `pos_loyalty` | Образец абонемента/кошелька: начисление, списание, сгорание, журнал движения | P2 | planned |
| POS-01 | Кассовые сессии POS | `point_of_sale` (`pos.session`, `pos.order`, `pos.payment`) | Кассовая смена, закрытие с проводками, наличные; база для eKasa-слоя | P2 | planned |
| EDI-01 | UBL / CII / Peppol BIS 3 | `account_edi_ubl_cii` (`account_edi_xml_ubl_bis3.py`, `account_edi_ubl_cen_en16931.py`), `account_peppol` (только формат) | e-faktúra SK с 1.1.2027 (B2B Peppol) | P2 | planned |
| PAY-01 | Онлайн-платежи | `payment` (`payment.transaction`, провайдеры) | Онлайн-оплата абонементов на публичном сайте | P3 | planned |
| SALE-01 | Продажа и счёт из заказа | `sale` | Как заказ превращается в счёт; аналог `direct_check` / `external_order` | P3 | planned |

## C. Другие разделы платформы

| ID | Тема | Модули Odoo | Зачем нам | Приоритет | Статус |
|---|---|---|---|---|---|
| RES-01 | Ресурсы и рабочие календари | `resource`, `calendar`, `hr_holidays` | Рабочие часы, отпуска, слоты для расписания | P3 | planned |
| EVT-01 | События и регистрации с отметкой «посетил» | `event` (`event.registration`) | Аналог `Booking` → `Visit` для групповых занятий | P3 | planned |
| HR-01 | Расходы сотрудников | `hr_expense` | Флоу «расход → утверждение → проводка»; образец для выплат и расходов | P3 | planned |
| HR-02 | Сотрудники и учёт времени | `hr`, `hr_work_entry`, `hr_timesheet` | Основа начислений специалистам (payroll — Enterprise, не открываем) | P3 | planned |
| WO-01 | Заказ-наряды (ремонт) | `repair` | Шаблон work-order для Auto Service: работы + запчасти | P3 | planned |
| STK-01 | Склад и оценка запасов | `stock`, `stock_account` | Запчасти и расходники, проводки по складу | P4 | planned |
| PRT-01 | Портал клиента | `portal` | Кабинет клиента, доступ к документам и оплатам | P4 | planned |
| SRV-01 | Анкеты | `survey` | Анкеты для `Chart` в Medical / Beauty | P4 | planned |
| DOC-01 | PDF-отчёты и вложения | `ir.actions.report` (base), `mail` attachments | Генерация документов | P4 | planned |

Новые темы, найденные агентами, дописываются в конец нужной группы со статусом `planned` и пометкой «найдено в <ID>».

## D. Чего в Odoo Community нет — исследуем по закону, не по Odoo

Эти темы не пишутся по коду Odoo. Источники: законы и официальные сайты, вопросы боту «Налоги и бухгалтерия SK», проверка живым бухгалтером. Здесь они перечислены, чтобы помнить о пробелах.

| ID | Тема | Юрисдикция | Почему не из Odoo | Статус |
|---|---|---|---|---|
| LAW-SK-01 | SZČO: podvojné vs jednoduché účtovníctvo, daňová evidencia, paušálne výdavky | SK | Нет в Odoo | planned |
| LAW-SK-02 | DPH: регистрация, ставки, декларация (структура в `l10n_sk` есть только для сверки) | SK | Ставки и формы — из закона | planned |
| LAW-SK-03 | Kontrolný výkaz (до отмены с 1.7.2030) | SK | Нет в Community | planned |
| LAW-SK-04 | eKasa (обязательна с 1.1.2026, программная касса, сертификация) | SK | Нет в Odoo | planned |
| LAW-SK-05 | e-faktúra B2B через Peppol с 1.1.2027, Digitálny poštár | SK | Odoo даёт только формат UBL (EDI-01), не доставку | planned |
| LAW-SK-06 | Подача в eDane, daňové priznanie FO typ B, odvody (SP/ZP) | SK | Нет в Odoo | planned |
| LAW-SK-07 | План счетов (rámcová účtová osnova) и формы výkazov | SK | Данные — из нормативного акта | planned |
| LAW-UA-01 | ФОП (группы единого налога), ТОВ: что и как отчитывается | UA | Нет в Odoo | planned |
| LAW-UA-02 | ПРРО, податкові накладні, ЄРПН | UA | Нет в Odoo | planned |
| LAW-UA-03 | XML-формы ДПС, КЕП, подача | UA | Нет в Odoo | planned |
| LAW-UA-04 | План рахунків (наказ Мінфіну № 291) | UA | Данные — из нормативного акта | planned |

**Только Enterprise — образца нет и не открываем:** `account_accountant` (UI сверки, deferred revenue), `account_reports` (рендер отчётов), бюджеты, консолидация, OCR, банк-синк, `sale_subscription`, `appointment`, `planning`, `hr_payroll`, Documents, Sign, quality, field service, `website_appointment`. Для этих тем пишем требования сами (из наших процессов и закона) или читаем только публичную пользовательскую документацию odoo.com.
