# 🔍 Queries, Upsert & Regex

[🏠 Home](../README.md) > [🐘 PostgreSQL](./README.md)

## 🎯 Regex & Pattern Matching

PostgreSQL uses POSIX regular expressions via `~` operators alongside standard `LIKE`/`ILIKE`.

| Operator | Meaning | Example |
| :--- | :--- | :--- |
| `LIKE` | Case-sensitive wildcard (`%`, `_`) | `email LIKE '%@gmail.com'` |
| `ILIKE` | Case-insensitive wildcard | `name ILIKE 'john%'` |
| `~` | Regex match (case-sensitive) | `code ~ '^[A-Z]{3}-\d+$'` |
| `~*` | Regex match (case-insensitive) | `email ~* '^[a-z0-9._%+-]+@'` |
| `!~` | Does NOT match regex (case-sensitive) | `username !~ '[^a-zA-Z0-9_]'` |
| `!~*` | Does NOT match regex (case-insensitive) | `tag !~* '^test'` |

### Regex Extraction & Replacement

```sql
-- Extract first matching substring
SELECT substring('Order #INV-2026-99' FROM 'INV-\d+-\d+'); -- Returns: INV-2026-99

-- Replace matching patterns ('g' flag = global / replace all)
SELECT regexp_replace('0812-3456-7890', '[^0-9]', '', 'g'); -- Returns: 081234567890
```

## 🔄 Upsert (`ON CONFLICT`) & `RETURNING`

```sql
-- Insert and return generated fields immediately
INSERT INTO users(org_id, id, role)
VALUES ('550e8400-e29b-41d4-a716-446655440000', 'dev@example.com', 'admin')
RETURNING id, created_at;

-- Upsert: Insert or update if unique constraint collides
INSERT INTO users (org_id, email, role)
VALUES ('550e8400-e29b-41d4-a716-446655440000', 'dev@example.com', 'admin')
ON CONFLICT (org_id, email)
DO UPDATE SET 
    role = EXCLUDED.role,
    status = 'active'
RETURNING *;
```

## 📦 `JSONB` Quick Operations

```sql
-- Query nested key as text (->>)
SELECT * FROM events WHERE payload->>'type' = 'PAYMENT_SUCCESS';

-- Containment check (@>) — uses GIN index if available
SELECT * FROM events WHERE payload @> '{"status": "paid", "currency": "IDR"}';

-- Update a specific key inside JSONB without overwriting the whole column
UPDATE users 
SET profile = jsonb_set(profile, '{theme}', '"dark"') 
WHERE id = '550e8400-e29b-41d4-a716-446655440000';
```
