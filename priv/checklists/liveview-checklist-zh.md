# LiveView 最佳实践检查清单

在实现 Phoenix LiveView 功能时使用此检查清单，以确保最佳实践并避免常见陷阱。

## 生命周期实现

### Mount
- [ ] 处理连接和断开连接状态
- [ ] 仅在 `connected?(socket)` 时订阅 PubSub
- [ ] 高效加载初始数据
- [ ] 设置所有必需的 assigns
- [ ] 为集合使用流（而不是常规 assigns）
- [ ] 返回 `{:ok, socket}` 或 `{:ok, socket, options}`

### Handle Event
- [ ] 所有交互元素都有 phx-* 属性
- [ ] 事件名称具有描述性（"save"、"delete"，而不是 "click"）
- [ ] 使用 `phx-value-*` 传递事件数据
- [ ] 更新后返回 `{:noreply, socket}`
- [ ] 提供用户反馈（flash、UI 更新）
- [ ] 优雅处理错误

### Handle Info
- [ ] 正确处理 PubSub 消息
- [ ] 正确处理进程消息
- [ ] 根据消息更新 UI
- [ ] 返回 `{:noreply, socket}`

## Socket Assigns

### 最小化 Assigns
- [ ] 仅存储渲染所需的内容
- [ ] 为所有集合使用流
- [ ] 避免存储大型数据结构
- [ ] 不再需要时删除 assigns
- [ ] 为一次性数据使用临时 assigns

### 命名
- [ ] Assign 名称具有描述性
- [ ] 布尔 assign 以 `?` 结尾（例如，`loading?`、`empty?`）
- [ ] LiveViews 之间命名一致
- [ ] 无通用名称如 `data` 或 `info`

## 流

### 使用
- [ ] 所有集合使用流（不是 assigns）
- [ ] 容器有 `phx-update="stream"` 和唯一 `id`
- [ ] 流项目有唯一 `id` 属性：`<div id={id}>`
- [ ] 使用 `stream/3` 添加项目
- [ ] 使用 `stream_delete/3` 删除项目
- [ ] 使用 `stream_insert/4` 和 `at:` 进行定位
- [ ] 用 `stream/4` 和 `reset: true` 重置

### 空状态
- [ ] 优雅处理空集合
- [ ] 使用 Tailwind 的 `only:` 变体进行空状态消息
- [ ] 在没有项目时提供清晰消息

```elixir
<div id="products" phx-update="stream">
  <div class="hidden only:block">还没有产品。</div>
  <div :for={{id, product} <- @streams.products} id={id}>
    ...
  </div>
</div>
```

## 表单

### 设置
- [ ] 使用 `to_form/1` 创建表单结构
- [ ] 表单有唯一 `id` 属性
- [ ] 实现 `phx-change="validate"` 进行实时验证
- [ ] 实现 `phx-submit="save"` 进行提交
- [ ] 使用 core_components 中的 `<.input>` 组件

### 验证
- [ ] 用 `action: :validate` 在更改时验证
- [ ] 内联显示错误
- [ ] 处理过程中禁用提交按钮
- [ ] 成功提交后清除表单
- [ ] 提供清晰的成功/错误反馈

### 示例
```elixir
<.form
  for={@form}
  id="product-form"
  phx-change="validate"
  phx-submit="save"
>
  <.input field={@form[:name]} label="名称" />
  <.input field={@form[:price]} label="价格" type="number" />
  <.button phx-disable-with="保存中...">保存</.button>
</.form>
```

## 事件与交互

### 事件属性
- [ ] 使用 `phx-click` 进行点击
- [ ] 使用 `phx-submit` 进行表单提交
- [ ] 使用 `phx-change` 进行表单/输入更改
- [ ] 使用 `phx-value-*` 传递数据
- [ ] 使用 `phx-target` 进行组件事件
- [ ] 使用 `phx-debounce` 进行搜索输入

### 用户反馈
- [ ] 显示加载状态（`phx-disable-with`）
- [ ] 提供成功消息（flash 或内联）
- [ ] 清晰显示错误消息
- [ ] 在适当位置实现乐观 UI 更新
- [ ] 处理过程中禁用按钮

## PubSub 与实时

### 订阅
- [ ] 仅在 `connected?(socket)` 时订阅
- [ ] 主题名称具有描述性和作用域
- [ ] 断开连接时自动取消订阅（无需手动清理）
- [ ] 正确管理多个订阅

### 广播
- [ ] 在成功变异后广播
- [ ] 包含 UI 更新所需的所有数据
- [ ] 广播到正确的主题
- [ ] 优雅处理广播失败

### 示例
```elixir
def mount(_params, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "products")
  end

  {:ok, stream(socket, :products, list_products())}
end

def handle_info({:product_created, product}, socket) do
  {:noreply, stream_insert(socket, :products, product, at: 0)}
end
```

## 性能

