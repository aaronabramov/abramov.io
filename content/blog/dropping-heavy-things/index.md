---
slug: rust-dropping-things-in-another-thread
date: 2020-03-21
title: 'Rust: Dropping heavy things in another thread can make your code 10000 times faster'
description: 'When working on Rust applications or CLIs that need to show something to the end user as fast as possible I often find that a significant chunk of the time is usually spent not in doing any computations, but in dropping large data structures at the end of the function. We can optimize it by deferring the `drop` and moving it to another thread...'
banner: './banner.png'
published: true
---



When working on Rust applications or CLIs that need to show something to the end user as fast as possible I often find that a significant chunk of the time is usually spent not in doing any computations, but in dropping large data structures at the end of the function.

For example, imagine you have a function that takes some heavy object and returns its size:
```
fn get_size(a: HeavyThing) -> usize {
    a.size()
}
```

Getting `size` of `a` is extremely cheap. it takes about `0.01ms`, hovever the entire function can take `1000ms` before it returns anything.
This is because Rust needs to drop the value of `a` and deallocate all memory it was using before it can return.

```
fn get_size(a: HeavyThing) -> usize {
    a.size()
    // <--- a is dropped here
}
```
 
And if `HeavyThing` is a very complex data structure it might take a while to deallocate all memory it's using.

This really sucks for UIs and interactive CLIs. We got all needed data to respond to a user request, but can't do anything with it until we finish cleaning things up!


One of the workarounds for this problem is to defer dropping the value by moving it to another thread and letting it take care of it.


```
fn get_size(a: HeavyThing) -> usize {
    let size = a.size();
    std::thread::spawn(move || drop(a));
    size
}
```

In this example, we spawn another thread, move the heavy data structure into it and forget about it. The function returns right away and somewhere in the future the other thread will start doing the work of dropping the data structure and deallocating all used memory.


Here's a small example of working with a `HashMap<usize, Vec<usize>>` data structure that has 1M keys.
The function that defers dropping to another therad runs about 10K times faster since the only work it needs to do is to start a new thread

```
// drop in another thread 0.087471ms
// drop in this thread 907.369466ms
```

https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=e6036d23879b0d0abda5196dfa8a131e

```
use std::collections::HashMap;
use std::thread;
use std::time::Instant;

const NUM_ELEMENTS: usize = 1000000;

type HeavyThings = HashMap<usize, Vec<usize>>;

fn main() {
    let heavy_things_1 = make_heavy_things();
    let heavy_things_2 = make_heavy_things();

    let len =log_time("drop in another thread", || {
        fn_that_drops_heavy_things_in_another_thread(heavy_things_2)
    });
    assert_eq!(len, NUM_ELEMENTS);

    let len = log_time("drop in this thread", || {
        fn_that_drops_heavy_things(heavy_things_1)
    });
    assert_eq!(len, NUM_ELEMENTS);
}

fn make_heavy_things() -> HeavyThings {
    (1..=NUM_ELEMENTS).map(|v| (v, vec![v])).collect()
}

fn fn_that_drops_heavy_things(things: HeavyThings) -> usize {
    things.len()
}

fn fn_that_drops_heavy_things_in_another_thread(things: HeavyThings) -> usize {
    let len = things.len();
    thread::spawn(move || drop(things));
    len
}

fn log_time<T, F: FnOnce() -> T>(name: &str, f: F) -> T {
    let time = Instant::now();
    let result = f();
    println!("{} {:?}", name, time.elapsed());
    result
}
```
