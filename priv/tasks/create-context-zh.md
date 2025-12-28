# 任务：创建 Phoenix 上下文

**目的**：添加新的有界上下文来组织领域逻辑

**代理**：elixir-dev

**持续时间**：2-4 小时

## 概述

Phoenix 上下文是专门的模块，用于公开和分组相关功能。它们提供清晰的 API 边界并封装业务逻辑。

## 何时创建上下文

在以下情况下创建新上下文：
- 添加新的领域概念（Accounts、Catalog、Billing 等）
- 分组不适合现有上下文的相关功能
- 在系统的不同部分之间强制执行边界

## 步骤 1：定义上下文边界

**持续时间**：15-30 分钟

回答这些问题：
- 领域概念是什么？（例如，"Accounts" 用于用户管理）
- 此上下文将提供哪些操作？
- 此上下文拥有哪些数据？
- 它如何与其他上下文相关？

**示例：**
```
领域：Catalog
目的：管理产品、类别和库存
操作：产品的 CRUD、搜索、过滤、库存管理
拥有：products、categories、product_categories 表
关系：
  - Accounts 上下文（产品属于卖家）
  - Orders 上下文（订单项引用产品）
```

## 步骤 2：生成带有模式的上下文

**持续时间**：5-10 分钟

使用 Phoenix 生成器：

```bash
mix phx.gen.context ContextName SchemaName table_name field:type field:type ...
```

**示例：**
```bash
mix phx.gen.context Catalog Product products \
  name:string \
  description:text \
  price:decimal \
  sku:string:unique \
  quantity:integer \
  active:boolean
```

这将生成：
- `lib/my_app/catalog.ex` - 上下文模块，包含 CRUD 函数
- `lib/my_app/catalog/product.ex` - 模式模块
- `priv/repo/migrations/20240115120000_create_products.exs` - 迁移
- `test/my_app/catalog_test.exs` - 上下文测试
- `test/support/fixtures/catalog_fixtures.ex` - 测试夹具

## 步骤 3：自定义模式

**持续时间**：30 分钟 - 1 小时

### 添加关联

```elixir
# lib/my_app/catalog/product.ex
defmodule MyApp.Catalog.Product do
  use Ecto.Schema
  import Ecto.Changeset

  schema "products" do
    field :name, :string
    field :description, :string  # 注意：即使是 :text 列也使用 :string 类型
    field :price, :decimal
    field :sku, :string
    field :quantity, :integer
    field :active, :boolean, default: true

    # 添加关联
    belongs_to :category, MyApp.Catalog.Category
    belongs_to :seller, MyApp.Accounts.User
    has_many :order_items, MyApp.Orders.OrderItem
    has_many :reviews, MyApp.Catalog.Review

    timestamps(type: :utc_datetime)
  end

  @doc false
  def changeset(product, attrs) do
    product
    |> cast(attrs, [:name, :description, :price, :sku, :quantity, :active, :category_id])
    |> validate_required([:name, :price, :sku])
    |> validate_length(:name, min: 2, max: 100)
    |> validate_length(:description, max: 5000)
    |> validate_number(:price, greater_than: 0)
    |> validate_number(:quantity, greater_than_or_equal_to: 0)
    |> validate_format(:sku, ~r/^[A-Z0-9-]+$/, message: "必须是大写字母数字和连字符")
    |> unique_constraint(:sku)
    |> foreign_key_constraint(:category_id)
    |> foreign_key_constraint(:seller_id)
  end
end
```

### 添加虚拟字段（如需要）

```elixir
schema "products" do
  # ... 常规字段

  # 虚拟字段用于搜索高亮
  field :search_rank, :float, virtual: true

  # 虚拟字段用于 API 响应
  field :display_price, :string, virtual: true
end
```

### 添加自定义验证

