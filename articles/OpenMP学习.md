

## 核心概念
- 共享内存模型：所有线程访问同一地址空间
- Fork-Join模型：串行 -> 并行域（fork）-> 同步汇合（join）-> 串行
- 编译制导：#pragma omp ... 串行编译器忽略，OpenMP编译器生效
- 线程角色：主线程（ID=0）创建并行域，工作线程协同执行

## 一个简单示例

```cpp
// test.cpp
#include<iostream>
#include<omp.h>

int main() {

    printf("AAA\n");
    #pragma omp parallel num_threads(3)
    {
        printf("BBB\n");
    }
    printf("CCC\n");
}
```
<img src="./images/OpenMP_learn/simple_example.png" width="80%" />

## 指令结构

```cpp
#pragma omp 指令名称 [子句...]
```

以`#pragma omp`开头，指令用于指定操作类型，子句用于控制行为。
子句可以有零个或多个。

## 常用的指令名

**并行结构类：**
- parallel - 创建并行区域（创建线程组）
- parallel for - 并行循环
- parallel sections - 并行分段

**工作共享类：**
- for - 分发循环迭代
- sections - 分发代码段
- task - 创建任务并分发

**同步控制类：**
- critical - 互斥访问
- atomic - 原子操作
- barrier - 同步屏障
- master - 主线程执行
- single - 单线程执行

### 使用示例一

```cpp
#pragma omp sections
{
    #pragma omp section
    { task1(); }
    #pragma omp section  
    { task2(); }
}
```
**解释：**
`#pragma omp sections` 标记了一个可并行执行的代码块集合的开始。
在这个代码块内，每个 `#pragma omp section` 指令定义了一个独立的、互不依赖的代码段。
运行时，这些 `section` 会被分配给不同的线程同时执行。

**关键点：**
- 并行执行：`task1()` 和 `task2()` 可以由两个线程同时运行，而不是顺序执行。
- 负载均衡：如果 `task1()` 和 `task2()` 工作量不同，空闲的线程可能会协助完成剩余任务（具体行为依赖于实现和运行时）。
- 隐式同步：在 `sections` 块的末尾有一个隐式屏障，所有线程必须完成各自分配的 section 后，才能继续执行后续代码。
- 扩展示例：可以定义任意数量的 `section`（通常不超过可用线程数）。

### 使用示例二

```cpp
#pragma omp parallel
#pragma omp single // 通常用单个线程创建任务
{
    #pragma omp task depend(out: x) // 任务1：产生数据x
    { x = compute(); }
    
    #pragma omp task depend(in: x)  // 任务2：依赖x
    { process1(x); }
    
    #pragma omp task depend(in: x)  // 任务3：也依赖x
    { process2(x); }
    
    #pragma omp taskwait // 等待所有任务完成
} // 隐式屏障：所有线程在此同步
```

**解释：**
`#pragma omp task` 创建一个显式任务。
遇到此指令时，运行时系统可能会立即执行 `process(data)`，也可能将其放入任务池，稍后由空闲线程执行。
任务非常适合处理不规则并行、递归算法或异步操作。

**关键点：**
- 动态生成：任务可以在循环或条件语句中动态创建。
- 依赖控制：可使用 `depend` 子句定义任务间的数据依赖关系。
- 任务等待：`#pragma omp taskwait` 可等待当前任务的所有子任务完成。

**注意：**
- `task` 是工作共享指令，它本身不创建线程，只创建任务。
- `parallel` 创建线程组：`#pragma omp parallel` 会生成一组工作线程（如4个线程）。
- `single` 控制任务生成：在并行区域，用 `single` 指定一个线程（如主线程）来执行花括号内的代码并生成任务。
- 其他线程执行任务：生成的任务会被放入任务池，由包括生成线程在内的所有线程竞争执行。

**与 sections 的区别：**
- sections：结构化的、静态的并行块，在编译时确定所有并行段。
- task：动态的、灵活的任务生成，可在运行时根据条件或数据创建。

## 常用的子句

**数据作用域类：**
- private - 每个线程私有副本
- shared - 线程间共享变量
- firstprivate - 私有且继承初值
- lastprivate - 私有且最后迭代值传出
- default - 设置默认作用域

**线程数量控制类：**
- num_threads - 指定线程数

**调度控制类：**
- schedule - 循环迭代分配策略（static/dynamic/guided/auto/runtime）

**同步控制类：**
- nowait - 取消隐式屏障
- ordered - 保持循环顺序执行
- reduction - 规约操作（+、*、max等）

**任务相关类：**
- depend - 任务数据依赖（in/out/inout）
- final - 条件性终止任务生成
- mergeable - 允许任务合并

**其他：**
- if - 条件并行
- collapse - 嵌套循环合并并行
- copyin - 初始化线程私有变量

## 常用的函数
```cpp
int omp_get_thread_num();   // 当前线程ID

int omp_get_num_threads();  // 并行域内线程总数

int omp_get_max_threads();  // 最大可用线程数

void omp_set_num_threads(int n); // 设置后续并行域线程数

double omp_get_wtime();     // 高精度计时（秒）
```