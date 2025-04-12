# Rust Axum 风格魔法函数参数实现解析

<details>
  <summary>支持语言</summary>
  <ul>
    <li>
      <a href='https://github.com/alexpusch/rust-magic-patterns/tree/master/axum-style-magic-function-param'>English</a> - <a href="https://github.com/alexpusch">@alexpusch</a>
    </li>
  </ul>
</details>

初学 Rust 时，我接触到的是一门严格的静态类型语言，它没有函数重载或可选参数等特性。

但当我发现 [Axum](https://github.com/tokio-rs/axum) 框架时，这样的代码让我感到惊奇：

```rust
let app = Router::new()
  .route("/users", get(get_users))
  .route("/products", get(get_product));

async fn get_users(Query(params): Query<Params>) -> impl IntoResponse {
    let users = /* ... */

    Json(users)
}

async fn get_product(State(db): State<Db>, Json(payload): Json<Payload>) -> String {
  let product = /* ... */

  product.to_string()
}
```

`get`方法可以接收不同类型函数的指针！这是怎么实现的？🤯 我创建了简化版本来解析这个魔法。

```rust
fn print_id(id: Id) {
    println!("id is {}", id.0);
}

// Param(param) 只是模式匹配
fn print_all(Param(param): Param, Id(id): Id) {
    println!("param is {param}, id is {id}");
}

pub fn main() {
    let context = Context::new("magic".into(), 33);

    trigger(context.clone(), print_id);
    trigger(context.clone(), print_all);
}
```

示例中 `trigger` 方法接收 `Context` 对象和函数指针，函数指针可能接收 1 个或 2 个参数（ `Id` 或 `Param` 类型）。魔法何在？

## 核心组件解析

### 上下文对象（Context）

```rust
struct Context {
    param: String,
    id: u32,
}
```

`Context` 类似 Axum 中的 `Request`，是参数的来源。本例包含两个数据字段。

### FromContext 特征（Trait）

```rust
trait FromContext {
    fn from_context(context: &Context) -> Self;
}
```

第一个技巧是 `FromContext` 特征，允许创建从 `Context` 提取数据的 **Extractor**。例如：

```rust
pub struct Param(pub String);

impl FromContext for Param {
    fn from_context(context: &Context) -> Self {
        Param(context.param.clone())
    }
}
```

该特征使我们能够将 `Context` 转换为函数需要的 `Param` 参数。更多内容稍后介绍。

### Handler 特征（Trait）

```rust
trait Handler<T> {
    fn call(self, context: Context);
}
```

第二个技巧是 `Handler` 特征。我们为[闭包类型](https://doc.rust-lang.org/reference/types/closure.html) Fn(T) 实现该特征。是的，我们可以实现闭包类型的特征。此实现将使我们能够在函数调用及其参数之间具有**中间件 (middleware)**。在这里，我们将调用 `FromContext::from_context` 方法，将 `Context` 转换为预期函数参数，即`Param` 或 `Id`。

```rust
impl<F, T> Handler<T> for F
where
    F: Fn(T),
    T: FromContext,
{
    fn call(self, context: Context) {
        (self)(T::from_context(&context));
    }
}
```

支持多参数的实现：

```rust
impl<T1, T2, F> Handler<(T1, T2)> for F
where
    F: Fn(T1, T2),
    T1: FromContext,
    T2: FromContext,
{
    fn call(self, context: Context) {
        (self)(T1::from_context(&context), T2::from_context(&context));
    }
}
```

该实现不关心参数顺序，支持 `fn foo(p: Param, id: Id)` 和 `fn foo(id: Id, p: Param)` 两种形式。

### 整合实现

`trigger` 函数的实现现在变得简单直接：

```rust
pub fn trigger<T, H>(context: Context, handler: H)
where
    H: Handler<T>,
{
    handler.call(context);
}
```

解析调用过程：

```rust
let context = Context::new("magic".into(), 33);
trigger(context.clone(), print_id);
```

1. `print_id` 类型为 `Fn(Id)`，对应 `Handler<Id>` 实现
2. 调用 `Handler::call` 方法，通过 `Id::from_context(context)` 获取参数
3. 使用转换后的参数调用 `print_id`

魔法揭秘。
