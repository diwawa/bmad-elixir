# Ecto 最佳实践检查清单

在使用 Ecto 模式、迁移和数据库操作时使用此检查清单。

## 模式设计

### 字段
- [ ] 所有字段都有适当的类型（:string, :integer, :decimal, :utc_datetime 等）
- [ ] 适当的位置定义了字符串长度限制
- [ ] 金钱/货币字段定义了小数精度
- [ ] 枚举字段使用 Ecto.Enum 和值列表
- [ ] 虚拟字段标记为 `virtual: true`
- [ ] 默认值在数据库中设置，不在模式中（除非是虚拟字段）

### 关联
- [ ] belongs_to 关联有相应的外键
- [ ] has_many 关联在需要时在两个方向都定义
- [ ] many_to_many 使用连接表或 has_many :through
- [ ] 外键有删除策略（:delete_all, :nilify_all, :nothing）
- [ ] 避免循环关联或谨慎管理

### 时间戳
- [ ] 使用 `timestamps()` 宏
- [ ] 如果不是默认值则指定类型：`timestamps(type: :utc_datetime)`
- [ ] 手动时间戳字段使用 :utc_datetime，不使用 :naive_datetime

## Changesets

### 验证
- [ ] `cast/3` 只包含可填充字段
- [ ] `validate_required/2` 用于所有必填字段
- [ ] 验证字符串格式（电子邮件、URL 等）
- [ ] 验证数字范围（最小值、最大值）
- [ ] 验证字符串长度
- [ ] 复杂规则的自定义验证
- [ ] 使用时验证虚拟字段

### 约束
- [ ] 唯一索引使用 `unique_constraint`
- [ ] 外键使用 `foreign_key_constraint`
- [ ] 数据库级检查使用 `check_constraint`
- [ ] 防止删除关联时使用 `no_assoc_constraint`
- [ ] 约束在数据库操作后运行

### 嵌入模式
- [ ] JSON 字段使用嵌入模式
- [ ] 根据情况使用 `embeds_one` 或 `embeds_many`
- [ ] 嵌入 changesets 的验证
- [ ] JSON 编码/解码处理

## 迁移

### 结构
- [ ] 迁移文件有描述性名称
- [ ] 每个迁移包含一个逻辑更改
- [ ] 迁移是可逆的（`down` 函数或 `change`）
- [ ] 上升和下降都已测试
- [ ] 尽可能幂等

### 表
- [ ] 定义主键（通常是 bigserial `id`）
- [ ] 添加时间戳（`timestamps()`）
- [ ] 外键引用正确的表
- [ ] 指定删除/更新操作
- [ ] 表名是复数

### 索引
- [ ] 唯一约束有唯一索引
- [ ] 外键有索引
- [ ] 常查询字段有索引
- [ ] 多列查询的复合索引
- [ ] 索引名称遵循约定或显式设置

### 数据类型
- [ ] 每列使用适当的类型
- [ ] :text 用于无限制字符串
- [ ] :string 与限制用于有限字符串
- [ ] :decimal 用于金钱（不是 :float）
- [ ] :utc_datetime 用于时间戳
- [ ] :map 或 :jsonb 用于 JSON 数据

### 约束
- [ ] 必填字段有 NOT NULL
- [ ] 业务规则有 CHECK 约束
- [ ] 按需有 UNIQUE 约束
- [ ] 外键约束
- [ ] 合理的默认值

## 查询

### 查询构建
- [ ] 使用 Ecto.Query DSL，不使用原始 SQL
- [ ] 查询是可组合的（使用 from, where, select 等）
- [ ] 预加载关联以避免 N+1
- [ ] 通过关联过滤时使用连接
- [ ] 复杂查询时使用子查询

### 预加载
- [ ] `preload` 用于简单关联
- [ ] `preload: [assoc: query]` 用于过滤的关联
- [ ] 尽可能在单个查询中预加载
- [ ] 避免在循环中预加载

### 性能
- [ ] 无 N+1 查询（使用预加载或连接）
- [ ] 大查询中只选择需要的字段
- [ ] 大结果集使用分页
- [ ] 仅需要部分记录时使用限制
- [ ] 索引支持 WHERE 子句

### 事务
- [ ] 多步骤操作使用 Repo.transaction
- [ ] 复杂事务使用 Ecto.Multi
- [ ] 错误时回滚
- [ ] 保持事务简短

## 多租户

如果实现多租户：

### 模式级别
- [ ] 所有相关表上有 tenant_id
- [ ] tenant_id 在唯一约束中
- [ ] 适当时外键限定租户范围
- [ ] 数据隔离已验证

