# Part 39: Database Migrations ใน CI/CD

## บทนำ

Database Migrations เป็นหนึ่งในส่วนที่ท้าทายที่สุดในกระบวนการ CI/CD การเปลี่ยนแปลง schema ของฐานข้อมูลต้องทำอย่างระมัดระวัง โดยเฉพาะใน production environment ที่ต้องการ zero downtime ในบทนี้เราจะเรียนรู้การใช้ Flyway, Liquibase, Alembic, และ strategies ต่างๆ สำหรับ database migrations ใน CI/CD pipeline

## วัตถุประสงค์การเรียนรู้

- เข้าใจหลักการ database migration
- ใช้ Flyway สำหรับ Java/JVM projects
- ใช้ Liquibase สำหรับ multi-database support
- ทำ zero-downtime migrations
- Rollback migrations อย่างปลอดภัย
- Test migrations ใน CI pipeline

---

## 39.1 หลักการ Database Migration

### Migration Lifecycle

```
Development ──► Version Control ──► CI Pipeline ──► Staging ──► Production
     │                │                  │             │            │
  Write SQL         Commit            Validate       Test        Deploy
  migration         migration         schema         data        migration
```

### Best Practices

1. **Never edit migrations** - แก้ไข migration ที่ deploy ไปแล้วไม่ได้
2. **Backwards compatible** - migration ใหม่ควร work กับ code เวอร์ชันเก่า
3. **Test rollbacks** - ต้องมี rollback script เสมอ
4. **One migration per change** - แต่ละ migration ควรทำสิ่งเดียว
5. **Include data migrations** - เมื่อต้อง transform ข้อมูล

---

## 39.2 Flyway

### การติดตั้ง Flyway

```bash
# Maven
# pom.xml
# <plugin>
#   <groupId>org.flywaydb</groupId>
#   <artifactId>flyway-maven-plugin</artifactId>
#   <version>9.22.3</version>
# </plugin>

# Gradle
# build.gradle
# plugins {
#   id 'org.flywaydb.flyway' version '9.22.3'
# }

# Docker
docker run --rm \
  -v $(pwd)/migrations:/flyway/sql \
  flyway/flyway \
  -url=jdbc:postgresql://localhost/mydb \
  -user=postgres \
  -password=secret \
  migrate
```

### Flyway Migration Files

```sql
-- V1__Create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE,
    
    CONSTRAINT users_email_check CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at);

-- Trigger สำหรับ updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- V2__Create_products_table.sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE,
    slug VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    parent_id BIGINT REFERENCES categories(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
    stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    category_id BIGINT REFERENCES categories(id),
    sku VARCHAR(100) UNIQUE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_price ON products(price);

-- V3__Create_orders_table.sql
CREATE TYPE order_status AS ENUM (
    'pending', 'confirmed', 'processing', 
    'shipped', 'delivered', 'cancelled', 'refunded'
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status order_status NOT NULL DEFAULT 'pending',
    total_amount DECIMAL(12, 2) NOT NULL CHECK (total_amount >= 0),
    shipping_address JSONB NOT NULL,
    payment_method VARCHAR(50),
    payment_transaction_id VARCHAR(255),
    notes TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) NOT NULL CHECK (unit_price >= 0),
    total_price DECIMAL(12, 2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);

CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at);
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);

-- V4__Add_user_roles.sql
-- Zero-downtime migration example
-- Step 1: เพิ่ม column ใหม่แบบ nullable ก่อน
ALTER TABLE users 
    ADD COLUMN role VARCHAR(50) DEFAULT 'customer';

-- Step 2: Update existing records
UPDATE users SET role = 'customer' WHERE role IS NULL;

-- Step 3: เพิ่ม NOT NULL constraint
ALTER TABLE users 
    ALTER COLUMN role SET NOT NULL;

CREATE INDEX idx_users_role ON users(role);
```

### Flyway Configuration

```properties
# flyway.conf
flyway.url=jdbc:postgresql://localhost:5432/mydb
flyway.user=${DB_USER}
flyway.password=${DB_PASSWORD}
flyway.locations=classpath:db/migration
flyway.baselineOnMigrate=false
flyway.outOfOrder=false
flyway.validateOnMigrate=true
flyway.cleanDisabled=true
flyway.table=flyway_schema_history

# Placeholders
flyway.placeholders.appUser=app_user
flyway.placeholders.schemaName=public
```

