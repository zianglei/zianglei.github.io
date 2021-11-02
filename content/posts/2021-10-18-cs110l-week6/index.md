---
title: "CS110L Week 6"
date: 2021-10-18T16:20:21+08:00
description: 简易 parallel_map
toc: true
tags:
- CS110L
- Rust
---

实现 parallel_map，使用多线程在向量上执行函数 F，返回所有执行结果

## 设计思路

分为主线程和工作线程两部分，主线程负责将输入的元素通过通道发送给工作线程，工作线程需要将执行结果返回给主线程，主线程负责将结果放入结果向量中，因此需要两个 channel。

由于需要将每个元素的执行结果按照原有的输入顺序返回，在给工作线程传递参数的时候需要将元素在向量中的位置一并发送。

注意 crossbeam_channel 在一端**所有**引用断开连接之后，另一段的 send/recv 才会跳出循环，因此注意找到所有的引用。在此处实现中，工作线程的输出端在主线程也会有一个引用，应该在所有工作线程创建完成后将该引用 drop 掉。

## 代码

```rust
fn parallel_map<T, U, F>(mut input_vec: Vec<T>, num_threads: usize, f: F) -> Vec<U>
where
    F: FnOnce(T) -> U + Send + Copy + 'static,
    T: Send + 'static,
    U: Send + 'static + Default,
{
    let mut output_vec: Vec<U> = Vec::with_capacity(input_vec.len());
    // TODO: implement parallel map!
    let (input_tx, input_rx) = crossbeam_channel::unbounded::<Data<T>>();
    let (output_tx, output_rx) = crossbeam_channel::unbounded::<Data<U>>();
    let mut threads = Vec::new();

    for _ in 0..num_threads {
        let input_rx = input_rx.clone();
        let output_tx = output_tx.clone();
        threads.push(
            thread::spawn(move || {
                while let Ok(received) = input_rx.recv() {
                    let output: Data<U> = Data { data: f(received.data), index: received.index };
                    output_tx.send(output).unwrap();
                }
                drop(output_tx);
            })
        );
    }

    drop(output_tx);

    for (index, data) in input_vec.into_iter().enumerate() {
        input_tx.send(Data { data, index }).unwrap();
    }

    drop(input_tx);
    
    while let Ok(received) = output_rx.recv() {
        if output_vec.len() <= received.index {
            let len = output_vec.len();
            for _ in 0..(received.index - len + 1) {
                output_vec.push(U::default());
            }
        }
        output_vec[received.index] = received.data;
    }

    for handle in threads {
        handle.join().expect("Panic occurs in a thread!");
    }

    output_vec
}
```



