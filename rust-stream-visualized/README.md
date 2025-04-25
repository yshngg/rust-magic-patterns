# Rust Stream API 可视化解析

<details>
  <summary>支持语言</summary>
  <ul>
    <li>
      <a href='https://github.com/alexpusch/rust-magic-patterns/tree/master/rust-stream-visualized'>English</a> - <a href="https://github.com/alexpusch">@alexpusch</a>
    </li>
  </ul>
</details>

在真实应用中管理并发（Concurrency）颇具挑战。Rust 通过 async/await 机制和[Stream API](https://docs.rs/futures/latest/futures/stream/index.html)提供了优雅的解决方案。但优雅有时掩盖了复杂性——我们能否直观理解流式管道的并行行为和顺序？本文通过可视化实验揭示 Stream API 的运行时特性。

## Stream API 概览

```rust
async fn async_work(i32) -> i32 {...}
async fn async_predicate(i32) -> Option<i32> {...}

async fn buffered_filter_example() {
    let stream = stream::iter(0..10)
        .map(async_work)  // 生成异步任务流
        .buffered(3)      // 最大3个任务并发执行
        .filter_map(async_predicate); // 异步过滤

    pin!(stream);  // 固定流以便消费

    while let Some(next) = stream.next().await {
        println!("处理完成: {}", next);
    }
}
```

关键概念：

- `buffered(n)`：保持顺序的缓冲执行
- `filter_map`：异步过滤转换
- `pin!`：固定流的内存位置

## 实验工具 - 可视化实现

使用[Bevy 引擎](https://bevyengine.org/)实时可视化流式处理过程，每个工作单元通过进度条展示执行状态：

<p align="center">
    <img src="./resources/buffer_1.gif">
</p>

真实代码通过事件通道报告进度，完整代码参见[GitHub 仓库](./rust-stream-vis)。

## 实验 1：buffered 方法

```rust
stream::iter(0..10)
    .map(async_work)
    .buffered(5);  // 最大5个并发
```

<p align="center">
    <img src="./resources/buffer_5.gif">
</p>

实验结果：

- 严格按顺序缓冲结果
- 前序任务完成后才获取新任务
- 内存使用可控但吞吐量受限

## 实验 2：buffer_unordered 方法

```rust
stream::iter(0..10)
    .map(async_work)
    .buffer_unordered(5);  // 无序缓冲
```

<p align="center">
    <img src="./resources/buffer_unordered_5.gif">
</p>

实验结果：

- 按完成顺序输出结果
- 任意任务完成立即获取新任务
- 吞吐量更高但内存使用波动

## 实验 3：filter_map 方法

```rust
stream::iter(0..10)
    .filter_map(async_predicate);  // 异步过滤
```

<p align="center">
    <img src="./resources/filter.gif">
</p>

实验结果：

- 串行执行过滤逻辑
- 可通过`map().buffered().filter_map(future::ready)`实现并发过滤

## 实验 4：buffered + filter_map 组合

```rust
stream::iter(0..10)
    .map(async_work)
    .buffered(3)
    .filter_map(async_predicate);  // 长耗时过滤
```

<p align="center">
    <img src="./resources/buffer_filter_long.gif">
</p>

意外发现：

1. 过滤阶段会阻塞上游任务执行
2. 初始批次完成后才会获取新任务
3. 替换为`buffer_unordered`行为相同

## 原理剖析

通过分析[Stream 实现源码](https://github.com/rust-lang/futures-rs)发现：

- 流式管道通过适配器（Adapter）链式组合
- `buffered`使用`FuturesOrdered`管理并发
- 每个`poll_next`调用驱动管道执行
- 下游阶段会阻塞上游 future 的 poll 调用

伪代码示意：

```rust
loop {
    // 填充缓冲队列
    while queue.len() < limit {
        queue.push(source.next())
    }

    // 并发执行缓冲任务
    let result = queue.poll_next().await;

    // 执行过滤（阻塞上游poll）
    let filtered = async_predicate(result).await;
}
```

## 生产环境启示

1. 避免长耗时下游操作阻塞管道
2. 复杂管道建议使用任务（Task）和通道（Channel）组合
3. 关注 future 的 poll 频率（参见[Async Book](https://rust-lang.github.io/async-book/)）
4. 性能敏感场景考虑替代方案（如[tokio-stream](https://docs.rs/tokio-stream/)）

完整实验结论：

- Stream API 适合简单管道场景
- 复杂需求需谨慎评估执行特性
- 可视化工具帮助理解运行时行为

相关资源：

- [理解 Rust Future 底层原理](https://fasterthanli.me/articles/understanding-rust-futures-by-going-way-too-deep)
- [Barbara 的并发难题](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_battles_buffered_streams.html)
