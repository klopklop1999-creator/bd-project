# Описание предметной области и карта данных

Источник: датасет Lending Club (`accepted_2007_to_2018Q4.csv`), 2 260 668 строк, 151 колонка, дубликатов по `id` нет.

## 1. Описание предметной области

Датасет описывает портфель потребительских займов P2P-платформы Lending Club за 2007–2018 годы: заявку заёмщика, параметры выданного займа, кредитный профиль из бюро и историю погашения. Автоматизируются процессы оценки заявки (присвоение рейтинга `grade`/`sub_grade` и ставки), выдачи займа, учёта платежей, работы с просрочкой (реструктуризация `hardship`, урегулирование `settlement`) и взыскания (`recoveries`). Пользователи системы: кредитные аналитики, операторы выдачи и обслуживания, кредитный комитет, риск-менеджеры и специалисты по взысканию, администратор БД. На основе данных принимаются решения об одобрении займа, ставке и рейтинге, лимитах по целям и регионам, резервах под потери и стратегии работы с проблемными займами. В датасете нет сведений об обеспечении и поручителях, поэтому для соответствия заданию добавлены синтетические таблицы `collaterals` и `guarantors`, а также прямые идентификаторы заёмщика `borrower_pii`.

## 2. Сущности (15 таблиц)

| Сущность | Назначение | Ключевые атрибуты | PK | Внешние ключи |
|---|---|---|---|---|
| `us_states` | Справочник штатов | `state_code`, `state_name` | `state_code` | нет |
| `loan_grades` | Справочник рейтингов займа | `sub_grade`, `grade` | `sub_grade` | нет |
| `loan_purposes` | Справочник целей займа | `purpose_code` | `purpose_id` | нет |
| `loan_statuses` | Справочник статусов займа | `status_name` | `status_id` | нет |
| `borrowers` | Заёмщик (косвенные идентификаторы) | `identity_key`, `state_code`, `zip3`, `emp_title` | `borrower_id` | `state_code` → `us_states` |
| `borrower_pii` | Прямые идентификаторы заёмщика (синтетика) | `full_name`, `birth_date`, `phone`, `email` | `borrower_id` | `borrower_id` → `borrowers` |
| `loans` | Кредитный договор | `loan_amnt`, `term_months`, `int_rate`, `issue_date`, `annual_inc` | `loan_id` | `borrower_id`, `sub_grade`, `purpose_id`, `status_id` |
| `credit_profiles` | Кредитный профиль на момент заявки | `fico_range_*`, `delinq_2yrs`, `bureau_extra` | `loan_id` | `loan_id` → `loans` |
| `loan_payments` | Состояние платежей по договору | `total_pymnt`, `total_rec_prncp`, `total_rec_int`, `recoveries` | `loan_id` | `loan_id` → `loans` |
| `hardship_plans` | План реструктуризации | `hardship_type`, `hardship_reason`, `hardship_status` | `hardship_id` | `loan_id` → `loans` (UNIQUE) |
| `debt_settlements` | Урегулирование задолженности | `settlement_status`, `settlement_amount` | `settlement_id` | `loan_id` → `loans` (UNIQUE) |
| `joint_applicants` | Созаёмщик (только `Joint App`) | `annual_inc_joint`, `dti_joint`, `sec_app_*` | `loan_id` | `loan_id` → `loans` |
| `collaterals` | Залог (синтетика) | `collateral_type`, `market_value` | `collateral_id` | `loan_id` → `loans` |
| `guarantors` | Поручитель (синтетика) | `full_name`, `passport_data`, `guarantee_amount` | `guarantor_id` | `loan_id` → `loans` |
| `audit_log` | Журнал аудита | `table_name`, `operation`, `old_data`, `new_data` | `log_id` | нет |

### Обоснование решений по проектированию