### 查询级别
- [ ] 所有查询按 tenant_id 过滤
- [ ] 无跨租户数据泄漏
- [ ] 每租户授权检查
- [ ] 测试验证租户隔离

## 测试

### 模式测试
- [ ] 测试 changeset 验证
- [ ] 测试必填字段
- [ ] 测试格式验证（电子邮件等）
- [ ] 测试唯一约束
- [ ] 测试外键约束
- [ ] 测试自定义验证

### 查询测试
- [ ] 查询返回预期结果
- [ ] 过滤器正常工作
- [ ] 预加载正常工作
- [ ] 边缘情况测试（空结果等）

## 示例

### 良好模式设计
```elixir
schema "users" do
  field :email, :string
  field :name, :string
  field :role, Ecto.Enum, values: [:admin, :user, :guest]
  field :password, :string, virtual: true
  field :password_hash, :string

  belongs_to :organization, Organization
  has_many :posts, Post
  has_many :comments, Comment

  timestamps(type: :utc_datetime)
end

def changeset(user, attrs) do
  user
  |> cast(attrs, [:email, :name, :role, :password, :organization_id])
  |> validate_required([:email, :name, :organization_id])
  |> validate_format(:email, ~r/@/)
  |> validate_length(:name, min: 2, max: 100)
  |> validate_inclusion(:role, [:admin, :user, :guest])
  |> unique_constraint(:email)
  |> foreign_key_constraint(:organization_id)
  |> hash_password()
end
```

### 良好迁移
```elixir
def change do
  create table(:users) do
    add :email, :string, null: false
    add :name, :string, null: false
    add :role, :string, null: false, default: "user"
    add :password_hash, :string, null: false
    add :organization_id, references(:organizations, on_delete: :delete_all), null: false

    timestamps(type: :utc_datetime)
  end

  create unique_index(:users, [:email])
  create index(:users, [:organization_id])
  create index(:users, [:email, :organization_id])
end
```

### 良好查询
```elixir
def list_active_users_with_posts(organization_id) do
  from(u in User,
    where: u.organization_id == ^organization_id,
    where: u.active == true,
    preload: [:posts],
    order_by: [desc: u.inserted_at]
  )
  |> Repo.all()
end
```

## 常见陷阱

❌ **使用浮点数表示金钱**
```elixir
field :price, :float  # 错误 - 精度问题
field :price, :decimal  # 正确
```

❌ **N+1 查询**
```elixir
# 坏 - N+1 查询
users = Repo.all(User)
Enum.map(users, fn user -> user.posts end)  # 每个用户查询！

# 好 - 预加载
users = Repo.all(User) |> Repo.preload(:posts)
Enum.map(users, fn user -> user.posts end)  # 已加载
```

❌ **不使用约束**
```elixir
# 坏 - 无约束
def changeset(user, attrs) do
  user
  |> cast(attrs, [:email])
  |> validate_required([:email])
  # 电子邮件仍可能重复！
end

# 好 - 有约束
def changeset(user, attrs) do
  user
  |> cast(attrs, [:email])
  |> validate_required([:email])
  |> unique_constraint(:email)  # 捕获数据库级重复
end
```

❌ **缺少外键索引**
```elixir
# 坏 - 外键上无索引
create table(:posts) do
  add :user_id, references(:users)
end

# 好 - 有索引
create table(:posts) do
  add :user_id, references(:users)
end
create index(:posts, [:user_id])
```

❌ **不处理错误**
```elixir
# 坏 - 未处理错误
def create_user(attrs) do
  %User{}
  |> User.changeset(attrs)
  |> Repo.insert!()  # 错误时抛出！
end

# 好 - 返回元组
def create_user(attrs) do
  %User{}
  |> User.changeset(attrs)
  |> Repo.insert()  # 返回 {:ok, user} 或 {:error, changeset}
end
```

## 性能提示

- **使用 select 仅加载大型数据集所需的字段**
- **使用分页**（限制 + 偏移或基于游标）
- **为 WHERE、ORDER BY 和 JOIN 列添加索引**
- **批量操作使用 insert_all/update_all**
- **在开发中使用 `config :logger, :console, format: "[$level] $message\n"` 配置文件查询**
- **在生产中监控慢查询**

## 合并前

- [ ] 所有迁移已测试（上升和下降）
- [ ] 所有约束都有相应的验证
- [ ] 未引入 N+1 查询
- [ ] 为外键和常用查询添加索引
- [ ] Changesets 验证所有业务规则
- [ ] 测试覆盖正常路径和错误情况
- [ ] 模式文档完整