### Flyway Java Integration

```java
// MigrationConfig.java
@Configuration
public class MigrationConfig {

    @Bean
    public FlywayMigrationStrategy migrationStrategy() {
        return flyway -> {
            // Validate ก่อน migrate
            flyway.validate();
            
            // รัน migration
            flyway.migrate();
            
            log.info("Database migration completed successfully");
        };
    }
    
    @Bean
    public Flyway flyway(DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration")
            .validateOnMigrate(true)
            .outOfOrder(false)
            .load();
    }
}
```

---

## 39.3 Liquibase

### Liquibase Changelogs

```xml
<!-- db/changelog/db.changelog-master.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">
    
    <include file="db/changelog/changes/001_create_users.xml"/>
    <include file="db/changelog/changes/002_create_products.xml"/>
    <include file="db/changelog/changes/003_create_orders.xml"/>
    <include file="db/changelog/changes/004_add_indexes.xml"/>
    
</databaseChangeLog>
```

```xml
<!-- db/changelog/changes/001_create_users.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">
    
    <changeSet id="001" author="devteam" labels="v1.0.0" context="!test">
        <comment>Create users table</comment>
        
        <createTable tableName="users">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="email" type="VARCHAR(255)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="name" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="password_hash" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="role" type="VARCHAR(50)" defaultValue="customer">
                <constraints nullable="false"/>
            </column>
            <column name="is_active" type="BOOLEAN" defaultValueBoolean="true">
                <constraints nullable="false"/>
            </column>
            <column name="created_at" type="TIMESTAMP WITH TIME ZONE" defaultValueComputed="NOW()">
                <constraints nullable="false"/>
            </column>
            <column name="updated_at" type="TIMESTAMP WITH TIME ZONE" defaultValueComputed="NOW()">
                <constraints nullable="false"/>
            </column>
        </createTable>
        
        <createIndex tableName="users" indexName="idx_users_email">
            <column name="email"/>
        </createIndex>
        
        <rollback>
            <dropTable tableName="users"/>
        </rollback>
    </changeSet>
    
    <!-- Data migration -->
    <changeSet id="001-seed" author="devteam" context="dev,staging">
        <comment>Seed admin user</comment>
        
        <insert tableName="users">
            <column name="email" value="admin@example.com"/>
            <column name="name" value="System Admin"/>
            <column name="password_hash" value="$2b$12$placeholder_hash"/>
            <column name="role" value="admin"/>
        </insert>
        
        <rollback>
            <delete tableName="users">
                <where>email = 'admin@example.com'</where>
            </delete>
        </rollback>
    </changeSet>
</databaseChangeLog>
```

### Liquibase YAML Format

```yaml
# db/changelog/changes/005_add_product_images.yaml
databaseChangeLog:
  - changeSet:
      id: "005"
      author: "devteam"
      changes:
        - createTable:
            tableName: product_images
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: product_id
                  type: BIGINT
                  constraints:
                    nullable: false
                    foreignKeyName: fk_product_images_product
                    references: products(id)
                    deleteCascade: true
              - column:
                  name: url
                  type: VARCHAR(500)
                  constraints:
                    nullable: false
              - column:
                  name: alt_text
                  type: VARCHAR(255)
              - column:
                  name: sort_order
                  type: INTEGER
                  defaultValueNumeric: 0
              - column:
                  name: is_primary
                  type: BOOLEAN
                  defaultValueBoolean: false
        
        - createIndex:
            tableName: product_images
            indexName: idx_product_images_product
            columns:
              - column:
                  name: product_id
      
      rollback:
        - dropTable:
            tableName: product_images
```

---

## 39.4 Python - Alembic

### Alembic Setup