- **Справочники** (`loan_grades`, `loan_purposes`, `loan_statuses`, `us_states`) убирают повторение строк в 2,26 млн записей, экономят место и дают целостность через FK (3НФ).
- **`loans` и `loan_payments` разделены (1:1).** Параметры договора неизменны, платежи обновляются ежемесячно, а права доступа к ним разные.
- **`borrowers` и `borrower_pii` разделены.** Прямые идентификаторы хранятся отдельно, чтобы аналитик работал только с `borrowers` (минимизация данных по 152-ФЗ).
- **`bureau_extra jsonb` + GIN-индекс.** 51 редкий показатель бюро (заполнены на 23–97%) не стоит выносить в 51 колонку. В JSONB сохраняются все данные, а GIN даёт поиск по ключам.
- **Типы данных.** Деньги: `numeric(p,2)`, без `float`. Даты: `date` вместо строк `Dec-2015`. Небольшие счётчики: `smallint`. Справочные значения: `char`/`varchar` с CHECK.

## 3. Карта данных

Обозначения: «Источник» это `LC: колонка` (исходный датасет Lending Club) или «Синтетика». Категория: **ПДн** (персональные данные), **КТ** (коммерческая тайна), **Откр.** (открытые).

| Источник | Сущность | Атрибут | Тип данных | Пример | Обяз. | Уник. | Примечание |
|---|---|---|---|---|---|---|---|
| LC: addr_state | us_states | state_code | char(2) | PA | Да | Да | Откр. |
| Справочник | us_states | state_name | varchar(60) | Pennsylvania | Да | Да | Откр. |
| LC: sub_grade | loan_grades | sub_grade | char(2) | C4 | Да | Да | КТ (методика скоринга) |
| LC: grade | loan_grades | grade | char(1) | C | Да | Нет | КТ |
| Суррогат | loan_purposes | purpose_id | smallint | 3 | Да | Да | Откр. |
| LC: purpose | loan_purposes | purpose_code | varchar(30) | debt_consolidation | Да | Да | Откр. |
| Суррогат | loan_statuses | status_id | smallint | 1 | Да | Да | Откр. |
| LC: loan_status | loan_statuses | status_name | varchar(70) | Fully Paid | Да | Да | Откр. |
| Суррогат | borrowers | borrower_id | bigint | 1048 | Да | Да | КТ (внутренний ключ) |
| Вычисляется | borrowers | identity_key | char(32) | 9e107d9d372bb682… | Да | Да | КТ |
| LC: addr_state | borrowers | state_code | char(2) | PA | Да | Нет | ПДн (косвенный) |
| LC: zip_code | borrowers | zip3 | char(3) | 190 | Да | Нет | ПДн (косвенный) |
| LC: earliest_cr_line | borrowers | earliest_cr_line | date | 2003-08-01 | Нет | Нет | ПДн (косвенный) |
| LC: emp_title | borrowers | emp_title | text | leadman | Нет | Нет | ПДн (косвенный) |
| Системное | borrowers | created_at | timestamptz | 2026-11-03 12:00:00+03 | Да | Нет | Откр. |
| Синтетика | borrower_pii | borrower_id | bigint | 1048 | Да | Да | ПДн |
| Синтетика | borrower_pii | full_name | varchar(150) | Ivan Petrov | Да | Нет | ПДн |
| Синтетика | borrower_pii | birth_date | date | 1984-05-17 | Да | Нет | ПДн |
| Синтетика | borrower_pii | phone | varchar(20) | +79001234567 | Да | Да | ПДн |
| Синтетика | borrower_pii | email | varchar(120) | i.petrov@example.com | Да | Да | ПДн |
| LC: id | loans | loan_id | bigint | 68407277 | Да | Да | КТ |
| Связь | loans | borrower_id | bigint | 1048 | Да | Нет | КТ (FK) |
| LC: sub_grade | loans | sub_grade | char(2) | C4 | Да | Нет | КТ (FK) |
| LC: purpose | loans | purpose_id | smallint | 3 | Да | Нет | Откр. (FK) |
| LC: loan_status | loans | status_id | smallint | 1 | Да | Нет | КТ (FK) |
| LC: loan_amnt | loans | loan_amnt | numeric(10,2) | 3600.00 | Да | Нет | КТ |
| LC: funded_amnt | loans | funded_amnt | numeric(10,2) | 3600.00 | Да | Нет | КТ |
| LC: funded_amnt_inv | loans | funded_amnt_inv | numeric(10,2) | 3600.00 | Нет | Нет | КТ |
| LC: term | loans | term_months | smallint | 36 | Да | Нет | КТ |
| LC: int_rate | loans | int_rate | numeric(5,2) | 13.99 | Да | Нет | КТ |
| LC: installment | loans | installment | numeric(8,2) | 123.03 | Да | Нет | КТ |
| LC: issue_d | loans | issue_date | date | 2015-12-01 | Да | Нет | КТ |
| LC: annual_inc | loans | annual_inc | numeric(14,2) | 55000.00 | Да | Нет | ПДн (финансовые сведения) |
| LC: emp_length | loans | emp_length_years | smallint | 10 | Нет | Нет | ПДн (косвенный) |
| LC: home_ownership | loans | home_ownership | varchar(10) | MORTGAGE | Да | Нет | ПДн (косвенный) |
| LC: verification_status | loans | verification_status | varchar(20) | Not Verified | Да | Нет | КТ |
| LC: application_type | loans | application_type | varchar(12) | Individual | Да | Нет | КТ |
| LC: dti | loans | dti | numeric(7,2) | 5.91 | Нет | Нет | КТ |
| LC: initial_list_status | loans | initial_list_status | char(1) | w | Нет | Нет | КТ |
| LC: disbursement_method | loans | disbursement_method | varchar(10) | Cash | Нет | Нет | КТ |
| Связь | credit_profiles | loan_id | bigint | 68407277 | Да | Да | КТ (FK, PK) |
| LC: fico_range_low | credit_profiles | fico_range_low | smallint | 675 | Да | Нет | ПДн (кредитная история) |
| LC: fico_range_high | credit_profiles | fico_range_high | smallint | 679 | Да | Нет | ПДн (кредитная история) |
| LC: delinq_2yrs | credit_profiles | delinq_2yrs | smallint | 0 | Нет | Нет | ПДн (кредитная история) |
| LC: inq_last_6mths | credit_profiles | inq_last_6mths | smallint | 1 | Нет | Нет | ПДн (кредитная история) |
| LC: mths_since_last_delinq | credit_profiles | mths_since_last_delinq | smallint | 30 | Нет | Нет | ПДн (кредитная история) |
| LC: mths_since_last_record | credit_profiles | mths_since_last_record | smallint | 106 | Нет | Нет | ПДн (кредитная история) |
| LC: open_acc | credit_profiles | open_acc | smallint | 7 | Нет | Нет | ПДн (кредитная история) |
| LC: pub_rec | credit_profiles | pub_rec | smallint | 0 | Нет | Нет | ПДн (кредитная история) |
| LC: revol_bal | credit_profiles | revol_bal | numeric(12,2) | 2765.00 | Нет | Нет | ПДн (кредитная история) |
| LC: revol_util | credit_profiles | revol_util | numeric(6,2) | 29.70 | Нет | Нет | ПДн (кредитная история) |
| LC: total_acc | credit_profiles | total_acc | smallint | 13 | Нет | Нет | ПДн (кредитная история) |
| LC: collections_12_mths_ex_med | credit_profiles | collections_12_mths_ex_med | smallint | 0 | Нет | Нет | ПДн (кредитная история) |
| LC: acc_now_delinq | credit_profiles | acc_now_delinq | smallint | 0 | Нет | Нет | ПДн (кредитная история) |
| LC: tot_coll_amt | credit_profiles | tot_coll_amt | numeric(12,2) | 722.00 | Нет | Нет | ПДн (кредитная история) |
| LC: tot_cur_bal | credit_profiles | tot_cur_bal | numeric(12,2) | 144904.00 | Нет | Нет | ПДн (кредитная история) |
| LC: pub_rec_bankruptcies | credit_profiles | pub_rec_bankruptcies | smallint | 0 | Нет | Нет | ПДн (кредитная история) |
| LC: tax_liens | credit_profiles | tax_liens | smallint | 0 | Нет | Нет | ПДн (кредитная история) |
| LC: 51 колонка бюро (см. раздел 5) | credit_profiles | bureau_extra | jsonb | {"open_acc_6m": 2, …} | Нет | Нет | ПДн (кредитная история) |
| Связь | loan_payments | loan_id | bigint | 68407277 | Да | Да | КТ (FK, PK) |
| LC: out_prncp | loan_payments | out_prncp | numeric(12,2) | 0.00 | Да | Нет | КТ |
| LC: out_prncp_inv | loan_payments | out_prncp_inv | numeric(12,2) | 0.00 | Нет | Нет | КТ |
| LC: total_pymnt | loan_payments | total_pymnt | numeric(14,2) | 4421.72 | Да | Нет | КТ |
| LC: total_pymnt_inv | loan_payments | total_pymnt_inv | numeric(14,2) | 4421.72 | Нет | Нет | КТ |
| LC: total_rec_prncp | loan_payments | total_rec_prncp | numeric(12,2) | 3600.00 | Да | Нет | КТ |
| LC: total_rec_int | loan_payments | total_rec_int | numeric(12,2) | 821.72 | Да | Нет | КТ |
| LC: total_rec_late_fee | loan_payments | total_rec_late_fee | numeric(10,2) | 0.00 | Да | Нет | КТ |
| LC: recoveries | loan_payments | recoveries | numeric(12,2) | 0.00 | Да | Нет | КТ |
| LC: collection_recovery_fee | loan_payments | collection_recovery_fee | numeric(12,2) | 0.00 | Да | Нет | КТ |
| LC: last_pymnt_d | loan_payments | last_pymnt_date | date | 2019-01-01 | Нет | Нет | КТ |
| LC: last_pymnt_amnt | loan_payments | last_pymnt_amnt | numeric(12,2) | 122.67 | Нет | Нет | КТ |
| LC: next_pymnt_d | loan_payments | next_pymnt_date | date | 2019-04-01 | Нет | Нет | КТ |
| LC: last_credit_pull_d | loan_payments | last_credit_pull_date | date | 2019-03-01 | Нет | Нет | КТ |
| LC: last_fico_range_low | loan_payments | last_fico_range_low | smallint | 560 | Нет | Нет | ПДн (кредитная история) |
| LC: last_fico_range_high | loan_payments | last_fico_range_high | smallint | 564 | Нет | Нет | ПДн (кредитная история) |
| Суррогат | hardship_plans | hardship_id | bigint | 17 | Да | Да | КТ |
| Связь | hardship_plans | loan_id | bigint | 68407277 | Да | Да | КТ (FK) |
| LC: hardship_type | hardship_plans | hardship_type | varchar(60) | INTEREST ONLY-3 MONTHS DEFERRAL | Нет | Нет | КТ |
| LC: hardship_reason | hardship_plans | hardship_reason | varchar(40) | NATURAL_DISASTER | Нет | Нет | КТ |
| LC: hardship_status | hardship_plans | hardship_status | varchar(20) | BROKEN | Нет | Нет | КТ |
| LC: deferral_term | hardship_plans | deferral_term | smallint | 3 | Нет | Нет | КТ |
| LC: hardship_amount | hardship_plans | hardship_amount | numeric(10,2) | 47.01 | Нет | Нет | КТ |
| LC: hardship_start_date | hardship_plans | hardship_start_date | date | 2017-09-01 | Нет | Нет | КТ |
| LC: hardship_end_date | hardship_plans | hardship_end_date | date | 2017-12-01 | Нет | Нет | КТ |
| LC: payment_plan_start_date | hardship_plans | payment_plan_start_date | date | 2017-10-01 | Нет | Нет | КТ |
| LC: hardship_length | hardship_plans | hardship_length | smallint | 3 | Нет | Нет | КТ |
| LC: hardship_dpd | hardship_plans | hardship_dpd | smallint | 26 | Нет | Нет | КТ |
| LC: hardship_loan_status | hardship_plans | hardship_loan_status | varchar(30) | Late (16-30 days) | Нет | Нет | КТ |
| LC: orig_projected_additional_accrued_interest | hardship_plans | orig_projected_additional_accrued_interest | numeric(10,2) | 442.41 | Нет | Нет | КТ |
| LC: hardship_payoff_balance_amount | hardship_plans | hardship_payoff_balance_amount | numeric(12,2) | 3058.14 | Нет | Нет | КТ |
| LC: hardship_last_payment_amount | hardship_plans | hardship_last_payment_amount | numeric(12,2) | 61.23 | Нет | Нет | КТ |
| Суррогат | debt_settlements | settlement_id | bigint | 5 | Да | Да | КТ |
| Связь | debt_settlements | loan_id | bigint | 68407277 | Да | Да | КТ (FK) |
| LC: debt_settlement_flag_date | debt_settlements | debt_settlement_flag_date | date | 2017-11-01 | Нет | Нет | КТ |
| LC: settlement_status | debt_settlements | settlement_status | varchar(15) | COMPLETE | Нет | Нет | КТ |
| LC: settlement_date | debt_settlements | settlement_date | date | 2017-09-01 | Нет | Нет | КТ |
| LC: settlement_amount | debt_settlements | settlement_amount | numeric(12,2) | 9915.00 | Нет | Нет | КТ |
| LC: settlement_percentage | debt_settlements | settlement_percentage | numeric(6,2) | 55.00 | Нет | Нет | КТ |
| LC: settlement_term | debt_settlements | settlement_term | smallint | 1 | Нет | Нет | КТ |
| Связь | joint_applicants | loan_id | bigint | 68341763 | Да | Да | ПДн (FK) |
| LC: annual_inc_joint | joint_applicants | annual_inc_joint | numeric(14,2) | 71000.00 | Нет | Нет | ПДн (финансовые сведения) |
| LC: dti_joint | joint_applicants | dti_joint | numeric(7,2) | 13.85 | Нет | Нет | ПДн (финансовые сведения) |
| LC: verification_status_joint | joint_applicants | verification_status_joint | varchar(20) | Not Verified | Нет | Нет | КТ |
| LC: revol_bal_joint | joint_applicants | revol_bal_joint | numeric(12,2) | 7101.00 | Нет | Нет | ПДн (кредитная история) |
| LC: sec_app_fico_range_low / _high | joint_applicants | sec_app_fico_range_low, sec_app_fico_range_high | smallint | 610 / 614 | Нет | Нет | ПДн (кредитная история) |
| LC: sec_app_earliest_cr_line | joint_applicants | sec_app_earliest_cr_line | date | 2005-02-01 | Нет | Нет | ПДн (косвенный) |
| LC: sec_app_inq_last_6mths, sec_app_mort_acc, sec_app_open_acc, sec_app_open_act_il, sec_app_num_rev_accts, sec_app_chargeoff_within_12_mths, sec_app_collections_12_mths_ex_med, sec_app_mths_since_last_major_derog | joint_applicants | sec_app_inq_last_6mths, sec_app_mort_acc, sec_app_open_acc, sec_app_open_act_il, sec_app_num_rev_accts, sec_app_chargeoff_within_12_mths, sec_app_collections_12_mths_ex_med, sec_app_mths_since_last_major_derog | smallint | 3 / 1 / 14 / 11 / 8 / 0 / 2 / 37 | Нет | Нет | ПДн (кредитная история) |
| LC: sec_app_revol_util | joint_applicants | sec_app_revol_util | numeric(6,2) | 80.00 | Нет | Нет | ПДн (кредитная история) |
| Суррогат | collaterals | collateral_id | bigint | 902 | Да | Да | КТ |
| Связь | collaterals | loan_id | bigint | 68407277 | Да | Нет | КТ (FK) |
| Синтетика | collaterals | collateral_type | varchar(20) | REAL_ESTATE | Да | Нет | КТ |
| Синтетика | collaterals | market_value | numeric(12,2) | 85000.00 | Да | Нет | КТ |
| Синтетика | collaterals | appraisal_date | date | 2015-11-20 | Да | Нет | КТ |
| Синтетика | collaterals | description | text | Квартира, 2 комнаты | Нет | Нет | КТ |
| Суррогат | guarantors | guarantor_id | bigint | 311 | Да | Да | ПДн |
| Связь | guarantors | loan_id | bigint | 68407277 | Да | Нет | ПДн (FK) |
| Синтетика | guarantors | full_name | varchar(150) | Anna Smirnova | Да | Нет | ПДн |
| Синтетика | guarantors | birth_date | date | 1979-02-03 | Да | Нет | ПДн |
| Синтетика | guarantors | phone | varchar(20) | +79007654321 | Да | Нет | ПДн |
| Синтетика | guarantors | passport_data | varchar(11) | 4510 123456 | Да | Да (с loan_id) | ПДн |
| Синтетика | guarantors | guarantee_amount | numeric(10,2) | 2500.00 | Да | Нет | КТ |
| Суррогат | audit_log | log_id | bigint | 1 | Да | Да | КТ |
| Триггер | audit_log | table_name | text | borrower_pii | Да | Нет | КТ |
| Триггер | audit_log | operation | char(1) | U | Да | Нет | КТ |
| Триггер | audit_log | record_pk | text | 1048 | Нет | Нет | КТ |
| Триггер | audit_log | changed_by | name | credit_operator | Да | Нет | КТ |
| Триггер | audit_log | changed_at | timestamptz | 2026-11-03 12:30:00+03 | Да | Нет | КТ |
| Триггер | audit_log | old_data | jsonb | {"phone": "…"} | Нет | Нет | ПДн / КТ (содержит копии данных) |
| Триггер | audit_log | new_data | jsonb | {"phone": "…"} | Нет | Нет | ПДн / КТ (содержит копии данных) |

