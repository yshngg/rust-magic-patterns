# 深入浅出 Rust 迭代器原理

<details>
  <summary>支持语言</summary>
  <ul>
    <li>
      <a href='https://github.com/alexpusch/rust-magic-patterns/tree/master/dumbing-down-iterator'>English</a> - <a href="https://github.com/alexpusch">@alexpusch</a>
    </li>
  </ul>
</details>

Rust 迭代器 API 是新手在掌握语言基础后需要学习的重要内容，但其文档对初学者可能有些晦涩。

例如查看 [map](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map) 方法文档时：

```rust
fn map<B, F>(self, f: F) -> Map<Self, F>
where
    Self: Sized,
    F: FnMut(Self::Item) -> B,
```

B 和 F 代表什么？为何返回 `Map` 类型？

再看 `collect` 方法，它允许我们将迭代器转换为各种类型的 `Collection`：

```rust
let v: Vec<_> = (0..10).collect();
let s: HashSet<_> = (0..10).collect();
```
这怎么可能？Rust 不是静态类型语言吗？同一个方法怎么能返回不同类型？

让我们通过简化实现来解析迭代器的类型系统原理。

## 迭代器特征（Iterator Trait）

[Iterator trait](https://doc.rust-lang.org/std/iter/trait.Iterator.html) 是迭代器 API 的基础：

```rust
pub trait MyIterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

该特性定义了迭代的基本机制。它有一个名为 `next` 的单一方法，返回迭代中的下一项，如果迭代结束则返回 `None`。[关联类型 (Associated type)](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types) `Item` 定义了迭代中项的类型。

## 示例 - 切片迭代器

让我们为一个简单的切片 - `[T]` 实现一个简单的迭代器。这是 [std::slice::Iter](https://doc.rust-lang.org/std/slice/struct.Iter.html) 的简化版：

```rust
/// SliceIterator 结构体持有对向量的引用以及迭代中的当前位置。
/// 生命周期参数`'a`确保迭代器不超越数据存在期
pub struct SliceIterator<'a, T> {
    data: &'a [T],
    pos: usize,
}

impl<'a, T> SliceIterator<'a, T> {
    //  pub(crate) 用于使此构造函数仅对本 crate 可见
    pub(crate) fn new(data: &'a [T]) -> Self {
        SliceIterator { data, pos: 0 }
    }
}

impl<'a, T> MyIterator for SliceIterator<'a, T> {
    /// SliceIterator 的 Item 类型是对切片类型的引用。
    /// 生命周期参数 `'a` 确保返回的引用不会超过我们正在迭代的数据的生存期。
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        if self.pos >= self.data.len() {
            None
        } else {
            let result = Some(&self.data[self.pos]);
            self.pos += 1;
            result
        }
    }
}
```

测试用例：

```rust
#[test]
fn slice_iterator_next_returns_next_item() {
    let mut iter = SliceIterator::new(&[1, 2, 3]);
    assert_eq!(iter.next(), Some(&1));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), None);
}
```

**生命周期参数 `'a` **确保迭代器不会超过数据存在期。这很重要，因为迭代器持有对切片的引用，如果切片在迭代器之前被丢弃，迭代器将持有一个悬垂引用 (dangling reference)。我们不希望这种情况发生。

多亏了这个生命周期参数，以下代码将无法编译：

```rust
let data = vec![1, 2, 3];                 // ──| 数据生命周期 'a 开始
let mut iter = SliceIterator::new(&data); // SliceIterator::new 绑定到 'a 生命周期
drop(data);                               // __| 数据被释放，生命周期结束
iter.next();                              // 此处违反 SliceIterator 的 'a 约束
```

## 迭代器方法

