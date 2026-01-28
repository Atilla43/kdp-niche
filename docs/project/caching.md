# Стратегия Кэширования

> Как экономить 80% на API-запросах

## Зачем Кэширование

### Проблема: API стоят денег

| API | Стоимость | Лимит |
|-----|-----------|-------|
| **Keepa** | €19/мес | Токены ограничены планом |
| **Keywords Everywhere** | $21/год | 100,000 credits |

### Без кэша — катастрофа

```
100 пользователей × 10 поисков/день = 1,000 запросов/день

Keepa токены на €19/мес ≈ 10,000-20,000 запросов
→ Закончатся за 10-20 дней

Keywords Everywhere: 100K credits/год
→ Закончатся за 100 дней
```

### С кэшом — экономия 80%

```
1,000 запросов/день
- 80% уже в кэше (популярные ниши повторяются)
= 200 уникальных запросов/день

Keepa токены хватит на 50-100 дней
Keywords Everywhere хватит на 500 дней
```

---

## Стратегия TTL (Time To Live)

### Какие данные как долго хранить

| Данные | TTL | Почему |
|--------|-----|--------|
| **BSR книги** | 24 часа | Меняется ежедневно, но тренд важнее точности |
| **Search volume** | 7 дней | Меняется редко, месячная метрика |
| **Category bestsellers** | 6 часов | Важно для трендов, но не критично |
| **Niche Score** | 24 часа | Пересчитывается из BSR |
| **ASIN metadata** | 7 дней | Название, автор — редко меняются |
| **"Not found" ответы** | 1 час | Чтобы не долбить API несуществующим |

### Логика выбора TTL

```
Чем чаще меняются данные → тем короче TTL
Чем дороже API-запрос → тем длиннее TTL
Чем важнее свежесть → тем короче TTL
```

---

## Расчёт Экономии

### Сценарий: 100 активных пользователей

| Метрика | Без кэша | С кэшом (80% hit) | Экономия |
|---------|----------|-------------------|----------|
| Запросов/день | 1,000 | 200 | **80%** |
| Запросов/месяц | 30,000 | 6,000 | **80%** |
| Keepa токенов/мес | 30,000 | 6,000 | **€15/мес** |
| Keywords credits/мес | 30,000 | 6,000 | **$17.50/мес** |

### Формула Cache Hit Rate

```
Cache Hit Rate = Запросов из кэша / Всего запросов × 100%

Цель: ≥70%
Хороший: ≥80%
Отличный: ≥90%
```

### Почему 80% реалистично

- **Популярные ниши повторяются** — "gratitude journal" ищут все
- **Один bestseller во многих нишах** — одна книга = один запрос
- **Пользователи ищут похожее** — "planner 2025", "planner 2026"

---

## Реализация в Supabase

### Таблица Cache

```sql
-- Создание таблицы
create table public.cache (
  key text primary key,           -- Уникальный ключ запроса
  value jsonb,                    -- JSON с данными
  expires_at timestamp,           -- Когда устареет
  created_at timestamp default now(),
  hit_count int default 0         -- Сколько раз использовали (для аналитики)
);

-- Индекс для быстрой очистки устаревших
create index idx_cache_expires on public.cache(expires_at);

-- Индекс для поиска по ключу (уже есть, т.к. primary key)

-- RLS (Row Level Security) — кэш общий для всех
alter table public.cache enable row level security;

create policy "Cache is readable by all authenticated users"
  on public.cache for select
  to authenticated
  using (true);

create policy "Cache is writable by service role only"
  on public.cache for insert
  to service_role
  with check (true);
```

### Формат Ключей

```typescript
// Примеры ключей
const keys = {
  // BSR книги
  bsr: `bsr:${asin}`,                    // bsr:B08XYZ123

  // Keywords
  keyword: `kw:${keyword}:${country}`,   // kw:gratitude journal:us

  // Category bestsellers
  category: `cat:${categoryId}`,         // cat:digital-ebook-journals

  // Niche score
  niche: `niche:${categoryId}`,          // niche:digital-ebook-journals
};
```

---

## Код Кэширования

### Основная Функция

```typescript
// lib/cache.ts

import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY! // Важно: service role для записи
);

interface CacheOptions {
  ttlSeconds: number;
  staleWhileRevalidate?: boolean; // Вернуть старые данные пока обновляем
}

export async function getWithCache<T>(
  key: string,
  fetchFn: () => Promise<T>,
  options: CacheOptions
): Promise<T> {
  const { ttlSeconds, staleWhileRevalidate = false } = options;

  // 1. Проверить кэш
  const { data: cached, error } = await supabase
    .from('cache')
    .select('value, expires_at')
    .eq('key', key)
    .single();

  const now = new Date();
  const isValid = cached && new Date(cached.expires_at) > now;

  // 2. Если валидный — вернуть из кэша
  if (isValid) {
    // Увеличить счётчик попаданий (async, не ждём)
    supabase
      .from('cache')
      .update({ hit_count: supabase.rpc('increment_hit', { row_key: key }) })
      .eq('key', key);

    return cached.value as T;
  }

  // 3. Если stale-while-revalidate и есть старые данные
  if (staleWhileRevalidate && cached) {
    // Вернуть старые данные, обновить в фоне
    fetchAndCache(key, fetchFn, ttlSeconds);
    return cached.value as T;
  }

  // 4. Запросить свежие данные
  return fetchAndCache(key, fetchFn, ttlSeconds);
}

async function fetchAndCache<T>(
  key: string,
  fetchFn: () => Promise<T>,
  ttlSeconds: number
): Promise<T> {
  try {
    const fresh = await fetchFn();

    // Сохранить в кэш
    await supabase.from('cache').upsert({
      key,
      value: fresh,
      expires_at: new Date(Date.now() + ttlSeconds * 1000).toISOString(),
      hit_count: 0
    });

    return fresh;
  } catch (error) {
    // Если API упал — попробовать вернуть stale данные
    const { data: stale } = await supabase
      .from('cache')
      .select('value')
      .eq('key', key)
      .single();

    if (stale) {
      console.warn(`API failed, returning stale cache for: ${key}`);
      return stale.value as T;
    }

    throw error;
  }
}
```