## 4. Классификация данных

### Персональные данные (152-ФЗ, ст. 3: информация, относящаяся к прямо или косвенно определённому физическому лицу)

- **Прямые идентификаторы (синтетика):** `borrower_pii.*` (ФИО, дата рождения, телефон, email), `guarantors.full_name`, `birth_date`, `phone`, `passport_data`. Паспортные данные требуют наивысшей защиты.
- **Косвенные идентификаторы:** `borrowers.zip3`, `emp_title`, `earliest_cr_line`, `loans.emp_length_years`, `home_ownership`. Сочетание индекса, работодателя и даты первой кредитной линии позволяет выделить человека, поэтому комбинация считается ПДн.
- **Финансовые сведения о лице:** `loans.annual_inc`, `joint_applicants.annual_inc_joint`, `dti_joint`.
- **Кредитная история:** `credit_profiles.*` (включая `bureau_extra`), `loan_payments.last_fico_*`, `joint_applicants.sec_app_*`, `revol_bal_joint`.
- **`borrowers.state_code`** сам по себе слишком грубый признак и не идентифицирует лицо. Он помечен как косвенный только потому, что входит в комбинацию с `zip3`.

### Коммерческая тайна (98-ФЗ «О коммерческой тайне»)

- **Методика скоринга и ценообразования:** `loan_grades.*`, `loans.int_rate`, `sub_grade`, `installment`.
- **Условия и объёмы портфеля:** `loans.loan_amnt`, `funded_amnt`, `funded_amnt_inv`, `term_months`, `issue_date`, `status_id`.
- **Финансовый результат и платежи:** `loan_payments.*` (кроме `last_fico_*`), `hardship_plans.*`, `debt_settlements.*`.
- **Обеспечение и поручительство:** `collaterals.*`, `guarantors.guarantee_amount`.
- **Служебное:** `audit_log` (кто и что менял), `borrowers.identity_key` (алгоритм связывания заёмщиков).