```python
# alembic.ini (สร้างด้วย alembic init)
# script_location = alembic
# sqlalchemy.url = postgresql://user:pass@localhost/db

# env.py
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context
from app.models import Base  # Import your models

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# Use models metadata
target_metadata = Base.metadata

def get_url():
    import os
    return os.environ.get("DATABASE_URL", "postgresql://localhost/mydb")

def run_migrations_offline() -> None:
    url = get_url()
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,
        compare_server_default=True,
    )
    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online() -> None:
    config_section = config.get_section(config.config_ini_section, {})
    config_section["sqlalchemy.url"] = get_url()
    
    connectable = engine_from_config(
        config_section,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    
    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
            compare_server_default=True,
        )
        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

```python
# alembic/versions/001_initial_schema.py
"""Initial schema

Revision ID: 001
Revises: 
Create Date: 2024-01-15 10:00:00.000000
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision = '001'
down_revision = None
branch_labels = None
depends_on = None


def upgrade() -> None:
    # Create users table
    op.create_table(
        'users',
        sa.Column('id', sa.BigInteger(), nullable=False, autoincrement=True),
        sa.Column('email', sa.String(255), nullable=False),
        sa.Column('name', sa.String(255), nullable=False),
        sa.Column('password_hash', sa.String(255), nullable=False),
        sa.Column('role', sa.String(50), nullable=False, server_default='customer'),
        sa.Column('is_active', sa.Boolean(), nullable=False, server_default='true'),
        sa.Column(
            'created_at',
            sa.TIMESTAMP(timezone=True),
            nullable=False,
            server_default=sa.text('NOW()')
        ),
        sa.Column(
            'updated_at',
            sa.TIMESTAMP(timezone=True),
            nullable=False,
            server_default=sa.text('NOW()')
        ),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('email', name='uq_users_email')
    )
    
    op.create_index('idx_users_email', 'users', ['email'])
    op.create_index('idx_users_role', 'users', ['role'])


def downgrade() -> None:
    op.drop_index('idx_users_role', 'users')
    op.drop_index('idx_users_email', 'users')
    op.drop_table('users')
```

```python
# alembic/versions/002_zero_downtime_add_column.py
"""Add phone number to users (zero-downtime)

Revision ID: 002
Revises: 001
Create Date: 2024-01-20 10:00:00.000000
"""
from alembic import op
import sqlalchemy as sa

revision = '002'
down_revision = '001'
branch_labels = None
depends_on = None


def upgrade() -> None:
    """
    Zero-downtime strategy:
    1. Add column as nullable (no downtime)
    2. Backfill data
    3. Add NOT NULL constraint (after all rows have data)
    """
    
    # Step 1: Add nullable column
    op.add_column(
        'users',
        sa.Column('phone_number', sa.String(20), nullable=True)
    )
    
    # Step 2: Create index (CONCURRENTLY ใน PostgreSQL - ไม่ lock table)
    op.execute(
        "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_phone "
        "ON users(phone_number) WHERE phone_number IS NOT NULL"
    )
    
    # Note: Step 3 (NOT NULL constraint) ควรทำใน migration อีกตัว
    # หลังจาก deploy code ที่ populate phone_number แล้ว


def downgrade() -> None:
    op.execute("DROP INDEX CONCURRENTLY IF EXISTS idx_users_phone")
    op.drop_column('users', 'phone_number')
```

---

## 39.5 Zero-Downtime Migration Strategies

### Expand-Contract Pattern

```sql
-- === Migration 1: Expand (เพิ่ม column ใหม่) ===
-- รันก่อน deploy code ใหม่

-- เพิ่ม column ใหม่ (nullable)
ALTER TABLE products ADD COLUMN new_price DECIMAL(12, 4);

-- Copy data
UPDATE products SET new_price = price::DECIMAL(12, 4);

-- === Deploy code ใหม่ที่อ่านทั้ง price และ new_price ===

-- === Migration 2: Contract (ลบ column เก่า) ===
-- รันหลัง deploy code ใหม่ stable แล้ว

ALTER TABLE products 
    ALTER COLUMN new_price SET NOT NULL,
    ALTER COLUMN new_price SET DEFAULT 0;

-- Rename
ALTER TABLE products RENAME COLUMN price TO price_old;
ALTER TABLE products RENAME COLUMN new_price TO price;

-- หลังจาก deploy code ที่ใช้เฉพาะ price แล้ว
ALTER TABLE products DROP COLUMN price_old;
```

### Column Rename Strategy

```python
# alembic/versions/rename_column_safely.py
"""Rename column safely (zero-downtime)

