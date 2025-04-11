# 深入浅出 Rust 迭代器原理

<details>
  <summary>支持语言</summary>
  <ul>
    <li>
      <a href='https://github.com/alexpusch/rust-magic-patterns/tree/master/dumbing-down-iterator'>English</a> - <a href="https://github.com/alexpusch">@alexpusch</a>
    </li>
  </ul>
</details>

Rust 迭代器 API 是新手在掌握语言基础后需要学习的重要内容，但其文档对初学者可能有些晦涩。例如查看[map 方法文档](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map)时：

```rust
fn map<B, F>(self, f: F) -> Map<Self, F>
where
    Self: Sized,
    F: FnMut(Self::Item) -> B,
```

这些泛型参数代表什么？为何返回`Map`类型？再看`collect`方法如何实现不同类型收集：

```rust
let v: Vec<_> = (0..10).collect();
let s: HashSet<_> = (0..10).collect();
```

让我们通过简化实现来解析迭代器的类型系统原理。

## 迭代器特征（Iterator Trait）

[Iterator trait](https://doc.rust-lang.org/std/iter/trait.Iterator.html)是迭代器 API 的基础：

```rust
pub trait MyIterator {
    type Item;  // 关联类型（Associated Type）

    fn next(&mut self) -> Option<Self::Item>;
}
```

## 示例 - 切片迭代器

这是[std::slice::Iter](https://doc.rust-lang.org/std/slice/struct.Iter.html)的简化版：

```rust
/// 生命周期参数`'a`确保迭代器不超越数据存在期
pub struct SliceIterator<'a, T> {
    data: &'a [T],
    pos: usize,
}

impl<'a, T> MyIterator for SliceIterator<'a, T> {
    type Item = &'a T;  // 关联类型为切片元素的引用

    fn next(&mut self) -> Option<Self::Item> {
        // 迭代逻辑...
    }
}
```

测试用例：

```rust
#[test]
fn slice_iterator_next_returns_next_item() {
    let mut iter = SliceIterator::new(&[1, 2, 3]);
    assert_eq!(iter.next(), Some(&1));
    // ...
}
```

**生命周期参数`'a`**确保迭代器不会超过数据存在期，以下代码将无法编译：

```rust
let data = vec![1, 2, 3];          // ──| 数据生命周期'a开始
let mut iter = SliceIterator::new(&data);
drop(data);                        // __| 数据被释放，生命周期结束
iter.next();                       // 此处违反SliceIterator的'a约束
```

## 迭代器方法

扩展`MyIterator`特征实现`map`和`filter`：

```rust
pub trait MyIterator {
    // ...

    fn map<B, F>(self, map_fn: F) -> MyMap<Self, F>
    where
        Self: Sized,
        F: FnMut(Self::Item) -> B,
    {
        MyMap::new(self, map_fn)  // 返回映射适配器
    }

    fn filter<P>(self, filter_fn: P) -> MyFilter<Self, P>
    where
        Self: Sized,
        P: FnMut(&Self::Item) -> bool,
    {
        MyFilter::new(self, filter_fn)  // 返回过滤适配器
    }
}
```

### 映射适配器（Map Adapter）

简化版[std::iter::Map](https://doc.rust-lang.org/std/iter/struct.Map.html)实现：

```rust
pub struct MyMap<I, F>
where
    I: MyIterator,
{
    iter: I,     // 原始迭代器
    map_fn: F,   // 映射闭包
}

impl<B, I, F> MyIterator for MyMap<I, F>
where
    I: MyIterator,
    F: FnMut(I::Item) -> B,
{
    type Item = B;

    fn next(&mut self) -> Option<B> {
        self.iter.next().map(&mut self.map_fn)  // 应用映射函数
    }
}
```

### 过滤适配器（Filter Adapter）

简化版[std::iter::Filter](https://doc.rust-lang.org/std/iter/struct.Filter.html)：

```rust
pub struct MyFilter<I, P>
where
    I: MyIterator,
{
    iter: I,
    filter_fn: P,  // 过滤谓词
}

impl<I, P> MyIterator for MyFilter<I, P>
where
    I: MyIterator,
    P: FnMut(&I::Item) -> bool,
{
    type Item = I::Item;

    fn next(&mut self) -> Option<Self::Item> {
        // 跳过不满足条件的元素
        while let Some(x) = self.iter.next() {
            if (self.filter_fn)(&x) {
                return Some(x);
            }
        }
        None
    }
}
```

测试用例：

```rust
#[test]
fn my_map_filter_next_returns_next_item() {
    let iter = SliceIterator::new(&[1, 2, 3]);
    let filter = MyFilter::new(iter, |x: &&i32| **x % 2 == 0);
    let mut map = MyMap::new(filter, |x| x * 2);
    // 断言结果...
}
```

## 收集方法（Collect Method）

扩展`collect`方法实现：

```rust
pub trait MyIterator {
    // ...

    fn collect<B>(self) -> B
    where
        B: MyFromIterator<Self::Item>,  // 需实现收集特征
        Self: Sized,
    {
       B::my_from_iter(self)
    }
}
```

## 收集特征（FromIterator Trait）

[std::iter::FromIterator](https://doc.rust-lang.org/std/iter/trait.FromIterator.html)的简化实现：

```rust
pub trait MyFromIterator<T> {
    fn my_from_iter<I>(iter: I) -> Self
    where
        I: MyIterator<Item = T>;
}
```

为`Vec<T>`和`HashSet<T>`实现：

```rust
impl<T> MyFromIterator<T> for Vec<T> {
    fn my_from_iter<I>(mut iter: I) -> Self {
        let mut vec = Vec::new();
        while let Some(x) = iter.next() {
            vec.push(x);  // 收集元素到向量
        }
        vec
    }
}

impl<T> MyFromIterator<T> for HashSet<T>
where
    T: Eq + Hash,
{
    fn my_from_iter<I>(mut iter: I) -> Self {
        let mut set = HashSet::new();
        while let Some(x) = iter.next() {
            set.insert(x);  // 插入哈希集合
        }
        set
    }
}
```

使用示例：

```rust
let iter = SliceIterator::new(&[1, 2, 3]);
let v: Vec<_> = iter.collect();  // 调用Vec::my_from_iter
let s: HashSet<_> = iter.collect();  // 调用HashSet::my_from_iter
```

## 扩展学习

建议阅读[std::iter 官方文档](https://doc.rust-lang.org/std/iter/index.html)了解完整实现，本文完整代码参见[src 目录](./src/my_iterator.rs)。