### Открытые данные

- Справочники: `us_states`, `loan_purposes`.
- Технические атрибуты: `borrowers.created_at`, суррогатные ключи справочников.

### Минимизация данных (учтено при проектировании)

- Исключены свободные тексты `desc` и `title`, где заёмщики могут написать что угодно, включая реальные имена и контакты.
- Исключён `url` (содержит `loan_id`, но ничего не добавляет к данным).
- Индекс сокращён до `zip3` (в исходных данных уже 3 цифры + `xx`).
- Прямые идентификаторы вынесены из `borrowers` в `borrower_pii`.
- **Важно:** данные Lending Club публичные и обезличенные. Таблицы `borrower_pii` и `guarantors` синтетические, настоящих ПДн в проекте нет. Классификация ПДн строится так, как если бы это была реальная система.

## 5. Допущения и преобразования при загрузке

### Определение заёмщика

Колонка `member_id` пуста (0% заполнения), поэтому настоящего идентификатора заёмщика в датасете нет. Заёмщик определяется **эвристически**: `identity_key = md5(lower(addr_state) | zip3 | earliest_cr_line | lower(emp_title))`. Один ключ означает одного человека с одним и тем же штатом, индексом, датой первой кредитной линии и работодателем.

Ограничения подхода: у 7,4% записей `emp_title` пуст, поэтому для них ключ слабее. Возможны ложные склейки разных людей и разрывы (человек сменил работу). Это допущение нужно зафиксировать в отчёте и озвучить на защите.

