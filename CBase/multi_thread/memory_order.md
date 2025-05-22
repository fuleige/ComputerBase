# C++ 浅谈原子变量内存序使用

## 1. 基本概念和问题引入

### 1.1 基本概念

- 内存序就是CPU访问内存的顺序,它其实取决于编译器生成的指令顺序和CPU执行时的指令顺序.
- 程序执行即有指令重排序现象, 为了提高性能, 编译器可能会对指令进行重排序
    - 重排序为什么能提高性能, 举个很简单例子, 至少可以将访问某块相近的内存一些指令放在一起, 充分利用cache提高速度(本来这块内存本来加入到cache中的,随后随着访问移除cache了,然后后面有指令访问这块内存时,那不还得重新从内存读到cache啊)
    - 当然实际上编译器或者CPU对指令的优化为有着更加复杂的规则, 但不可否认, 指令并非完整按照编程的顺序执行

### 1.2 问题引入
以下是一个经典用于描述这个现象的示例
```C++
// 全局变量
int a = 0;
int b = 0;
bool init = false; 

// 线程1 (做一些初始化工作,然后更改标记状态)
a = 16;
b = 32;
init = true; // 由于存在指令重排的情况,可以先于前面的指令执行

// 线程2 
while(!init);
assert(a>10); // 仍然可能触发断言
```

- 指令之间的依赖关系

```C++
int x = 10;
int y = x + 5; // y对x有依赖关系,y执行这个操作不可能先于 x=10执行
int z = 3;     // 对前两条没有依赖关系, 可能早于前两条指令执行
```
- 对于没有依赖关系的指令,指令可能会重排序,这种行为应该是不可预期的
- 毕竟没有依赖关系的,怎么执行都不会影响程序最终的正确性
- 但是是如果是多个线程之间, 其他线程确实依赖于你的指令顺序性, 那么可能会发生非预期的结果

## 2. 原子操作和内存序

### 2.1 原子操作

- 所谓原子操作, 就是整个操作的过程中是不可以被中断的
    - store 写进值
    - load 读取出值
    - 读-改-写 操作: 如 fetch_add等
        - fetch_add 是一个加法操作, 稍微不严谨的说, 它从内存中读取到最新值, 加上值, 再把这个值写回内存
        - 由于这个过程不可中断, 别的线程即便同时想调用这个方法, 也必然有个先后顺序, 而且别的线程能从获取到最新的值继续进行加法操作

- 示例代码

```C++
std::atomic_uint32_t val{0};
    
val.store(1);
uint32_t tmp_val = val.load(); // tmp_val值为1

val.fetch_add(2);
tmp_val = val.load(); // tmp_val值为3
```


- C++提供了多种内存序策略,供你在不同的场景下使用
### 2.2 C++内存序

#### 2.2.1 memory_order_relaxed

- 宽松型的内存序, 用这种内存序就没有任何内存序上的顺序保护, 只保证你的操作本身是原子性的
- 实际上在编译器层面不做额外的事, 如插入内存屏障保证指令的顺序性, 至少C++编译器层面不做这些事
    - 一般的C++实现是依靠硬件提供的指令完成原子操作, C++本身不做任何特殊处理
- 如果你本身没有指令上没有顺序性的要求, 只有原子性的要求, 那么使用这个内存序开销最小


- 示例代码

```C++
// 支持参数设置内存序
val.store(1, std::memory_order_relaxed);
uint32_t tmp_val = val.load(std::memory_order_relaxed); // tmp_val值为1

val.fetch_add(2, std::memory_order_relaxed);
tmp_val = val.load(std::memory_order_relaxed); // tmp_val值为3
```

#### 2.2.2 memory_order_acquire 和 memory_order_release

- 上面说到的这种内存序,本质上还是没有解决问题, 因为没有对内存序做任何处理, 所以这里引入这里的内存序
- memory_order_acquire, 只可用于load操作(读操作), 它确保在它之后的读写指令不会排在该操作之前
- memory_order_release, 只可用于store操作(写操作), 它确保在它之前的读写指令不会排在该操作之后 
    - 这里的读写指令就是读写内存的相关指令, 不管是不是原子操作都受到这个约束
- 靠这对指令就可以解决上述问题了

- 示例代码

```C++
// 全局变量
int a = 0;
int b = 0;
std::atomic_bool init{false};

// 线程1 
a = 16;
b = 32;

// 用memory_order_release可以确保,a和b这些操作指令不会被重排带init store之后
// 如果其他线程检测到了init为true, 那么a和b必定已经初始化
init.store(true, std::memory_order_release);

// 线程2 

while(!init.load(std::memory_order_acquire));
// 这里使用了memory_order_acquire, 保证 a>10这条指令不会被重排到init判断为true之前
// 而init设置为true之后, a和b必定已经设定值成功, 则必定不会导致断言失败
assert(a>10); 

```

#### 2.2.3 memory_order_consume

- 只能用于load(读操作), 保证之后的依赖原子变量相关的操作指令不会重排到该指令之前, 也可以和memory_order_release搭配使用
- 这个内存序的出现的原因很明显, 那就是优化 memory_order_acquire 的性能
- memory_order_acquire 要求所有的读写指令必须要都不能重排到load之前, 而 memory_order_consume 只要依赖原子变量相关的指令不排在load之前

- 官方示例demo解释

```C++
// 全局变量
std::atomic<std::string*> ptr; // 默认空指针
int data = 0;

// 线程1
std::string* p = new std::string("Hello");
data = 42;
// 如果ptr不是空指针, 则 data=42必定执行了
ptr.store(p, std::memory_order_release);

// 线程2
std::string* p2;
while (!(p2 = ptr.load(std::memory_order_consume))); //判断原子变量保存的地址是否为空指针

assert(*p2 == "Hello"); // 断言永远不会触发, 因为依赖于取原子变量的值, 这个指令不会优化到判空之前

assert(data == 42); // 可能会触发断言, 虽然 ptr不为空指针了,data一定等于42, 关键这个指令可能被优化了在判空之前就执行了
// 因为这条指令和原子变量 ptr没有依赖关系, 并不保证data==42这种判断一定在 (!(p2 = ptr.load(std::memory_order_consume)))之后执行
```

#### 2.2.4 memory_order_acq_rel

- 这个内存序官方介绍很简单, 可以用于store,也可以用于load
- 用于store(写操作)时和memory_order_release效果一致
- 用于load(读操作)时和memory_order_acquire一致
- 毕竟memory_order_acq_rel最后两个单词acq就是acquire简写,rel就是release简写
- 官方文档对这个内存序介绍很少,看起来它的意义就是怕你忘记store该用哪一个(acacquire还是release),load该用哪一个内存序
- 那就直接统一一下, 记不起来, 就直接用它就行了, 它的效果就是和memory_order_release和memory_order_acquire一致的

#### 2.2.5 memory_order_seq_cst (默认值)

- 前面讲的内存序缺少一个功能, 那就是对于"读-改-写"这种操作,我需要严格一点
- 既要保证在这个原子操作指令之前的指令不重排到后面去,也保证在后面的指令不准重排到前面来, memory_order_seq_cst能做到对"读-改-写"操作时实现这种功能
- 然后,如果用于store(写操作)时和memory_order_release效果一致, 用于load(读操作)时和memory_order_acquire一致
- 所以选择它当默认值, 属于最保险的一种内存序, 同时也是开销最大的内存序, 毕竟CPU重排序能加速程序优化性能, 它直接做了很严格的内存序削弱了这种优化
