+++
title = 'Understanding Concurrency'
date = "2024-09-06T09:54:48-03:00"
description = "Understanding concurrency in Rust."
tags = [
    "Rust",
    "Concurrency",
    "Multi-threading",
    "Operating Systems",
]
+++

When working with multi-threaded programs, concurrency issues can lead do **undefined behaviour** and unpredictable results. If you've ever encountered strange behaviour in your programs e.g. a shared counter not reflecting the expected value, you might stick to the end of this post! Here, I'll dive into the common pitfalls of concurrency.

## What's the Problem?
Concurrency arises when multiple threads execute simutaneously and interact with shared data. A core issue in multi-threaded programming is how threads access and modify shared variables. Without proper synchronization, this can lead to issues due to **race conditions**.

Consider this simple Rust program:

```rs
use std::thread;

static mut COUNTER: i32 = 0;
static LOOPS: i32 = 1000;

fn worker() {
    for _ in 0..LOOPS {
        unsafe {
            COUNTER += 1;
        }
    }
}

fn main() {
    let handles: Vec<_> = (0..2).map(|| {
        thread::spawn(|| {
            worker();
        })
    }).collect();

    for handle in handles {
        handles.join().unwrap();
    }

    unsafe {
        println!("Final value: {}", COUNTER);
    }
}
```

## What is going on there?
In this code, two threads are created, each of which increments a shared counter variable a specified number of times. We expect that if `LOOPS` is set to 1,000, the final value of `COUNTER` should be 2,000. However, if you run this program you may observe results like 1282 or 1473.

## Why do we get these results?
The issue lies in how the counter is updated. Each incrementation consists of three steps:
+ 1. Load the value of `COUNTER` from memory.
* 2. Increment the value.
* 3. Store the updated value back to memory.

These steps are not **atomic**, meaning they can be interrupted by another thread between any two of them. When two threads try to update `COUNTER` simultaneously, they might read the same initial value, increment it, and then both write back the same result!

Try to compile the code yourself, give `LOOP` some larger values, you'll notice that the final value of `COUNTER` is often less than expected and varies between runs. This is a direct consequence of **race condition**, an issue regarding threads attempting to use and mutate a common resource at the same time.

## What now?
Gladly our ancient computer sorcerers already solved that to us! Syncrhonization mechanisms are a thing now, and a very common of them are **Mutexes**!

A mutex ensures that only one thread can access the critical section - i.e. race condition prone section - at a time. So now, we can get our `worker` function and make it atomic:
```rs
use std::sync::{Arc, Mutex};
use stc::thread;

static LOOPS: i32 = 1000;

fn worker(counter: Arc<Mutex<i32>>) {
    for _ in 0..LOOPS {
        let mut num = counter.lock().unwrap();
        *num += 1;
    }
}

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let handles: Vec<_> = (0..2).map(|| {
        let counter = Arc::clone(&counter);
        thread::spawn(move || {
            worker(counter);
        })
    }).collect();

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Final value is: {}", *counter.lock().unwrap());
}
```

Try to run it this time, I bet a dollar that the output will be 2,000. Now for the explanation, by providing the code with `Arc` and `Mutex`, we assure that our shared counter is properly handled.
* `Arc` or atomically reference counted, enables many threads to share ownership over the same variable (`Mutex<i32>` in this case). It ensures that the data is kept alive as long as there are references to it, and then it gets cleaned up when no longer needed!

So, this code works because `Arc` ensures safe shared ownership and `Mutex` prevents simultaneous alterations!

Stay safe!