### 优化
- [ ] 为集合使用流而不是 assigns
- [ ] 为大数据集实现分页
- [ ] 在搜索/过滤输入上使用 `phx-debounce`
- [ ] 最小化 socket assigns
- [ ] 避免在模板中进行昂贵计算
- [ ] 对外部 JS 使用 `phx-update="ignore"`

### 渲染
- [ ] 模板中无重计算
- [ ] 用 `:if` 属性进行条件渲染
- [ ] 循环使用 `:for` 属性（不是 Enum.map）
- [ ] 组件提取可重用标记
- [ ] 尽可能预计算 CSS 类

## 测试

### Mount 测试
- [ ] 测试初始渲染
- [ ] 测试数据加载
- [ ] 测试空状态
- [ ] 测试错误状态
- [ ] 测试不同用户权限

### 事件测试
- [ ] 测试所有 phx-click 处理器
- [ ] 测试表单提交
- [ ] 测试表单验证
- [ ] 测试删除/更新操作
- [ ] 测试边缘情况

### 集成测试
- [ ] 测试完整用户流程
- [ ] 测试实时更新
- [ ] 测试并发用户（如果适用）
- [ ] 测试 LiveViews 之间的导航

## 安全

### 授权
- [ ] 在 mount 中检查用户权限
- [ ] 在每个事件上验证访问
- [ ] assign 中不暴露未授权数据
- [ ] 无法访问其他用户数据
- [ ] 适当的租户隔离（如果多租户）

### 输入验证
- [ ] 验证所有事件参数
- [ ] 在 changeset 中验证表单数据
- [ ] 无原始 HTML 注入
- [ ] CSRF 保护已启用（Phoenix 中的默认值）

## 组件组织

### LiveComponents
- [ ] 仅在需要状态隔离时使用
- [ ] 有唯一 `id` 属性
- [ ] 通过 `send_update/2` 更新
- [ ] 与父级最小耦合

### 函数组件
- [ ] 无状态 UI 优先于 LiveComponents
- [ ] 用 `attr/3` 验证属性
- [ ] 用插槽进行灵活内容
- [ ] 在 LiveViews 之间可重用

## 常见模式

### 过滤
```elixir
def handle_event("filter", %{"filter" => filter}, socket) do
  products = list_products(filter)

  {:noreply,
   socket
   |> assign(:filter, filter)
   |> stream(:products, products, reset: true)}
end
```

### 分页
```elixir
def handle_event("load-more", _, socket) do
  page = socket.assigns.page + 1
  products = list_products(page: page)

  {:noreply,
   socket
   |> assign(:page, page)
   |> stream(:products, products)}
end
```

### 乐观 UI
```elixir
def handle_event("delete", %{"id" => id}, socket) do
  product = get_product!(id)

  # 乐观更新
  socket = stream_delete(socket, :products, product)

  # 异步删除
  Task.start(fn -> delete_product(product) end)

  {:noreply, socket}
end
```

## 常见陷阱

❌ **在 assigns 中存储集合**
```elixir
# 坏 - 在 socket 中存储整个列表
assign(socket, :products, list_products())

# 好 - 使用流
stream(socket, :products, list_products())
```

❌ **未连接时订阅**
```elixir
# 坏 - 即使未连接也订阅
def mount(_params, _session, socket) do
  Phoenix.PubSub.subscribe(MyApp.PubSub, "products")
  {:ok, socket}
end

# 好 - 仅在连接时订阅
def mount(_params, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "products")
  end
  {:ok, socket}
end
```

❌ **缺少 phx-update="stream"**
```elixir
# 坏 - 缺少 phx-update
<div id="products">  <!-- 缺少 phx-update="stream" -->
  <div :for={{id, p} <- @streams.products} id={id}>...</div>
</div>

# 好 - 有 phx-update
<div id="products" phx-update="stream">
  <div :for={{id, p} <- @streams.products} id={id}>...</div>
</div>
```

❌ **不使用 to_form**
```elixir
# 坏 - 直接使用 changeset
<.form for={@changeset}>

# 好 - 使用 to_form
<.form for={@form}>  <!-- 其中 @form = to_form(@changeset) -->
```

❌ **在模板中重计算**
```elixir
# 坏 - 模板中昂贵计算
<%= Enum.map(@products, &expensive_calculation/1) %>

# 好 - 在 handle_event/mount 中预计算
products_with_data = Enum.map(products, &expensive_calculation/1)
assign(socket, :products_with_data, products_with_data)
```

## 完成前

- [ ] 测试所有事件
- [ ] 实时更新工作
- [ ] 实现加载状态
- [ ] 完整错误处理
- [ ] 性能可接受
- [ ] 无内存泄漏（检查 socket assigns）
- [ ] 与多个并发用户工作
- [ ] 适当的授权检查
- [ ] 用户反馈清晰
- [ ] 处理边缘情况