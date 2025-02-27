# 全局变量
在一些场景，我们可能需要全局变量来简化状态共享的代码，包括全局ID，全局数据存储等等，下面一起来看看有哪些创建全局变量的方法。

首先，有一点可以肯定，全局变量的生命周期肯定是`'static`，但是不代表它需要用`static`来声明，例如常量、字符串字面值等无需使用`static`进行声明，原因是它们已经被打包到二进制可执行文件中。

下面我们从编译期初始化及运行期初始化两个类别来介绍下全局变量有哪些类型及该如何使用。

## 编译期初始化
我们大多数使用的全局变量都只需要在编译期初始化即可，例如静态配置、计数器、状态值等等。

#### 静态常量
全局常量可以在程序任何一部分使用，当然，如果它是定义在某个模块中，你需要引入对应的模块才能使用。常量，顾名思义它是不可变的，很适合用作静态配置：
```rust
const MAX_ID: usize =  usize::MAX / 2;
fn main() {
   println!("用户ID允许的最大值是{}",MAX_ID);
}
```

#### 静态变量
静态变量允许声明一个全局的变量，常用于全局数据统计，例如我们希望用一个变量来统计程序当前的总请求数：
```rust
static mut REQUEST_RECV: usize = 0;
fn main() {
   unsafe {
        REQUEST_RECV += 1;
        assert_eq!(REQUEST_RECV, 1);
   }
}
```

Rust要求必须使用`unsafe`语句块才能访问和修改`static`变量，因为这种使用方式往往并不安全，其实编译器是对的，当在多线程中同时去修改时，会不可避免的遇到脏数据。

只有在同一线程内或者不在乎数据的准确性时，才应该使用全局静态变量。

#### 原子类型
想要全局计数器、状态控制等功能，又想要线程安全的实现，原子类型是非常好的办法。

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
static REQUEST_RECV: AtomicUsize  = AtomicUsize::new(0);
fn main() {
    for _ in 0..100 {
        REQUEST_RECV.fetch_add(1, Ordering::Relaxed);
    }
  
    println!("当前用户请求数{:?}",REQUEST_RECV);
}
```

关于原子类型的讲解看[这篇文章](./concurrency-with-threads/sync2.md)


#### 示例：全局ID生成器
来看看如何使用上面的内容实现一个全局ID生成器:
```rust
use std::sync::atomic::{Ordering, AtomicUsize};

struct Factory{
    factory_id: usize,
}

static GLOBAL_ID_COUNTER: AtomicUsize = AtomicUsize::new(0);
const MAX_ID: usize = usize::MAX / 2;

fn generate_id()->usize{
    // 检查两次溢出，否则直接加一可能导致溢出
    let current_val = GLOBAL_ID_COUNTER.load(Ordering::Relaxed);
    if current_val > MAX_ID{
        panic!("Factory ids overflowed");
    }
    let next_id = GLOBAL_ID_COUNTER.fetch_add(1, Ordering::Relaxed);
    if next_id > MAX_ID{
        panic!("Factory ids overflowed");
    }
    next_id
}

impl Factory{
    fn new()->Self{
        Self{
            factory_id: generate_id()
        }
    }
}
```


## 运行期初始化
以上的静态初始化有一个致命的问题：无法用函数进行静态初始化，例如你如果想声明一个全局的`Mutex`锁：
```rust
use std::sync::Mutex;
static names: Mutex<String> = Mutex::new(String::from("Sunface, Jack, Allen"));

fn main() {
    let v = names.lock().unwrap();
    println!("{}",v);
}
```

运行后报错如下：
```console
error[E0015]: calls in statics are limited to constant functions, tuple structs and tuple variants
 --> src/main.rs:3:42
  |
3 | static names: Mutex<String> = Mutex::new(String::from("sunface"));
```

但你又必须在声明时就对`names`进行初始化，此时就陷入了两难的境地。好在天无绝人之路，我们可以使用`lazy_static`包来解决这个问题。

#### lazy_static
[`lazy_static`](https://github.com/rust-lang-nursery/lazy-static.rs)是社区提供的非常强大的宏，用于懒初始化静态变量，之前的静态变量都是在编译器初始化的，因此无法使用函数调用进行赋值，而`lazy_static`允许我们在运行期初始化静态变量！

```rust
use std::sync::Mutex;
use lazy_static::lazy_static;
lazy_static! {
    static ref names: Mutex<String> = Mutex::new(String::from("Sunface, Jack, Allen"));
}

fn main() {
    let mut v = names.lock().unwrap();
    v.push_str(", Myth");
    println!("{}",v);
}
```

当然，使用`lazy_static`在每次访问静态变量时，会有轻微的性能损失，因为其内部实现用了一个底层的并发原语`std::sync::Once`，在每次访问该变量时，程序都会执行一次原子指令用于确认静态变量的初始化是否完成。

可能有读者会问，为何需要在运行期初始化一个静态变量，除了上面的全局锁，你会遇到最常见的场景就是：**一个全局的动态配置，它在程序开始后，才加载数据进行初始化，最终可以让各个线程直接访问使用**

再来看一个使用`lazy_static`实现全局缓存的例子:
```rust
use lazy_static::lazy_static;
use std::collections::HashMap;