### Преобразования

| Колонка | Преобразование |
|---|---|
| `id` | строка → `bigint`, `loan_id` |
| `term` (` 36 months`) | убрать пробел и слово → `36` |
| `issue_d`, `earliest_cr_line`, `last_pymnt_d`, `next_pymnt_d`, `last_credit_pull_d`, `hardship_*_date`, `payment_plan_start_date`, `debt_settlement_flag_date`, `settlement_date`, `sec_app_earliest_cr_line` (`Dec-2015`) | → `date`, первое число месяца (`2015-12-01`). В источнике точность до месяца |
| `emp_length` (`10+ years`, `< 1 year`, `3 years`) | `10+ years` → 10, `< 1 year` → 0, остальные → число. Пусто → `NULL` |
| `zip_code` (`190xx`) | первые 3 символа → `zip3` |
| `dti` | отрицательные значения → `NULL` (в схеме `CHECK (dti >= 0)`) |
| Float-колонки с целыми значениями (`3600.0`, `675.0`) | → `numeric` / `smallint` |
| `fico_range_*` | значение вне диапазона 300–850 → строка отбраковывается и логируется |

### Колонки, не попавшие в схему

| Колонка | Причина |
|---|---|
| `member_id` | пустая (0%) |
| `policy_code` | константа (1 уникальное значение) |
| `url` | производная от `id` |
| `desc` | свободный текст, заполнено 5,6%, риск ПДн |
| `title` | свободный текст, дублирует `purpose`, риск ПДн |
| `pymnt_plan` | почти константа, не нужна для задач |
| `hardship_flag` | выводится из наличия строки в `hardship_plans` |
| `debt_settlement_flag` | выводится из наличия строки в `debt_settlements` |
| `grade` | выводится из `sub_grade` через `loan_grades` |

