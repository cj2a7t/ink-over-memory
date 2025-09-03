### 基本使用

```Rust
#[derive(Debug)]
struct Book {
    title: String,
}

impl Book {
    // 1️⃣ &self：只读方法，不能改动字段
    fn read(&self) {
        println!("Reading book: {}", self.title);
    }

    // 2️⃣ &mut self：可写方法，可以修改字段
    fn rename(&mut self, new_title: &str) {
        self.title = new_title.to_string();
    }

    // 3️⃣ self：拿走所有权，消费这个 Book
    fn destroy(self) {
        println!("Destroying book: {}", self.title);
        // ⚠️ 之后不能再使用这个 Book 了
    }

    // 4️⃣ mut self：可变绑定的 self，可以修改后返回
    fn into_upper(mut self) -> Self {
        self.title = self.title.to_uppercase();
        self // 返回修改后的自己
    }
}


fn main() {

    let mut book = Book { title: "Rust入门".to_string() };

    // ✅ 只读
    book.read();

    // ✅ 修改字段（但 book 仍然存在）
    book.rename("Rust进阶");
    book.read();

    // ✅ 修改 self 并返回一个新值
    let book = book.into_upper();
    book.read();

    // ✅ 最后消费这个 book（不能再用了）
    book.destroy();

    // ❌ 下面这一行如果取消注释会报错：
    // book.read(); // error: value borrowed here after move
}


```

### 建造者模式

```Rust
#[derive(Debug)]
struct Config {
    host: String,
    port: u16,
}

impl Config {
    // 构建器初始化
    fn new() -> Self {
        Config {
            host: "localhost".to_string(),
            port: 8080,
        }
    }

    // 👇 这里使用 mut self 而不是 &mut self
    fn set_host(mut self, host: &str) -> Self {
        self.host = host.to_string();
        self
    }

    fn set_port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }

    fn build(self) -> Self {
        self
    }
}


fn main() {
    let config = Config::new()
        .set_host("example.com")
        .set_port(9090)
        .build();

    println!("{:#?}", config);
}


```



## 🧠 更明确的对比总结：

只要带`&`，就是借用（borrow；不带`&`，就是移动（move），除非该类型是`Copy`

|接收方式|是否移动所有权|是否允许修改|说明|
|-|-|-|-|
|`self`|✅ 是|✅ 是|消费对象，不能再用了|
|`mut self`|✅ 是|✅ 是|消费对象，可变地使用再返回|
|`&self`|❌ 否|❌ 否|**只读借用**|
|`&mut self`|❌ 否|✅ 是|**可变借用**，不会移动所有权|


```Rust
fn take(val: String) {
    println!("{val}");
}

fn borrow(val: &String) {
    println!("{val}");
}

fn main() {
    let s = String::from("hello");

    borrow(&s);  // ✅ 借用，s 还能用
    take(s);     // ✅ 所有权 move
    // borrow(&s); ❌ 错：s 已被移动
}

```