lazy_static! {
    static ref HASHMAP: HashMap<u32, &'static str> = {
        let mut m = HashMap::new();
        m.insert(0, "foo");
        m.insert(1, "bar");
        m.insert(2, "baz");
        m
    };
}

fn main() {
    // 首次访问`HASHMAP`的同时对其进行初始化
    println!("The entry for `0` is \"{}\".", HASHMAP.get(&0).unwrap());

    // 后续的访问仅仅获取值，再不会进行任何初始化操作
    println!("The entry for `1` is \"{}\".", HASHMAP.get(&1).unwrap());
}
```

需要注意的是，`lazy_static`直到运行到`main`中的第一行代码时，才进行初始化，非常`lazy static`。

#### Box::leak
在`Box`智能指针章节中，我们提到了`Box::leak`可以用于全局变量，例如用作运行期初始化的全局动态配置，先来看看如果不使用`lazy_static`也不使用`Box::leak`，会发生什么：
```rust
#[derive(Debug)]
struct Config {
    a: String,
    b: String,
}
static mut config: Option<&mut Config> = None;

fn main() {
    unsafe {
        config = Some(&mut Config {
            a: "A".to_string(),
            b: "B".to_string(),
        });

        println!("{:?}",config)
    }   
}
```

以上代码我们声明了一个全局动态配置`config`，并且其值初始化为`None`，然后在程序开始运行后，给它赋予相应的值，运行后报错:
```console
error[E0716]: temporary value dropped while borrowed
  --> src/main.rs:10:28
   |
10 |            config = Some(&mut Config {
   |   _________-__________________^
   |  |_________|
   | ||
11 | ||             a: "A".to_string(),
12 | ||             b: "B".to_string(),
13 | ||         });
   | ||         ^-- temporary value is freed at the end of this statement
   | ||_________||
   |  |_________|assignment requires that borrow lasts for `'static`
   |            creates a temporary which is freed while still in use
```

可以看到，Rust的借用和生命周期规则限制了我们做到这一点，因为试图将一个局部生命周期的变量赋值给全局生命周期的`config`，这明显是不安全的。

好在`Rust`为我们提供了`Box::leak`方法，它可以将一个变量从内存中泄漏(听上去怪怪的，竟然做主动内存泄漏)，然后将其变为`'static`生命周期，最终该变量将和程序活得一样久，因此可以赋值给全局静态变量`config`。

```rust
use std::sync::Mutex;

#[derive(Debug)]
struct Config {
    a: String,
    b: String
}
static mut config: Option<&mut Config> = None;

fn main() {
    let c = Box::new(Config {
        a: "A".to_string(),
        b: "B".to_string(),
    });

    unsafe {
        // 将`c`从内存中泄漏，变成`'static`生命周期
        config = Some(Box::leak(c));
        println!("{:?}", config);
    }
}
```


#### 从函数中返回全局变量
问题又来了，如果我们需要在运行期，从一个函数返回一个全局变量该如何做？例如：
```rust
#[derive(Debug)]
struct Config {
    a: String,
    b: String,
}
static mut config: Option<&mut Config> = None;

fn init() -> Option<&'static mut Config> {
    Some(&mut Config {
        a: "A".to_string(),
        b: "B".to_string(),
    })
}


fn main() {
    unsafe {
        config = init();

        println!("{:?}",config)
    }   
}
```

报错这里就不展示了，跟之前大同小异，还是生命周期引起的，那么该如何解决呢？依然可以用`Box::leak`:
```rust
#[derive(Debug)]
struct Config {
    a: String,
    b: String,
}
static mut config: Option<&mut Config> = None;

fn init() -> Option<&'static mut Config> {
    let c = Box::new(Config {
        a: "A".to_string(),
        b: "B".to_string(),
    });

    Some(Box::leak(c))
}


fn main() {
    unsafe {
        config = init();

        println!("{:?}",config)
    }   
}
```

## 标准库中的OnceCell
@todo

## 总结
在Rust中有很多方式可以创建一个全局变量，本章也只是介绍了其中一部分，更多的还等待大家自己去挖掘学习(当然，未来可能本章节会不断完善，最后变成一个巨无霸- , -)。

简单来说，全局变量可以分为两种：
- 编译期初始化的全局变量，`const`创建常量，`static`创建静态变量，`Atomic`创建原子类型
- 运行期初始化的全局变量，`lazy_static`用于懒初始化，`Box::leak`利用内存泄漏将一个变量的生命周期变为`'static`