```elixir
def changeset(product, attrs) do
  product
  |> cast(attrs, [:name, :price, :quantity, :active])
  |> validate_required([:name, :price])
  |> validate_price_for_active_products()
  |> validate_stock_levels()
end

defp validate_price_for_active_products(changeset) do
  active = get_field(changeset, :active)
  price = get_field(changeset, :price)

  if active && Decimal.compare(price, 0) != :gt do
    add_error(changeset, :price, "活动产品的价格必须大于 0")
  else
    changeset
  end
end

defp validate_stock_levels(changeset) do
  quantity = get_field(changeset, :quantity)
  active = get_field(changeset, :active)

  if active && quantity == 0 do
    add_error(changeset, :quantity, "活动产品必须有库存")
  else
    changeset
  end
end
```

## 步骤 4：增强迁移

**持续时间**：15-30 分钟

添加索引、约束和适当的默认值：

```elixir
# priv/repo/migrations/20240115120000_create_products.exs
defmodule MyApp.Repo.Migrations.CreateProducts do
  use Ecto.Migration

  def change do
    create table(:products) do
      add :name, :string, null: false
      add :description, :text
      add :price, :decimal, precision: 10, scale: 2, null: false
      add :sku, :string, null: false
      add :quantity, :integer, default: 0, null: false
      add :active, :boolean, default: true, null: false

      # 外键与 on_delete 操作
      add :category_id, references(:categories, on_delete: :nilify_all)
      add :seller_id, references(:users, on_delete: :delete_all), null: false

      timestamps(type: :utc_datetime)
    end

    # 唯一约束
    create unique_index(:products, [:sku])

    # 外键索引
    create index(:products, [:category_id])
    create index(:products, [:seller_id])

    # 查询优化索引
    create index(:products, [:active])
    create index(:products, [:active, :category_id])

    # 全文搜索索引（PostgreSQL 特定）
    execute(
      "CREATE INDEX products_name_trgm_idx ON products USING gin (name gin_trgm_ops)",
      "DROP INDEX products_name_trgm_idx"
    )

    # 检查约束
    create constraint(:products, :price_must_be_positive, check: "price > 0")
    create constraint(:products, :quantity_must_be_non_negative, check: "quantity >= 0")
  end
end
```

## 步骤 5：实现上下文 API

**持续时间**：1-2 小时

### 基本 CRUD 操作

生成器创建这些，但根据需要进行自定义：