### Состав `bureau_extra` (51 колонка)

`open_acc_6m`, `open_act_il`, `open_il_12m`, `open_il_24m`, `mths_since_rcnt_il`, `total_bal_il`, `il_util`, `open_rv_12m`, `open_rv_24m`, `max_bal_bc`, `all_util`, `total_rev_hi_lim`, `inq_fi`, `total_cu_tl`, `inq_last_12m`, `acc_open_past_24mths`, `avg_cur_bal`, `bc_open_to_buy`, `bc_util`, `chargeoff_within_12_mths`, `delinq_amnt`, `mo_sin_old_il_acct`, `mo_sin_old_rev_tl_op`, `mo_sin_rcnt_rev_tl_op`, `mo_sin_rcnt_tl`, `mort_acc`, `mths_since_recent_bc`, `mths_since_recent_bc_dlq`, `mths_since_recent_inq`, `mths_since_recent_revol_delinq`, `num_accts_ever_120_pd`, `num_actv_bc_tl`, `num_actv_rev_tl`, `num_bc_sats`, `num_bc_tl`, `num_il_tl`, `num_op_rev_tl`, `num_rev_accts`, `num_rev_tl_bal_gt_0`, `num_sats`, `num_tl_120dpd_2m`, `num_tl_30dpd`, `num_tl_90g_dpd_24m`, `num_tl_op_past_12m`, `pct_tl_nvr_dlq`, `percent_bc_gt_75`, `tot_hi_cred_lim`, `total_bal_ex_mort`, `total_bc_limit`, `total_il_high_credit_limit`, `mths_since_last_major_derog`.

При загрузке пустые значения (`NaN`) в JSON не включаются.

### Адаптация заданий под датасет

- **«Выручка» и «чистая прибыль» заёмщика.** Датасет описывает физических лиц. Вместо выручки используется `annual_inc`. Прибыль для кредитора считается по займу: `total_rec_int + total_rec_late_fee + recoveries − collection_recovery_fee` минус потери основного долга по списанным займам. Формулу нужно закрепить в представлении на этапе 3.
- **Триггер «кредит ≤ 50% выручки».** Читается как «`loan_amnt` ≤ 50% `annual_inc`». Часть исторических займов это правило нарушает, поэтому при начальной загрузке триггер отключается (`ALTER TABLE … DISABLE TRIGGER`), а при новых вставках работает.
- **Обеспечение и поручители.** Таблицы заполняются синтетикой. В `queries.sql` запрос «залог < 80% кредита» работает по ним.