Strategy: Add new column → Backfill → Update app → Drop old column
"""
from alembic import op
import sqlalchemy as sa

def upgrade() -> None:
    """Phase 1: Add new column"""
    # เพิ่ม column ใหม่ที่มีชื่อที่ถูกต้อง
    op.add_column(
        'orders',
        sa.Column('customer_id', sa.BigInteger(), nullable=True)
    )
    
    # Copy data จาก column เก่า
    op.execute("UPDATE orders SET customer_id = user_id WHERE customer_id IS NULL")
    
    # สร้าง trigger เพื่อ sync ระหว่าง transition period
    op.execute("""
        CREATE OR REPLACE FUNCTION sync_customer_id()
        RETURNS TRIGGER AS $$
        BEGIN
            IF NEW.user_id != OLD.user_id OR OLD.user_id IS NULL THEN
                NEW.customer_id = NEW.user_id;
            END IF;
            IF NEW.customer_id != OLD.customer_id OR OLD.customer_id IS NULL THEN
                NEW.user_id = NEW.customer_id;
            END IF;
            RETURN NEW;
        END;
        $$ LANGUAGE plpgsql;
        
        CREATE TRIGGER sync_user_customer_id
        BEFORE UPDATE ON orders
        FOR EACH ROW EXECUTE FUNCTION sync_customer_id();
    """)
    
    # Phase 2 จะทำใน migration ถัดไป หลัง deploy code ใหม่:
    # - Drop trigger
    # - Add NOT NULL constraint  
    # - Drop old column


def downgrade() -> None:
    op.execute("DROP TRIGGER IF EXISTS sync_user_customer_id ON orders")
    op.execute("DROP FUNCTION IF EXISTS sync_customer_id()")
    op.drop_column('orders', 'customer_id')
```

### Large Table Migration

```python
# alembic/versions/migrate_large_table.py
"""Backfill large table in batches"""

from alembic import op
import sqlalchemy as sa
from sqlalchemy import text

BATCH_SIZE = 10000

def upgrade() -> None:
    """Backfill data in batches เพื่อหลีกเลี่ยง lock ที่นานเกินไป"""
    
    connection = op.get_bind()
    
    # นับ total records
    result = connection.execute(text("SELECT COUNT(*) FROM products WHERE slug IS NULL"))
    total = result.scalar()
    
    print(f"Backfilling {total} records in batches of {BATCH_SIZE}...")
    
    processed = 0
    while True:
        # Update batch
        result = connection.execute(text(f"""
            WITH batch AS (
                SELECT id FROM products 
                WHERE slug IS NULL 
                LIMIT {BATCH_SIZE}
                FOR UPDATE SKIP LOCKED
            )
            UPDATE products p
            SET slug = LOWER(REGEXP_REPLACE(p.name, '[^a-zA-Z0-9]', '-', 'g'))
                    || '-' || p.id::text
            FROM batch
            WHERE p.id = batch.id
            RETURNING p.id
        """))
        
        updated = result.rowcount
        processed += updated
        
        if updated == 0:
            break
        
        print(f"Progress: {processed}/{total}")
        
        # ทำ checkpoint เพื่อให้ WAL ไม่โต
        connection.execute(text("SELECT pg_sleep(0.1)"))
    
    print(f"Backfill complete: {processed} records updated")

def downgrade() -> None:
    pass  # ไม่ต้อง rollback data
```

---

## 39.6 Testing Migrations

### Migration Testing

```python
# tests/test_migrations.py
import pytest
from alembic.config import Config
from alembic import command
from sqlalchemy import create_engine, text, inspect
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:15-alpine") as postgres:
        yield postgres

@pytest.fixture(scope="session")
def engine(postgres_container):
    engine = create_engine(postgres_container.get_connection_url())
    yield engine
    engine.dispose()

@pytest.fixture
def clean_database(engine):
    """Drop และ recreate database ก่อนแต่ละ test"""
    with engine.connect() as conn:
        conn.execute(text("DROP SCHEMA public CASCADE"))
        conn.execute(text("CREATE SCHEMA public"))
        conn.commit()
    yield engine

class TestMigrations:
    """ทดสอบ database migrations"""
    
    def get_alembic_config(self, database_url: str) -> Config:
        """สร้าง Alembic config"""
        alembic_cfg = Config("alembic.ini")
        alembic_cfg.set_main_option("sqlalchemy.url", database_url)
        return alembic_cfg
    
    def test_upgrade_from_zero(self, postgres_container, clean_database):
        """ทดสอบ upgrade จาก empty database"""
        alembic_cfg = self.get_alembic_config(
            postgres_container.get_connection_url()
        )
        
        # Upgrade to latest
        command.upgrade(alembic_cfg, "head")
        
        # ตรวจสอบว่า tables ถูกสร้าง
        inspector = inspect(clean_database)
        tables = inspector.get_table_names()
        
        assert "users" in tables
        assert "products" in tables
        assert "orders" in tables
        assert "order_items" in tables
    
    def test_downgrade_and_upgrade(self, postgres_container, clean_database):
        """ทดสอบ downgrade และ upgrade กลับ"""
        alembic_cfg = self.get_alembic_config(
            postgres_container.get_connection_url()
        )
        
        # Upgrade to latest
        command.upgrade(alembic_cfg, "head")
        
        # Downgrade all the way
        command.downgrade(alembic_cfg, "base")
        
        # ตรวจสอบว่า tables ถูกลบ
        inspector = inspect(clean_database)
        tables = inspector.get_table_names()
        assert "users" not in tables
        
        # Upgrade อีกครั้ง
        command.upgrade(alembic_cfg, "head")
        
        inspector = inspect(clean_database)
        tables = inspector.get_table_names()
        assert "users" in tables
    
    def test_each_migration_is_reversible(self, postgres_container, clean_database):
        """ทดสอบว่าแต่ละ migration สามารถ rollback ได้"""
        alembic_cfg = self.get_alembic_config(
            postgres_container.get_connection_url()
        )
        
        # ดึง list ของ revisions
        from alembic.script import ScriptDirectory
        script_dir = ScriptDirectory.from_config(alembic_cfg)
        revisions = list(script_dir.walk_revisions())
        revisions.reverse()  # เรียง oldest to newest
        
        for i, revision in enumerate(revisions):
            # Upgrade ไปที่ revision นี้
            command.upgrade(alembic_cfg, revision.revision)
            
            # Downgrade 1 step
            command.downgrade(alembic_cfg, "-1")
            
            # Upgrade อีกครั้ง
            command.upgrade(alembic_cfg, revision.revision)
    
    def test_migration_schema_matches_models(self, postgres_container, clean_database):
        """ทดสอบว่า schema จาก migrations ตรงกับ SQLAlchemy models"""
        from sqlalchemy_utils import database_exists
        
        alembic_cfg = self.get_alembic_config(
            postgres_container.get_connection_url()
        )
        command.upgrade(alembic_cfg, "head")
        
        # ตรวจสอบ columns ของ users table
        inspector = inspect(clean_database)
        columns = {c['name']: c for c in inspector.get_columns('users')}
        
        required_columns = ['id', 'email', 'name', 'password_hash', 'role', 
                           'is_active', 'created_at', 'updated_at']
        
        for col in required_columns:
            assert col in columns, f"Missing column: {col}"
        
        # ตรวจสอบ indexes
        indexes = inspector.get_indexes('users')
        index_names = {idx['name'] for idx in indexes}
        
        assert 'idx_users_email' in index_names
```

---

## 39.7 GitHub Actions Integration

```yaml
# .github/workflows/database-migrations.yml
name: Database Migrations

on:
  push:
    branches: [main]
    paths:
      - 'alembic/**'
      - 'migrations/**'
      - 'db/**'
  pull_request:
    branches: [main]
    paths:
      - 'alembic/**'
      - 'migrations/**'
      - 'db/**'

jobs:
  test-migrations:
    name: Test Migrations
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    env:
      DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install alembic pytest testcontainers
      
      - name: Validate migration files
        run: |
          python -c "
          from alembic.config import Config
          from alembic.script import ScriptDirectory
          from alembic import command
          
          cfg = Config('alembic.ini')
          script_dir = ScriptDirectory.from_config(cfg)
          
          # ตรวจสอบว่าไม่มี duplicate revision IDs
          revisions = list(script_dir.walk_revisions())
          ids = [r.revision for r in revisions]
          assert len(ids) == len(set(ids)), 'Duplicate revision IDs found!'
          
          print(f'Found {len(revisions)} migrations - all valid')
          "
      
      - name: Test upgrade to head
        run: |
          alembic upgrade head
          echo "✅ Migration to head successful"
      
      - name: Test downgrade to base
        run: |
          alembic downgrade base
          echo "✅ Downgrade to base successful"
      
      - name: Test upgrade again (verify idempotency)
        run: |
          alembic upgrade head
          echo "✅ Re-migration successful"
      
      - name: Run migration tests
        run: |
          pytest tests/test_migrations.py -v \
            --junit-xml=test-results/migration-results.xml
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: migration-test-results
          path: test-results/
  
  deploy-migration:
    name: Deploy Migration (Production)
    runs-on: ubuntu-latest
    needs: test-migrations
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install alembic psycopg2-binary
      
      - name: Check pending migrations
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
        run: |
          PENDING=$(alembic current 2>&1)
          echo "Current migration: $PENDING"
          
          HEADS=$(alembic heads 2>&1)
          echo "Latest migration: $HEADS"
      
      - name: Create database backup
        env:
          PGPASSWORD: ${{ secrets.DB_PASSWORD }}
        run: |
          pg_dump \
            -h ${{ secrets.DB_HOST }} \
            -U ${{ secrets.DB_USER }} \
            -d ${{ secrets.DB_NAME }} \
            --format=custom \
            --file=backup-$(date +%Y%m%d-%H%M%S).dump
          
          # Upload backup ไปยัง S3
          aws s3 cp backup-*.dump \
            s3://${{ secrets.BACKUP_BUCKET }}/migrations/ \
            --sse aws:kms
      
      - name: Run migration with timeout
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
        run: |
          # Set lock timeout เพื่อ fail fast ถ้า lock ไม่ได้
          export PGOPTIONS="-c lock_timeout=30s"
          
          alembic upgrade head
          echo "✅ Production migration completed"
      
      - name: Verify migration
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
        run: |
          # ตรวจสอบว่า migration ทำงานสำเร็จ
          python -c "
          from alembic.config import Config
          from alembic.runtime.migration import MigrationContext
          from alembic.script import ScriptDirectory
          from sqlalchemy import create_engine
          import os
          
          engine = create_engine(os.environ['DATABASE_URL'])
          with engine.connect() as conn:
              context = MigrationContext.configure(conn)
              current = context.get_current_revision()
          
          cfg = Config('alembic.ini')
          script_dir = ScriptDirectory.from_config(cfg)
          head = script_dir.get_current_head()
          
          assert current == head, f'Migration failed! Current: {current}, Expected: {head}'
          print(f'✅ Migration verified: {current}')
          "
      
      - name: Notify on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const message = `
            ❌ Production database migration FAILED!
            
            Branch: ${context.ref}
            Commit: ${context.sha}
            
            Please check the migration logs immediately.
            `;
            
            // Post to Slack
            await fetch(process.env.SLACK_WEBHOOK_URL, {
              method: 'POST',
              headers: {'Content-Type': 'application/json'},
              body: JSON.stringify({text: message})
            });
```

---

## 39.8 แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Migration Set

สร้าง migrations สำหรับ Blog application:
1. Create users table
2. Create posts table (foreign key ไปยัง users)
3. Create comments table
4. Add tags และ post_tags (many-to-many)
5. เพิ่ม full-text search index

### แบบฝึกหัดที่ 2: Zero-Downtime Column Rename

ทำ zero-downtime migration เพื่อ rename `user_name` เป็ `display_name` ใน users table:
1. Phase 1: Add new column, sync data
2. Deploy new code
3. Phase 2: Drop old column

### แบบฝึกหัดที่ 3: Data Migration

สร้าง migration ที่:
1. แยก `full_name` column เป็น `first_name` และ `last_name`
2. Migrate ข้อมูลเก่า (parse full_name)
3. ทำ batch update เพื่อหลีกเลี่ยง lock
4. เขียน rollback script

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **Flyway**: SQL-based migrations สำหรับ JVM projects
- **Liquibase**: XML/YAML/JSON migrations ที่รองรับหลาย databases
- **Alembic**: Python migrations สำหรับ SQLAlchemy
- **Zero-downtime strategies**: Expand-contract, column rename, batch updates
- **Migration testing**: ใช้ Testcontainers เพื่อ test migrations
- **CI/CD integration**: Automated migration deployment ที่ปลอดภัย

บทต่อไป (Part 40) เราจะเรียนรู้เรื่อง **Feature Flags ใน CI/CD**
