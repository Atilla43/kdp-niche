# KDP Niche Finder

> SaaS-платформа для автоматического поиска прибыльных ниш на Amazon KDP

## Что это?

Онлайн-инструмент, который анализирует Amazon KDP и помогает авторам находить ниши с высоким спросом и низкой конкуренцией. Вместо ручного анализа десятков метрик — одна понятная цифра: **Niche Score** (0-100).

```
"gratitude journal" → Анализ → Niche Score: 72/100 — ХОРОШАЯ НИША
```

## Для кого?

| Аудитория | Проблема | Решение |
|-----------|----------|---------|
| Начинающие KDP-авторы | Не знают как искать ниши | Автоматический поиск + понятный Score |
| Опытные издатели | Публикуют много книг, нужна автоматизация | Batch-анализ, алерты |
| Дизайнеры low-content | Ищут trending темы | Топ ниш недели, тренды |

## Основные возможности

### Free (5 поисков/мес)
- Поиск ниш по ключевому слову
- Niche Score (Demand + Competition + Opportunity)
- BSR, отзывы, цены топ-10 книг

### Pro ($19/мес)
- Безлимитный поиск
- Search Volume ключевых слов
- Исторические тренды
- Сохранение и экспорт (CSV)
- Email-дайджест топ ниш недели

## Технологии

| Компонент | Технология |
|-----------|------------|
| Frontend | Next.js 14 + TailwindCSS + shadcn/ui |
| Backend | Next.js API Routes (Serverless) |
| База данных | Supabase (PostgreSQL) |
| Авторизация | Supabase Auth |
| Хостинг | Vercel |
| Данные BSR | Keepa API |
| Search Volume | Keywords Everywhere API |
| Платежи | Stripe |

## Niche Score

```
Niche Score = Demand (40%) + Competition (35%) + Opportunity (25%)
```

| Компонент | Что измеряет | Источник |
|-----------|--------------|----------|
| **Demand** | Спрос на нишу | BSR + Search Volume |
| **Competition** | Сложность входа | Количество отзывов, рейтинги |
| **Opportunity** | Окно для входа | Новые книги в топе, разброс BSR |

## Документация

### Проект
| Документ | Описание |
|----------|----------|
| [docs/project/overview.md](docs/project/overview.md) | Общее описание проекта |
| [docs/project/technical.md](docs/project/technical.md) | Технический стек, база данных |
| [docs/project/audience.md](docs/project/audience.md) | Целевая аудитория, боли |
| [docs/project/budget.md](docs/project/budget.md) | Бюджет $500, финансы |
| [docs/project/caching.md](docs/project/caching.md) | Стратегия кэширования (80% экономии) |
| [docs/project/marketing.md](docs/project/marketing.md) | Маркетинговая стратегия |
| [docs/project/competitors.md](docs/project/competitors.md) | Анализ конкурентов |
| [docs/project/risks.md](docs/project/risks.md) | Риски и митигация |

### Архитектура
| Документ | Описание |
|----------|----------|
| [docs/technical/architecture/overview.md](docs/technical/architecture/overview.md) | Общая архитектура системы |
| [docs/technical/architecture/user-journey.md](docs/technical/architecture/user-journey.md) | Путь пользователя |
| [docs/technical/architecture/data-flow.md](docs/technical/architecture/data-flow.md) | Потоки данных |

### Конкуренты
| Документ | Описание |
|----------|----------|
| [docs/competitors/overview.md](docs/competitors/overview.md) | Сводный анализ |
| [docs/competitors/publisher-rocket.md](docs/competitors/publisher-rocket.md) | Publisher Rocket ($120K/мес) |
| [docs/competitors/book-bolt.md](docs/competitors/book-bolt.md) | Book Bolt |
| [docs/competitors/kdspy.md](docs/competitors/kdspy.md) | KDSPY (65K+ клиентов) |

## Бюджет

**Общий бюджет: $500**

| Статья | Сумма |
|--------|-------|
| Домен | $12 |
| Keepa API (6 мес) | $126 |
| Keywords Everywhere | $21 |
| Facebook Ads | $100 |
| Резерв | $141 |
| Vercel + Supabase | $0 |
| **Разработка** | **$0** (Claude Code) |

## Метрики (цель 6 мес)

| Метрика | Цель |
|---------|------|
| Регистрации | 2,000 |
| Конверсия Free→Pro | 4% |
| Платящих | 80 |
| MRR | $1,520 |

---

*Статус: В разработке*