### Константы TTL

```typescript
// lib/cache-config.ts

export const CACHE_TTL = {
  BSR: 24 * 60 * 60,           // 24 часа в секундах
  KEYWORDS: 7 * 24 * 60 * 60,  // 7 дней
  BESTSELLERS: 6 * 60 * 60,    // 6 часов
  NICHE_SCORE: 24 * 60 * 60,   // 24 часа
  METADATA: 7 * 24 * 60 * 60,  // 7 дней
  NOT_FOUND: 60 * 60,          // 1 час
} as const;
```

### Использование

```typescript
// api/search.ts

import { getWithCache, CACHE_TTL } from '@/lib/cache';
import { fetchKeepaData } from '@/lib/keepa';
import { fetchKeywordVolume } from '@/lib/keywords-everywhere';

export async function searchNiche(query: string) {
  // Получить BSR данные (с кэшом)
  const bsrData = await getWithCache(
    `bsr:${query}`,
    () => fetchKeepaData(query),
    { ttlSeconds: CACHE_TTL.BSR }
  );

  // Получить search volume (с кэшом)
  const keywords = await getWithCache(
    `kw:${query}:us`,
    () => fetchKeywordVolume(query, 'us'),
    { ttlSeconds: CACHE_TTL.KEYWORDS }
  );

  // Рассчитать Niche Score
  const nicheScore = calculateNicheScore(bsrData, keywords);

  return { bsrData, keywords, nicheScore };
}
```

---

## Edge Cases

### 1. API Недоступен

```typescript
// Уже реализовано в fetchAndCache()
// Возвращаем stale данные + логируем

// На фронте показываем:
{
  data: staleData,
  warning: "Данные могут быть устаревшими. Обновим позже."
}
```

### 2. Несуществующий ASIN/Keyword

```typescript
// Кэшируем "not found" чтобы не долбить API
const result = await fetchKeepaData(asin);

if (!result) {
  await supabase.from('cache').upsert({
    key: `bsr:${asin}`,
    value: { notFound: true },
    expires_at: new Date(Date.now() + CACHE_TTL.NOT_FOUND * 1000)
  });
}
```

### 3. Слишком Много Уникальных Запросов

```typescript
// Rate limiting на уровне пользователя
const userSearches = await supabase
  .from('search_logs')
  .select('count')
  .eq('user_id', userId)
  .gte('created_at', startOfDay());

if (userSearches.count >= USER_DAILY_LIMIT) {
  throw new Error('Daily search limit reached');
}
```

### 4. Кэш Переполнен (>500MB на free tier)

```sql
-- Мониторинг размера
SELECT pg_size_pretty(pg_total_relation_size('cache'));

-- Если близко к лимиту — удалить старые неиспользуемые
DELETE FROM cache
WHERE hit_count = 0
  AND created_at < NOW() - INTERVAL '7 days';
```

---

## Очистка Кэша

### GitHub Actions Cron

```yaml
# .github/workflows/cleanup-cache.yml
name: Cleanup Expired Cache

on:
  schedule:
    - cron: '0 */6 * * *'  # Каждые 6 часов

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Delete expired cache entries
        run: |
          curl -X POST "${{ secrets.SUPABASE_URL }}/rest/v1/rpc/cleanup_expired_cache" \
            -H "apikey: ${{ secrets.SUPABASE_SERVICE_ROLE_KEY }}" \
            -H "Authorization: Bearer ${{ secrets.SUPABASE_SERVICE_ROLE_KEY }}"
```

### SQL Функция

```sql
-- Supabase SQL Editor
create or replace function cleanup_expired_cache()
returns void as $$
begin
  delete from cache where expires_at < now();
end;
$$ language plpgsql security definer;
```

---

## Мониторинг

### Метрики для отслеживания

| Метрика | Как измерить | Цель |
|---------|--------------|------|
| **Cache Hit Rate** | hits / total requests | ≥70% |
| **API расходы/день** | Логи Keepa/Keywords | <200 запросов |
| **Размер таблицы** | pg_total_relation_size | <100MB |
| **Avg response time** | cached vs fresh | cached <50ms |
| **Stale returns** | Логи | <1% запросов |

### Dashboard Query

```sql
-- Статистика кэша
SELECT
  COUNT(*) as total_entries,
  SUM(hit_count) as total_hits,
  AVG(hit_count) as avg_hits,
  pg_size_pretty(pg_total_relation_size('cache')) as size,
  COUNT(*) FILTER (WHERE expires_at > NOW()) as valid_entries,
  COUNT(*) FILTER (WHERE expires_at <= NOW()) as expired_entries
FROM cache;
```

---

## Источники

- [Supabase PostgreSQL Functions](https://supabase.com/docs/guides/database/functions)
- [Cache-Aside Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [Stale-While-Revalidate](https://web.dev/stale-while-revalidate/)
