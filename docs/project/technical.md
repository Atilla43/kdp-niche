# Техническая Документация

## Технический Стек

| Компонент | Решение | Стоимость | Примечание |
|-----------|---------|-----------|------------|
| **Frontend** | Next.js 14 + TailwindCSS + shadcn/ui | $0 | App Router |
| **Backend** | Next.js API Routes | $0 | Serverless |
| **База данных** | Supabase (PostgreSQL) | $0 | Free tier + ping |
| **Авторизация** | Supabase Auth | $0 | Email + OAuth |
| **Хостинг** | Vercel | $0 | Free tier |
| **Платежи** | Stripe | ~5% | С учётом налогов |
| **Email** | Amazon SES / Resend | $0-5/мес | |
| **Данные BSR** | Keepa API | €19/мес | |
| **Search Volume** | Keywords Everywhere API | $21/год | 100K credits |
| **Cron Jobs** | GitHub Actions | $0 | Ping Supabase |

## Архитектура

```
┌─────────────────────────────────────────────────┐
│                   Frontend                       │
│       Next.js 14 + TailwindCSS + shadcn/ui      │
│                 (Vercel)                        │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│              API Routes                          │
│           Next.js Backend                        │
└──┬──────────┬──────────┬──────────┬─────────────┘
   │          │          │          │
┌──▼──┐  ┌────▼────┐ ┌───▼───┐ ┌────▼────┐
│Supa-│  │ Keepa   │ │Keywords│ │ Stripe  │
│base │  │  API    │ │Every-  │ │         │
│     │  │ (BSR)   │ │where   │ │(Payment)│
└──┬──┘  └────┬────┘ └───┬───┘ └─────────┘
   │          │          │
   │    ┌─────▼──────────▼─────┐
   │    │      CACHE           │
   │    │   (Supabase DB)      │
   │    │ - BSR data (24h TTL) │
   │    │ - Keywords (7d TTL)  │
   └────►─────────────────────┘
```

## Страницы Приложения

| Путь | Описание | Доступ |
|------|----------|--------|
| `/` | Landing page | Публичный |
| `/login` | Вход | Публичный |
| `/signup` | Регистрация | Публичный |
| `/dashboard` | Главный экран | Auth |
| `/search` | Поиск ниш | Auth |
| `/saved` | Сохранённые ниши | Auth |
| `/settings` | Настройки аккаунта | Auth |
| `/pricing` | Тарифы | Публичный |

## API Routes

| Endpoint | Метод | Описание |
|----------|-------|----------|
| `/api/auth/*` | — | Supabase Auth |
| `/api/search` | POST | Поиск ниш по категории/ключевому слову |
| `/api/niche/[id]` | GET | Детали ниши |
| `/api/saved` | GET/POST/DELETE | Сохранённые ниши |
| `/api/usage` | GET | Статистика использования |
| `/api/webhooks/stripe` | POST | Stripe webhooks |

## База Данных (Supabase)

### Таблица: profiles
```sql
create table public.profiles (
  id uuid references auth.users primary key,
  email text,
  plan text default 'free', -- 'free', 'pro'
  stripe_customer_id text,
  searches_this_month int default 0,
  searches_reset_at timestamp,
  created_at timestamp default now()
);
```

### Таблица: saved_niches
```sql
create table public.saved_niches (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles,
  category_id text,
  category_name text,
  niche_score int,
  avg_bsr int,
  avg_reviews int,
  avg_search_volume int,
  competition_level text, -- 'low', 'medium', 'high'
  data jsonb, -- полный ответ API
  created_at timestamp default now()
);
```

### Таблица: search_logs
```sql
create table public.search_logs (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles,
  query text,
  results_count int,
  created_at timestamp default now()
);
```

### Таблица: cache (для API ответов)
```sql
create table public.cache (
  key text primary key,
  value jsonb,
  expires_at timestamp,
  created_at timestamp default now()
);

-- Индекс для автоочистки
create index idx_cache_expires on public.cache(expires_at);
```

## Стратегия Кэширования

| Данные | TTL | Причина |
|--------|-----|---------|
| BSR книги | 24 часа | Меняется ежедневно |
| Keywords volume | 7 дней | Меняется редко |
| Category bestsellers | 6 часов | Важно для трендов |
| Niche Score | 24 часа | Пересчитывается |

**Экономия:** Кэширование снижает API-запросы на 70-80%

## Niche Score Алгоритм

```typescript
interface NicheScore {
  total: number;        // 0-100
  demand: number;       // 0-100
  competition: number;  // 0-100
  opportunity: number;  // 0-100
}

function calculateNicheScore(data: NicheData): NicheScore {
  const demand = calculateDemandScore(data.avgBSR, data.searchVolume);
  const competition = calculateCompetitionScore(data.avgReviews, data.topBookAge);
  const opportunity = calculateOpportunityScore(data);

  return {
    total: Math.round(demand * 0.4 + competition * 0.35 + opportunity * 0.25),
    demand,
    competition,
    opportunity
  };
}
```

### Demand Score (0-100)
| BSR среднего топ-10 | Score |
|---------------------|-------|
| < 10,000 | 100 |
| 10,000-50,000 | 80 |
| 50,000-100,000 | 60 |
| 100,000-300,000 | 40 |
| > 300,000 | 20 |

### Competition Score (0-100)
| Среднее отзывов топ-10 | Score |
|------------------------|-------|
| < 50 | 100 |
| 50-200 | 80 |
| 200-500 | 60 |
| 500-1000 | 40 |
| > 1000 | 20 |

### Opportunity Score (0-100)
- Книги с < 50 отзывов в топ-20: +30
- Новейшая книга в топ-10 < 6 мес: +20
- Высокий разброс BSR: +20
- Цена топ-книг > $15: +15
- Растущий тренд: +15

## Ограничения Free Tier

### Supabase
| Лимит | Значение | Решение |
|-------|----------|---------|
| Storage | 500MB | Достаточно для MVP |
| Bandwidth | 10GB/мес | Мониторинг |
| **Пауза после 7 дней** | Критично! | GitHub Actions ping |

### Vercel
| Лимит | Значение | Решение |
|-------|----------|---------|
| Function timeout | 10 сек | Кэширование |
| Bandwidth | 100GB/мес | Достаточно |
| Invocations | 150K/мес | Достаточно |

### Keepa API
| Лимит | Значение | Решение |
|-------|----------|---------|
| Запросов/мин | 60 | Rate limiting |
| Токены | Ограничены планом | Кэширование |

## GitHub Actions: Supabase Ping

```yaml
# .github/workflows/ping-supabase.yml
name: Ping Supabase

on:
  schedule:
    - cron: '0 0 */6 * *'  # Каждые 6 дней

jobs:
  ping:
    runs-on: ubuntu-latest
    steps:
      - name: Ping Supabase
        run: |
          curl -X GET "${{ secrets.SUPABASE_URL }}/rest/v1/profiles?limit=1" \
            -H "apikey: ${{ secrets.SUPABASE_ANON_KEY }}"
```

## Переменные Окружения

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Keepa
KEEPA_API_KEY=

# Keywords Everywhere
KEYWORDS_EVERYWHERE_API_KEY=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# App
NEXT_PUBLIC_APP_URL=
```

## Источники

- [Supabase Documentation](https://supabase.com/docs)
- [Vercel Limits](https://vercel.com/docs/limits)
- [Keepa API Documentation](https://keepaapi.readthedocs.io/)
- [Keywords Everywhere API](https://keywordseverywhere.com/frequently-asked-questions.html)
- [Stripe Subscriptions](https://stripe.com/docs/billing/subscriptions)
