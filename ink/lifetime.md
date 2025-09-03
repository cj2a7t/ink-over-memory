## 1.回顾：引用&生命周期

### 指针很轻大很自由，但会带来内存问题

- C C++派：靠程序员能力，但容易百密一疏
- Java派：运行时引入GC，但性能、卡顿
- Rust使用ownership和引用的lifetime机制，在编译器进行检查和约束，不损失性能，但增加学习难度

### 有引用才会有生命周期

### 学习生命周期从两点入手：理解依赖+类比泛型

### Rust有两类引用

- &T：共享引用（**不可变引用？**），同一时刻可以有很多共享引用
- &mut T：独占引用（可变引用），同一时刻只能有一个独占引用

### 例子

```Rust
fn main() {
    let mut my_ref: Option<&i32> = None;
    // starting a scope.
    {
        let my_var: i32 = 7;
        my_ref = Some(&my_var);  // my_var 活的不够久
        // auto drop(my_var)
    }
     
    if let Some(ref) = my_ref {  // 这个地方还在用my_var，它的生命周期小于 my_ref
        println("{}", ref)      
    }
}
```

![](https://github.com/cj2a7t/ink-over-memory/blob/main/img/lifetime_1.png?raw=true)

## 2.生命周期-标注

### 编译器有时候无法帮我们判断，所以需要手动写生命周期的标注

- 类比：有时候编译器也无法帮你判断类型，需要你自己填写泛型类型

### 例子

#### (&, &) -> &: 返回值的引用一定是依赖入参的引用，如果是函数内部的引用生命周期早早就结束了

```Rust
fn main() {
    let a = 1;
    let my_num = comple_function(&a)
    println!("{my_num}");
}
fn complex_function(a: &i32) -> &i32 {
    let b = 2;
    max_of_refs(a, &b);
}
// (&, &) -> &: 返回值的引用一定是依赖入参的引用，如果是函数内部的引用生命周期早早就结束了
// &'static
fn max_of_refs(a: &i32, b:&i32) -> &i32 {
  if *a > *b {
      a
  } else { 
      b
  }
}


// 标注
fn main() {
    let a = 1;
    let my_num = comple_function(&a)
    println!("{my_num}");
}
fn complex_function(a: &i32) -> &i32 {
    let b = 2;
    max_of_refs(a, &b);
}
// 生命周期标注
fn max_of_refs<'a>(a: &'a i32, b:&'a i32) -> &'a i32 {
    if *a > *b {
        a
    } else { 
        b
    }
}
// 分别标注生命周期
fn max_of_refs<'a, 'b>(a: &'a i32, b:&'b i32) -> &'a i32 {
    a
}
```

#### 例子2只依赖某一个入参的练习

```Rust
// 如果只依赖一个参数的生命周期，那么就分别标，这样调用的时候会更宽松！
fn only_if_greater<'a, 'b>(number: &'a i32, greater_than: &'b i32) -> Option<&'a i32> {
    if number > greater_than {
        Some(number)
    } else {
        None
    }
}
```

```Rust
fn split<'a, 'b>(text: &'a str, delimiter: &'b str) -> Vec<&'a str> {
    let mut last_split = 0;
    let mut matches: Vec<&str> = vec![];
    for i in 0..text.len() {
        if i < last_split {
            continue
        }
        if text[i..].starts_with(delimiter) {
            matches.push(&text[last_split..i])
            last_split = i + delimiter.len();
        }
    }
    if last_split < text.len() {
        matchs.push(&text[last_split]..)
    }
    matches
}
```

#### 例子3

```Rust
fn main() {
    usage_1();
    usage_2();
}

fn identity(a: &i32) -> &i32 {
    a
}

// 没有问题
fn usage_1() {
    let x = 4;
    let x_ref = identity(&x);
    assert_eq(*x_ref, 4);
}

// x活的不够久
fn usage_2() {
    let mut x_ref: Opetion<&i32> = None;
    {
        let x = 7;
        x_ref = Some(identity(&x));
    }
    assert_eq!(&x_ref.unwrap(), 7);
}
```

## 3.生命周期-在函数上的消除

### 编译器会想办法帮你，尽量在一些场景下帮你自动标注生命周期

- 编译器会假定，在一些情况，我们大概率会这么标注，所以我们无需写标注
- 但有时并不是我们想要的，所以我们需要手工标注

### 入参规则

对于不同的引用的入参，会使用不同的生命周期标注

### 返回参规则

对于相同生命周期的入参，会进行返回参相同的生命周期标注

对于不同生命周期的入参，则不会进行标注

对于入参是&Self或者&mut Self，不管入参有多少个生命周期，都会使用Self的生命周期

## 4.生命周期-关注可变引用

有时候看起来可以省略生命周期，但实际不能省略（围绕依赖来理解）

```Rust
fn main() {
    let x = 1;
    let mut my_vec = vec![&x];
    
    {
        let y = 2;
        insert_value(&mut my_vec, &y);
        
        // 报错y活得不够久，因为y在进入到insert_value方法时候，生命周期被my_vec的值
        // 依赖，所以要和my_vec的值引用活得一样长
    
    }
    dbg!(my_vec);
}


fn insert_value<'a, 'b, 'c>(my_vec:&'a mut Vec<&'b i32>, value: &'c i32) {
     my_vec.push(vlue)
}
b 依赖 c，c要活得比b长，正确解法如下
fn insert_value<'a, 'b>(my_vec:&'a mut Vec<&'b i32>, value: &'b i32) {
     my_vec.push(vlue)
}



```

### 非常经典的例子

> 是否理解

```Rust
fn main() {
    let mut my_vec: Vec<&i32> = vec![];
    let val1 = 1;
    let val2 = 2;
    
    insert_value(&mut my_vec, &val1);
    insert_value(&mut my_vec, &val2);
    
    println!("{my_vec:?}");
}
// 生命周期全部标注为’a会报borrowed more than once，
// 因为&val1的生命周期到第九行，导致my_vec的生命周期也到第九行，才引发这个错误的
// 通过标记‘r和‘value两个生命周期，解绑他们的关系，所以即使&val1的生命周期到第九行，
// 但是my_vec的生命周期可以到第六行也是可以的
fn insert_value<'a>(my_vec:&'a mut Vec<&'a i32>, value: &'a i32) {
     my_vec.push(vlue)
}

// 解决方案: 生命周期解绑
fn insert_value<'r, 'vec>(my_vec:&'r mut Vec<&'vec i32>, value: &'vec i32) {
     my_vec.push(vlue)
}

```

## 5. struct/enum的生命周期

- 除了函数，类型上也需要指定生命周期，如struct、enum
- struct、enum上有了生命周期，impl也要有对应的声明

### 成员变量引用相同的生命周期

```Rust
// 两个参数的声明周期并不一定是相同的，取决于具体的用途
struct SplitStr<'a> {
    start: &'a str,
    end: &'a str
}
// 比如下面这个用途
fn split<'a, 'b>(text: &'a str, delimiter: &'b str) -> Option<SplitStr<'a>> {
    // split_onece这个函数的生命周期是&'a self，
    // 根据self规则->(start: &str, end: &str) 依赖text的生命周期
    // 所以这块可以标注SplitStr<'a>
    let (start: &str, end: &str) = text.split_onece(delimiter)?;
    Some(
        SplitStr {
            start, end
        }
    )
}
// 在写一个错误main，来熟悉这个规则
fn main() {

    // parts_of_string生命周期到println!("{parts_of_string:?}")这行结束
    // my_string到parts_of_string = split(my_string, ";");这个scope就结束了。
    // parts_of_string依赖my_string的生命周期
    // 所以会报my_string活得不够久
    let mut parts_of_string: Option<SplitStr> = None;
    {
        let my_string = String::from("first line; second line");
        parts_of_string = split(my_string, ";");
    }
    
    println!("{parts_of_string:?}")
    
}
```

### 成员变量引用不同的生命周期

```Rust
struct Different {
    first_only: Vec<&str>,
    second_only: Vec<&str>,
}


fn different(sentence_1: &str, sentence_2: &str) -> Different {
    // unimplements
}

// 上面这个函数里面，根据用途
struct Different<'a, 'b> {
    first_only: Vec<&'a str>,
    second_only: Vec<&'b str>,
}


fn different(sentence_1: &'a str, sentence_2: &'b str) -> Different<'a, 'b> {
    // unimplements
}


```

## 6.特殊的生命周期: 'static 和 '_

- 特殊的生命周期标识：'static和 '_
    - &'static T代表能在整个程序运行期间存活，如&'static str
    - 例子

```Rust
fn main() {
    let x;
    {
        let input = String::from("this is the input");
        x = foo(&input)
    }
    println!("{}", x);
}

fn foo<'a>(_input: &'a str) -> &'static str {
    let s: &'static str = "abc";
    s
}
```
- '_让编译器帮忙推倒生命周期，在如下这些地方可以用
    - impl 声明（帮助简化）
    - 输入/返回一个需要生命周期的类型是（Rust建议）
    - Trait object包含生命周期（有时候必须要用，这种情况下消除和'_不一样）
    - 例子

```Rust
struct Counter<'a> {
    counter: &'a mut i32
}

impl Counter<'_> {
    fn increment(&mut self) { 
        *self.counter += 1;
    }
}
// 可以使用'_，让其自行推倒
impl Counter<'_> {
    fn increment(&mut self) { 
        *self.counter += 1;
    }
}

```

## 7.生命周期 型变&Bound

### 协变（Covariant）

```Rust
// &'a T 对 'a 是协变的
fn covariant_example<'a, 'b: 'a>(x: &'a i32) -> &'b i32 {
    x // 错误！不能将较短的生命周期转换为较长的
}

// 正确的协变示例
fn covariant_correct<'a, 'b: 'a>(x: &'b i32) -> &'a i32 {
    x // 可以将较长的生命周期缩短
}
```

### 逆变（Contravariant）

```Rust
// 函数参数是逆变的
fn contravariant_example<'a, 'b: 'a>(f: fn(&'a i32)) -> fn(&'b i32) {
    f // 错误！不能将接受较短生命周期的函数转换为接受较长生命周期的函数
}
```

### 不变（Invariant）

```Rust
// Cell<T> 对 T 是不变的
use std::cell::Cell;

fn invariant_example(x: Cell<&'a i32>) -> Cell<&'b i32> {
    x // 错误！Cell 是不变的
}
```

### 规则

```Rust
&'a T 对 'a 协变，对 T 协变
&'a mut T 对 'a 协变，对 T 不变
fn(T) -> U 对 T 逆变，对 U 协变
Box<T> 对 T 协变
Vec<T> 对 T 协变
Cell<T> 对 T 不变

```

### 生命周期约束

```Rust
// 生命周期约束和泛型都是在<>中定义
// 从语法上来说，生命周期和泛型先后无所谓，但是社区约定都是生命周期在前，泛型在后
fn longest_with_an_announcement<'a, T>
//                              ^^^
//                              声明一个生命周期参数 'a
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where  // 约束
    T: std::fmt::Display,
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() { x } else { y }
}

```