```elixir
# lib/my_app/catalog.ex
defmodule MyApp.Catalog do
  @moduledoc """
  Catalog 上下文。

  管理产品、类别和库存操作。
  """

  import Ecto.Query, warn: false
  alias MyApp.Repo
  alias MyApp.Catalog.Product

  @doc """
  返回产品列表。

  ## 示例

      iex> list_products()
      [%Product{}, ...]

  """
  def list_products do
    Repo.all(Product)
  end

  @doc """
  返回带有过滤器和排序的产品列表。

  ## 选项

    * `:preload` - 要预加载的关联列表
    * `:filters` - 过滤器映射（active、category_id、search）
    * `:sort_by` - 要排序的字段（默认：:inserted_at）
    * `:sort_order` - :asc 或 :desc（默认：:desc）

  ## 示例

      iex> list_products(preload: [:category], filters: %{active: true})
      [%Product{category: %Category{}}, ...]

  """
  def list_products(opts \\ []) do
    Product
    |> apply_filters(opts[:filters] || %{})
    |> apply_sorting(opts[:sort_by], opts[:sort_order])
    |> maybe_preload(opts[:preload])
    |> Repo.all()
  end

  defp apply_filters(query, filters) do
    Enum.reduce(filters, query, fn {key, value}, query ->
      apply_filter(query, key, value)
    end)
  end

  defp apply_filter(query, :active, value) do
    where(query, [p], p.active == ^value)
  end

  defp apply_filter(query, :category_id, value) do
    where(query, [p], p.category_id == ^value)
  end

  defp apply_filter(query, :search, value) when is_binary(value) do
    search_term = "%#{value}%"
    where(query, [p], ilike(p.name, ^search_term) or ilike(p.description, ^search_term))
  end

  defp apply_filter(query, :min_price, value) do
    where(query, [p], p.price >= ^value)
  end

  defp apply_filter(query, :max_price, value) do
    where(query, [p], p.price <= ^value)
  end

  defp apply_filter(query, _key, _value), do: query

  defp apply_sorting(query, nil, _order), do: order_by(query, [desc: :inserted_at])
  defp apply_sorting(query, field, :asc), do: order_by(query, [asc: ^field])
  defp apply_sorting(query, field, _order), do: order_by(query, [desc: ^field])

  defp maybe_preload(query, nil), do: query
  defp maybe_preload(query, preloads), do: preload(query, ^preloads)

  @doc """
  获取单个产品。

  如果找到则返回 `{:ok, product}`，否则返回 `{:error, :not_found}`。

  ## 示例

      iex> get_product(123)
      {:ok, %Product{}}

      iex> get_product(456)
      {:error, :not_found}

  """
  def get_product(id) do
    case Repo.get(Product, id) do
      nil -> {:error, :not_found}
      product -> {:ok, product}
    end
  end

  @doc """
  获取单个产品。

  如果 Product 不存在则引发 `Ecto.NoResultsError`。

  ## 示例

      iex> get_product!(123)
      %Product{}

      iex> get_product!(456)
      ** (Ecto.NoResultsError)

  """
  def get_product!(id), do: Repo.get!(Product, id)

  @doc """
  创建产品。

  ## 示例

      iex> create_product(%{field: value})
      {:ok, %Product{}}

      iex> create_product(%{field: bad_value})
      {:error, %Ecto.Changeset{}}

  """
  def create_product(attrs \\ %{}) do
    %Product{}
    |> Product.changeset(attrs)
    |> Repo.insert()
    |> maybe_broadcast_change(:product_created)
  end

  @doc """
  更新产品。

  ## 示例

      iex> update_product(product, %{field: new_value})
      {:ok, %Product{}}

      iex> update_product(product, %{field: bad_value})
      {:error, %Ecto.Changeset{}}

  """
  def update_product(%Product{} = product, attrs) do
    product
    |> Product.changeset(attrs)
    |> Repo.update()
    |> maybe_broadcast_change(:product_updated)
  end

  @doc """
  删除产品。

  ## 示例

      iex> delete_product(product)
      {:ok, %Product{}}

      iex> delete_product(product)
      {:error, %Ecto.Changeset{}}

  """
  def delete_product(%Product{} = product) do
    Repo.delete(product)
    |> maybe_broadcast_change(:product_deleted)
  end

  # PubSub 广播用于实时更新
  defp maybe_broadcast_change({:ok, product}, event) do
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "products",
      {event, product}
    )

    {:ok, product}
  end

  defp maybe_broadcast_change(error, _event), do: error
end
```

### 高级查询函数

```elixir
@doc """
返回低库存水平的产品。
"""
def list_low_stock_products(threshold \\ 10) do
  Product
  |> where([p], p.quantity <= ^threshold)
  |> where([p], p.active == true)
  |> order_by([p], asc: p.quantity)
  |> Repo.all()
end

@doc """
使用全文搜索搜索产品（PostgreSQL）。
"""
def search_products(search_term) when is_binary(search_term) do
  from(p in Product,
    where: fragment("? % ?", p.name, ^search_term),
    order_by: fragment("similarity(?, ?) DESC", p.name, ^search_term),
    limit: 20
  )
  |> Repo.all()
end

@doc """
返回分页产品。
"""
def paginate_products(page \\ 1, per_page \\ 20) do
  offset = (page - 1) * per_page

  products =
    Product
    |> limit(^per_page)
    |> offset(^offset)
    |> Repo.all()

  total_count = Repo.aggregate(Product, :count)

  %{
    entries: products,
    page_number: page,
    page_size: per_page,
    total_entries: total_count,
    total_pages: ceil(total_count / per_page)
  }
end
```

## 步骤 6：编写测试

**持续时间**：1-2 小时

参见 `write-tests.md` 获取全面的测试指南。

