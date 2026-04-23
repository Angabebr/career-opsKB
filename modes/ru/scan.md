# Режим: scan — Сканер вакансий (Российский рынок)

Сканирует российские площадки, фильтрует по профилю, добавляет новые вакансии в pipeline.

## Источники (в порядке приоритета)

1. **hh.ru API** — прямой API, 100+ вакансий за запрос, всегда актуально
2. **WebSearch** — Habr Career, SuperJob, GeekJob и другие (10 результатов на запрос)
3. **Прямые страницы компаний** — стажировки Яндекс, Т-Банк, Контур, EPAM и др.

---

## Шаг 1 — Читаем конфигурацию

Прочитать:
- `config/profile.yml` → целевые роли, зарплата, локация, релокация
- `portals.yml` → title_filter (positive/negative), search_queries, tracked_companies
- `data/scan-history.tsv` → уже виденные URL (дедупликация)
- `data/applications.md` → уже оценённые вакансии
- `data/pipeline.md` → уже в очереди

---

## Шаг 2 — hh.ru API (основной источник)

Выполнить параллельно следующие WebFetch запросы к hh.ru API:

### Запрос 1 — Удалённые QA по всей России
```
https://api.hh.ru/vacancies?text=QA+тестировщик+junior&area=113&per_page=100&schedule=remote&order_by=publication_time
```

### Запрос 2 — Все QA по всей России (без фильтра по формату)
```
https://api.hh.ru/vacancies?text=тестировщик+junior&area=113&per_page=100&experience=noExperience&order_by=publication_time
```

### Запрос 3 — Junior QA удалённо
```
https://api.hh.ru/vacancies?text=QA+engineer+junior&area=113&per_page=100&schedule=remote&order_by=publication_time
```

### Запрос 4 — Стажёр / trainee QA
```
https://api.hh.ru/vacancies?text=стажёр+тестировщик&area=113&per_page=100&order_by=publication_time
```

### Запрос 5 — Ручное тестирование удалённо
```
https://api.hh.ru/vacancies?text=ручное+тестирование+junior&area=113&per_page=100&schedule=remote&order_by=publication_time
```

### Запрос 6 — Саратов (любой формат)
```
https://api.hh.ru/vacancies?text=тестировщик+QA&area=1399&per_page=100&order_by=publication_time
```

**Парсинг ответа hh.ru API:**
```json
{
  "items": [
    {
      "id": "...",
      "name": "Junior QA Engineer",
      "url": "https://api.hh.ru/vacancies/123",
      "alternate_url": "https://hh.ru/vacancy/123",   ← использовать этот URL
      "employer": { "name": "Компания" },
      "salary": { "from": 70000, "to": 90000, "currency": "RUR", "gross": false },
      "area": { "name": "Москва" },
      "schedule": { "name": "Удалённая работа" },
      "published_at": "2026-04-23T...",
      "snippet": { "requirement": "...", "responsibility": "..." }
    }
  ]
}
```

Использовать `alternate_url` как URL вакансии. Из каждого item извлечь:
- title: `name`
- url: `alternate_url`
- company: `employer.name`
- salary: `salary.from`–`salary.to` `salary.currency` (пометить gross/net)
- location: `area.name`
- schedule: `schedule.name`
- published: `published_at` (отсеивать старше 30 дней)
- snippet: `snippet.requirement` + `snippet.responsibility`

---

## Шаг 3 — WebSearch (другие площадки)

Выполнить параллельно все `search_queries` из `portals.yml` с `enabled: true`.

Дополнительно выполнить эти запросы:

```
site:career.habr.com "тестировщик" OR "QA" junior 2026
site:career.habr.com "manual QA" OR "ручное тестирование" удалённо
site:geekjob.ru "тестировщик" OR "QA" junior
site:superjob.ru "тестировщик" OR "QA" junior удалённо
"тестировщик" junior Postman SQL удалённо -senior -lead
"QA engineer" junior remote Russia -senior -lead site:linkedin.com
```

---

## Шаг 4 — Прямые страницы компаний (стажировки)

WebFetch или WebSearch для топ-стажировок:

