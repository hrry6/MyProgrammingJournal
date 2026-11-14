# Schema, UUIDs & Constraints

[🏠 Home](../README.md) > [🐘 PostgreSQL](./README.md)

## Generating UUID

PostgreSQL 13+ includes `gen_random_uuid()` natively (UUID v4) without needing external extensions.

```sql
-- Built-in (Recommended for Postgres 13+)
SELECT gen_random_uuid();

-- Legacy extension (only needed for UUID v1/v3/v5 or Postgres < 13)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();
```

## Reference & Constraints Cheatsheet

```sql
CREATE TABLE organizations(
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMPZ NOT NULL DEFAULT NOW()
)

-- Inline reference constraint(simple but complicated to change)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  email VARCHAR(255) NOT NULL,
  role VARCHAR(20) NOT NULL DEFAULT 'member',
  age INT CHECK (age >= 13),
  status VARCHAR(20) CHECK (status IN ('active', 'suspended', 'pending')),
  
  -- Composite unique constraint
  CONSTRAINT uq_org_email UNIQUE (org_id, email)
);

-- Expplisit reference constraint (bets practice)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID,
  email VARCHAR(255) NOT NULL,
  role VARCHAR(20) NOT NULL DEFAULT 'member',
  age INT CHECK (age >= 13),
  status VARCHAR(20) CHECK (status IN ('active', 'suspended', 'pending')),

  -- Composite unique constraint
  CONSTRAINT uq_org_email UNIQUE (org_id, email)
  CONSTRAINT fk_users_organizations FOREIGN KEY(id) REFERENCES organizations(id) ON DELETE CASCADE
);
```

## Foreign Key `ON DELETE` and `ON UPDATE` Actions

| Action | Behavior when parent is deleted | Behavior when parent is updated |
| :--- | :--- | :--- |
| `CASCADE` | Automatically delete all referencing child rows | Automatically update the id from all referencing child rows to follow the parent's |
| `SET NULL` | Set to null all referencing child rows | Set to null all referencing child rows |
| `SET DEFAULT` | Change child's foreign id to default value | Change child's foreign id to default value |
| `RESTRICT` | Blocks delete immidiately if child rows exists | Blocks update immidiately if child rows exists |
| `NO ACTION` | Simmilar to `RESTRICT`, but checked at the end of transations | Simmilar to `RESTRICT`, but checked at the end of transations |

## Altering Constraints on Existing Tables

```sql
--- Add a check constraint
ALTER TABLE users ADD CONSTRAINTS chk_valid_role CHECK (role IN ('MEMBER', 'ADMIN', 'GUEST'))

--- Drop a constraint
ALTER TABLE users DROP CONSTRAINTS chk_valid_role;

-- Add foreign key to 
ALTER TABLE users 
ADD CONSTRAINT fk_users_org FOREIGN KEY (org_id) REFERENCES organizations(id) ON DELETE SET NULL;
```
