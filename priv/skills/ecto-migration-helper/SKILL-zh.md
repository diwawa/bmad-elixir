---
name: ecto-migration-helper
description: 使用适当的回滚处理和最佳实践创建、管理和安全运行 Ecto 数据库迁移。在处理数据库模式更改、添加列或修改约束时使用。
allowed-tools: Bash, Read, Edit, Write
---

# Ecto 迁移助手

此技能帮助安全创建和管理 Ecto 迁移，具有适当的模式和回滚支持。

## 使用时机

- 创建新迁移
- 修改现有表
- 添加/删除索引
- 更改约束
- 数据迁移
- 回滚迁移

## 创建迁移

### 生成空迁移
```bash
mix ecto.gen.migration add_email_to_users
```

创建：`priv/repo/migrations/TIMESTAMP_add_email_to_users.exs`

### 迁移命名约定
- `create_table_name` - 创建新表
- `add_field_to_table` - 添加列
- `remove_field_from_table` - 删除列
- `add_index_to_table_on_field` - 添加索引
- `modify_field_in_table` - 更改列类型
- `add_constraint_to_table` - 添加约束

## 常见迁移模式

### 添加列
```elixir
defmodule MyApp.Repo.Migrations.AddEmailToUsers do
  use Ecto.Migration

  def change do
    alter table(:users) do
      add :email, :string
    end
  end
end
```

### 添加带默认值的列
```elixir
def change do
  alter table(:users) do
    add :active, :boolean, default: true, null: false
  end
end
```

### 添加带索引的列
```elixir
def change do
  alter table(:users) do
    add :email, :string
  end

  create unique_index(:users, [:email])
end
```

### 添加外键
```elixir
def change do
  alter table(:posts) do
    add :user_id, references(:users, on_delete: :delete_all), null: false
  end

  create index(:posts, [:user_id])
end
```

### 删除列
```elixir
def change do
  alter table(:users) do
    remove :old_field
  end
end
```

**警告**：使用 `change` 删除列是不可逆的。使用 `up`/`down`：

```elixir
def up do
  alter table(:users) do
    remove :old_field
  end
end

def down do
  alter table(:users) do
    add :old_field, :string
  end
end
```

### 修改列类型
```elixir
def change do
  alter table(:products) do
    modify :price, :decimal, precision: 10, scale: 2
  end
end
```

### 重命名列
```elixir
def change do
  rename table(:users), :username, to: :name
end
```

### 添加复合索引
```elixir
def change do
  create index(:posts, [:user_id, :published_at])
end
```

### 添加唯一约束
```elixir
def change do
  create unique_index(:users, [:email])
  create unique_index(:users, [:organization_id, :email])  # 复合唯一
end
```

### 添加检查约束
```elixir
def change do
  create constraint(:products, :price_must_be_positive, check: "price > 0")
end
```

## 安全迁移模式

### 使列不可为 NULL

**错误**（如果存在 NULL 会失败）：
```elixir
def change do
  alter table(:users) do
    modify :email, :string, null: false  # 失败！
  end
end
```

**正确**（两步方法）：
```elixir
# 迁移 1：添加默认值，填充 NULL
def change do
  # 为新行设置默认值
  alter table(:users) do
    modify :email, :string, default: "unknown@example.com"
  end

  # 填充现有 NULL
  execute(
    "UPDATE users SET email = 'unknown@example.com' WHERE email IS NULL",
    ""  # 不需要回滚
  )
end

# 迁移 2：添加 NOT NULL 约束
def change do
  alter table(:users) do
    modify :email, :string, null: false
  end
end
```

### 安全删除列

**步骤 1**：部署不使用该列的代码
**步骤 2**：运行删除列的迁移（部署后）

```elixir
# 在代码不再引用该字段后部署此迁移
def up do
  alter table(:users) do
    remove :old_field
  end
end

def down do
  alter table(:users) do
    add :old_field, :string  # 指定回滚类型
  end
end
```

### 大型数据迁移

使用批处理以避免锁定：
```elixir
def up do
  execute """
  UPDATE users
  SET status = 'active'
  WHERE status IS NULL
  AND id IN (SELECT id FROM users WHERE status IS NULL LIMIT 1000)
  """

  # 分批重复或使用递归函数
end
```

## 数据迁移

### 填充数据
```elixir
defmodule MyApp.Repo.Migrations.BackfillUserDefaults do
  use Ecto.Migration
  import Ecto.Query
  alias MyApp.Repo
  alias MyApp.Accounts.User

  def up do
    # 在迁移中小心使用应用程序代码
    User
    |> where([u], is_nil(u.status))
    |> Repo.update_all(set: [status: "active"])
  end

  def down do
    # 数据迁移通常不需要回滚
    :ok
  end
end
```