| Компания | Метод | URL / запрос |
|----------|-------|--------------|
| EPAM Саратов | WebSearch | `EPAM Systems тестировщик Саратов stажировка 2026` |
| Яндекс стажировка | WebFetch | `https://yandex.ru/jobs/vacancies/?department=testing` |
| Т-Банк Т-Старт | WebSearch | `Т-Банк Т-Старт QA тестировщик стажировка 2026` |
| Контур | WebSearch | `Контур стажёр тестировщик Екатеринбург 2026` |
| Авито | WebSearch | `Авито стажёр QA разработчик 2026` |
| Сбер | WebSearch | `Сбер тестировщик junior стажировка hh.ru` |

---

## Шаг 5 — Фильтрация по профилю

### Title filter (из `portals.yml`)
- **positive** — хотя бы одно слово должно быть в названии (без учёта регистра)
- **negative** — ни одного слова из списка. Добавить к стандартным: "1C", "1С", "SAP", "embedded", "mobile", "iOS", "Android"

### Salary filter (из `config/profile.yml`)
- Из hh.ru API: если `salary.from` указан и `salary.to < 50000` (net) → пометить как low_pay, не отбрасывать
- Если зарплата не указана → оставить (норма для джун-вакансий)

### Location filter
- Удалённая работа → ✅
- Саратов (офис/гибрид) → ✅
- Оплачиваемый переезд (relocation_package упомянут) → ✅
- Офис другой город без переезда → пометить, но оставить в pipeline для ручной проверки

### Freshness filter
- Из hh.ru API: отсеивать вакансии старше 30 дней (`published_at`)
- Из WebSearch: отсеивать если в сниппете год < 2025

---

## Шаг 6 — Дедупликация

Для каждой вакансии прошедшей фильтр:
1. Проверить `data/scan-history.tsv` по точному URL → пропустить если уже видели
2. Проверить `data/applications.md` по company + role → пропустить если уже оценивали
3. Проверить `data/pipeline.md` по URL → пропустить если уже в очереди

---

## Шаг 7 — Сортировка и приоритизация

Отсортировать новые вакансии по приоритету:

| Приоритет | Критерий |
|-----------|----------|
| ⭐ Высокий | Удалённо + junior + зарплата в диапазоне + свежая (< 7 дней) |
| 🔵 Средний | Гибрид/офис Саратов или удалённо без зарплаты |
| ⚪ Низкий | Другой город без пакета переезда или зарплата ниже минимума |

---

## Шаг 8 — Запись результатов

### `data/scan-history.tsv`
Добавить каждую просмотренную URL:
```
{url}\t{date}\t{source}\t{title}\t{company}\t{status}
```
status: `added` | `skipped_title` | `skipped_dup` | `skipped_old` | `skipped_location`

### `data/pipeline.md`
Добавить новые вакансии в секцию "Pending":
```
- [ ] {url} | {company} | {title} | {salary} | {location} | {schedule}
```

---

## Шаг 9 — Итоговый отчёт

```
═══════════════════════════════════════
  Скан вакансий QA — {YYYY-MM-DD}
═══════════════════════════════════════

Источники:
  hh.ru API:        {N} найдено → {N} новых
  WebSearch:        {N} найдено → {N} новых
  Прямые страницы:  {N} найдено → {N} новых

Фильтрация:
  Прошли фильтр:    {N}
  Отсеяны (title):  {N}
  Дубликаты:        {N}
  Устаревшие:       {N}

Добавлено в pipeline: {N}

──────────────────────────────────────
⭐ ТОП КАНДИДАТЫ (приоритет высокий)
──────────────────────────────────────
{company} — {title}
  {url}
  📍 {location} · 💰 {salary} · 📅 {days} дн. назад
  💡 {короткий комментарий по fit}

[... остальные ...]

──────────────────────────────────────
🔵 ОСТАЛЬНЫЕ НОВЫЕ ВАКАНСИИ
──────────────────────────────────────
[список]

──────────────────────────────────────
→ Запусти /career-ops pipeline чтобы оценить все новые вакансии.
→ Начни с ⭐ топ кандидатов.
═══════════════════════════════════════
```