[Iterator trait](https://doc.rust-lang.org/std/iter/trait.Iterator.html) 有一个必需的 `next` 方法，但它也提供了许多其他方法。例如，之前提到的 `map` 方法。

扩展 `MyIterator` 特征实现 `map` 和 `filter`：

```rust
pub trait MyIterator {
    type Item;

    // ...

    fn map<B, F>(self, map_fn: F) -> MyMap<Self, F>
    where
        Self: Sized,
        F: FnMut(Self::Item) -> B,
    {
        MyMap::new(self, map_fn)
    }

    fn filter<P>(self, filter_fn: P) -> MyFilter<Self, P>
    where
        Self: Sized,
        P: FnMut(&Self::Item) -> bool,
    {
        MyFilter::new(self, filter_fn)
    }
}
```

### 映射适配器（Map Adapter）

简化版 [std::iter::Map](https://doc.rust-lang.org/std/iter/struct.Map.html) 实现：

```rust
/// I 是应用了 map 方法的迭代器类型
/// F 是用于映射元素的闭包类型。在 MyIterator 实现中，它被约束为类型 `FnMut(Self::Item) -> B`
pub struct MyMap<I, F>
where
    I: MyIterator,
{
    iter: I,
    map_fn: F,
}

impl<I, F> MyMap<I, F>
where
    I: MyIterator,
{
    pub(crate) fn new(iter: I, map_fn: F) -> Self {
        MyMap { iter, map_fn }
    }
}

impl<B, I, F> MyIterator for MyMap<I, F>
where
    I: MyIterator,
    F: FnMut(I::Item) -> B,
{
    // MyMap 的 Item 类型是由 map 闭包返回的类型
    type Item = B;

    fn next(&mut self) -> Option<B> {
        if let Some(x) = self.iter.next() {
            Some((self.map_fn)(x))
        } else {
            None
        }
    }
}
```

### 过滤适配器（Filter Adapter）

简化版 [std::iter::Filter](https://doc.rust-lang.org/std/iter/struct.Filter.html)：

```rust
/// I 是应用 filter 方法的迭代器类型
/// P 是我们对每个项调用的谓词函数的类型
pub struct MyFilter<I, P>
where
    I: MyIterator,
{
    iter: I,
    filter_fn: P,
}

impl<I, P> MyFilter<I, P>
where
    I: MyIterator,
{
    pub(crate) fn new(iter: I, filter_fn: P) -> Self {
        MyFilter { iter, filter_fn }
    }
}

impl<I, P> MyIterator for MyFilter<I, P>
where
    I: MyIterator,
    P: FnMut(&I::Item) -> bool,
{
    /// MyFilter 的 Item 类型与内部迭代器的 Item 类型相同
    type Item = I::Item;

    fn next(&mut self) -> Option<Self::Item> {
        // 我们遍历内部迭代器，直到找到一个通过筛选条件的项
        while let Some(x) = self.iter.next() {
            if (self.filter_fn)(&x) {
                return Some(x);
            }
        }

        None
    }
}
```

像 `Map` 和 `Filter` 这样的适配器是迭代器内部实现的，因此你很少会直接使用它们，尽管如此，让我们看一个使用示例：

```rust
#[test]
fn my_map_filter_next_returns_next_item() {
    let iter = SliceIterator::new(&[1, 2, 3]);
     注意谓词闭包是对引用的引用。查看代码以了解原因！
    let filter = MyFilter::new(iter, |x: &&i32| **x % 2 == 0);
    let mut map = MyMap::new(filter, |x| x * 2);

    assert_eq!(map.next(), Some(4));
    assert_eq!(map.next(), None);
}
```

## 收集方法（Collect Method）

大多数迭代器用法最终都会调用 [`collect`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect) 方法，以便将迭代项收集到具体类型的集合中。我们也来深入了解一下这个方法的内部机制。

首先，我们扩展 `MyIterator` 特性以包含 `collect` 方法：

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

`collect` 方法定义了一个返回位置泛型类型B，该类型必须实现 `MyFromIterator` 特征。

## FromIterator 特征（FromIterator Trait）

[`std::iter::FromIterator`](https://doc.rust-lang.org/std/iter/trait.FromIterator.html) 是一个特征，定义了如何从迭代器创建集合。之前我们定义了 SliceIterator 结构体，它能将向量（vector）转换为迭代器。现在我们需要定义相反的操作 -- 将迭代器转换为集合的方法。

```rust
pub trait MyFromIterator<T> {
    fn my_from_iter<I>(iter: I) -> Self
    where
        I: MyIterator<Item = T>;
}
```

`my_from_iter` 允许我们将任何实现了 `MyIterator` 的类型转换为实现了该特征的集合。我们可以利用这一点将 `SliceIterator`、`MyMap`、`MyFilter` 以及其他 `MyIterator` 转换为集合。

请注意，为了简化，此处没有使用实际 `std::iter::FromIterator` 特征中的 `IntoIterator` 特征。

例如，让我们为 `Vec<T>` 和 `HashSet<T>` 实现 `MyFromIterator`。这两个示例都会遍历目标迭代器，并将每个元素推入构建的集合（Collection）中。

```rust
impl<T> MyFromIterator<T> for Vec<T> {
    fn my_from_iter<I>(mut iter: I) -> Self
    where
        I: MyIterator<Item = T>
    {
        let mut vec = Vec::new();

        while let Some(x) = iter.next() {
            vec.push(x);
        }

        vec
    }
}

impl<T> MyFromIterator<T> for HashSet<T>
where
    // 我们需要添加这些约束，因为 HashSet::new() 需要它们
    T: Eq + Hash,
{
    fn my_from_iter<I>(mut iter: I) -> Self
    where
        I: MyIterator<Item = T>,
    {
        let mut set = HashSet::new();
        while let Some(x) = iter.next() {
            set.insert(x);
        }

        set
    }
}
```

让我们再看一下 `collect` 使用示例：

```rust
let iter = SliceIterator::new(&[1, 2, 3]);
let v: Vec<_> = iter.collect();
```

使用我们简化的实现，它实际上扩展为

```rust
fn collect(self) -> Vec<u64>
{
    Vec::<u64>::my_from_iter(self)
}
```

更具体地

```rust
Vec::<u64>::my_from_iter(iter);
```

任何实现了 `MyFromIterator` 的其他集合类型都可以以同样的方式使用：

```rust
let s: HashSet<_> = iter.collect();
let b: BTreeSet<_> = iter.collect();
let l: LinkedList<_> = iter.collect();
```

这些 `collect` 调用中的每一个实际上都扩展为不同的 `my_from_iter` 实现。

```rust
let s = HashSet::<_>::my_from_iter(iter);
let b = BTreeSet::<_>::my_from_iter(iter);
let l = LinkedList::<_>::my_from_iter(iter);
```

现在我们知道了，`collect` 只是对 `FromIterator` 和 `from_iter` 的一层薄薄的语法糖，实际上并未违反 Rust 严格的类型系统。

## 扩展学习

请务必查阅 [std::iter](https://doc.rust-lang.org/std/iter/index.html) 文档以了解所有其他迭代器方法和适配器。

本文的所有源代码都可以在 [src](./src/my_iterator.rs) 目录中找到。