### 复杂数据迁移（单独模块）
```elixir
defmodule MyApp.Repo.Migrations.MigrateUserData do
  use Ecto.Migration

  def up do
    MyApp.ReleaseTasks.migrate_user_data()
  end

  def down do
    :ok
  end
end

# 在 lib/my_app/release_tasks.ex 中
defmodule MyApp.ReleaseTasks do
  def migrate_user_data do
    # 复杂逻辑在这里
  end
end
```

## 运行迁移

### 开发环境
```bash
# 运行所有待处理的迁移
mix ecto.migrate

# 运行到特定版本
mix ecto.migrate --to 20250101120000

# 回滚最后的迁移
mix ecto.rollback

# 回滚最后 3 个迁移
mix ecto.rollback --step 3

# 回滚到特定版本
mix ecto.rollback --to 20250101120000
```

### 测试环境
```bash
# 创建测试数据库
MIX_ENV=test mix ecto.create

# 在测试中运行迁移
MIX_ENV=test mix ecto.migrate

# 重置测试数据库（删除、创建、迁移）
MIX_ENV=test mix ecto.reset
```

### 生产环境
```bash
# 在生产环境中运行（通常通过发布任务）
bin/my_app eval "MyApp.ReleaseTasks.migrate()"

# 或如果 mix 可用
MIX_ENV=prod mix ecto.migrate
```

## 迁移状态

```bash
# 检查迁移状态
mix ecto.migrations

# 输出显示：
# 状态    迁移 ID        迁移名称
# --------------------------------------------------
# up      20250101120000  create_users
# up      20250101130000  add_email_to_users
# down    20250101140000  add_profile_to_users
```

## 可逆与不可逆

### 可逆（使用 `change`）
- 添加列
- 创建表
- 添加索引
- 添加引用

### 不可逆（使用 `up`/`down`）
- 删除列（数据丢失）
- execute() 与 SQL
- 数据转换
- 删除表

## 最佳实践

### 1. 每个迁移一个逻辑更改
```bash
# 好：专注的迁移
mix ecto.gen.migration add_email_to_users

# 坏：多个不相关的更改
mix ecto.gen.migration update_users_and_posts_and_comments
```

### 2. 始终为外键添加索引
```elixir
add :user_id, references(:users)
create index(:posts, [:user_id])  # 始终添加这个！
```

### 3. 为外键指定 on_delete
```elixir
# 明确级联行为
add :user_id, references(:users, on_delete: :delete_all)  # 级联
add :user_id, references(:users, on_delete: :nilify_all)  # 设置 NULL
add :user_id, references(:users, on_delete: :restrict)    # 防止删除
add :user_id, references(:users, on_delete: :nothing)     # 无操作
```

### 4. 为小数指定精度
```elixir
# 好
add :price, :decimal, precision: 10, scale: 2

# 坏（数据库决定精度）
add :price, :decimal
```

### 5. 明确约束
```elixir
# 电子邮件应该是唯一的且不可为空
add :email, :string, null: false
create unique_index(:users, [:email])
```

### 6. 在本地测试回滚
```bash
# 创建迁移后
mix ecto.migrate
mix ecto.rollback
mix ecto.migrate
```

## 故障排除

### 迁移失败

**列已存在：**
```bash
# 检查当前模式
mix ecto.migrations

# 在开发中删除并重新创建
mix ecto.drop && mix ecto.create && mix ecto.migrate
```

**无法回滚：**
- 检查迁移是否使用 `change` 与 `up`/`down`
- 查看迁移是否有不可逆操作
- 可能需要编写自定义 `down` 函数

**锁超时：**
```elixir
# 为迁移添加超时
@disable_ddl_transaction true  # 对于不能在事务中运行的操作
@disable_migration_lock true   # 对于长时间运行的迁移

def change do
  # 迁移代码
end
```

### 数据迁移问题

**大表超时：**
- 使用批处理
- 考虑在迁移之外运行（Rails 风格的 rake 任务）
- 使用 `@disable_ddl_transaction true`

**引用应用程序代码：**
- 小心模式更改
- 应用程序代码可能会更改，迁移不会
- 考虑对数据迁移使用原始 SQL

## 高级模式

### 并发索引创建（PostgreSQL）
```elixir
@disable_ddl_transaction true

def change do
  create index(:posts, [:user_id], concurrently: true)
end
```

### 条件迁移
```elixir
def change do
  if function_exported?(MyApp.Repo, :__adapter__, 0) do
    # 迁移代码
  end
end
```

### 时间戳助手
```elixir
create table(:users) do
  add :name, :string
  timestamps()  # 添加 inserted_at 和 updated_at
end
```