```elixir
# test/my_app/catalog_test.exs
defmodule MyApp.CatalogTest do
  use MyApp.DataCase

  alias MyApp.Catalog

  describe "list_products/1" do
    test "returns all products" do
      product = product_fixture()
      assert Catalog.list_products() == [product]
    end

    test "filters by active status" do
      active = product_fixture(active: true)
      _inactive = product_fixture(active: false)

      assert Catalog.list_products(filters: %{active: true}) == [active]
    end

    test "preloads associations" do
      product = product_fixture()
      [loaded] = Catalog.list_products(preload: [:category])

      assert Ecto.assoc_loaded?(loaded.category)
    end
  end

  # ... 更多测试
end
```

## 步骤 7：运行迁移

```bash
# 开发
mix ecto.migrate

# 测试
MIX_ENV=test mix ecto.migrate
```

## 上下文最佳实践

### 保持上下文专注

✅ **好：** 小而专注的上下文
```
Accounts (用户、认证)
Catalog (产品、类别)
Orders (订单、订单项、支付)
```

❌ **坏：** 上帝上下文
```
Store (用户、产品、订单、支付、配送、评论等)
```

### 公共和私有函数

```elixir
# 公共 API - 有文档且稳定
def list_products(opts \\ [])
def get_product(id)
def create_product(attrs)

# 私有助手 - 可以自由更改
defp apply_filters(query, filters)
defp maybe_broadcast_change(result, event)
```

### 永远不要在公共 API 中暴露 Ecto.Changeset

✅ **好：**
```elixir
@spec create_product(map()) :: {:ok, Product.t()} | {:error, Ecto.Changeset.t()}
def create_product(attrs)
```

❌ **坏：**
```elixir
# 不要暴露 changeset 创建
def product_changeset(attrs)
```

### 避免跨上下文数据库查询

✅ **好：** 调用其他上下文的公共 API
```elixir
def list_user_products(user_id) do
  # 不要直接查询 Accounts.User
  {:ok, user} = Accounts.get_user(user_id)

  Product
  |> where(seller_id: ^user.id)
  |> Repo.all()
end
```

❌ **坏：** 访问其他上下文的表
```elixir
def list_user_products(user_id) do
  # 不要这样做 - 违反上下文边界！
  from(p in Product,
    join: u in Accounts.User,
    where: u.id == ^user_id and p.seller_id == u.id
  )
  |> Repo.all()
end
```

## 常见陷阱

### 缺少预加载（N+1 查询）

```elixir
# 坏：N+1 查询
products = Catalog.list_products()
Enum.each(products, fn product ->
  IO.puts product.category.name  # 为每个产品查询！
end)

# 好：预加载关联
products = Catalog.list_products(preload: [:category])
Enum.each(products, fn product ->
  IO.puts product.category.name  # 已加载
end)
```

### 不使用数据库约束

```elixir
# 在迁移中 - 添加数据库约束
create constraint(:products, :price_must_be_positive, check: "price > 0")

# 在模式中 - 处理约束违规
def changeset(product, attrs) do
  product
  |> cast(attrs, [:price])
  |> validate_number(:price, greater_than: 0)
  |> check_constraint(:price, name: :price_must_be_positive,
       message: "必须大于 0")
end
```

## 检查清单

完成前标记：

- [ ] 上下文有清晰、单一的职责
- [ ] 模式有所有必需字段和关联
- [ ] 迁移包含索引和约束
- [ ] Changeset 验证全面
- [ ] 公共 API 函数用 @doc 文档化
- [ ] 返回类型用 @spec 文档化（或 dialyzer 正确推断）
- [ ] 所有函数返回 `{:ok, result}` 或 `{:error, reason}` 元组
- [ ] 测试覆盖 CRUD 操作
- [ ] 测试覆盖验证和约束
- [ ] 测试覆盖边缘情况
- [ ] 迁移成功运行
- [ ] 常见用例中无 N+1 查询
- [ ] 上下文遵循代码库中的现有模式

## 下一步

创建上下文后：
1. 运行 `mix test` 验证所有测试通过
2. 运行 `mix ecto.migrate` 应用迁移
3. 更新路由器/控制器以使用新上下文
4. 在 README 或指南中记录上下文 API
5. 考虑添加到项目的架构文档中
