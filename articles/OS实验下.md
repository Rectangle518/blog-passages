
## 实验六-并发与锁机制

对于一些在多个线程之间共享的变量，如果我们不采取任何措施来协调线程之间对这个共享变量的访问顺序，就会产生和我们预期不一致的结果，或者说是race condition。这个问题的解决就在于我们需要通过一种工具来协调线程之间的对共享变量的访问顺序，这个工具就是锁。而“协调线程之间的对共享变量的访问顺序”也被称为线程的同步和互斥。

### 任务一-锁机制的实现

#### 代码复现

分别使用自旋锁和信号量实现一个锁机制，并解决消失的芝士汉堡问题。

- 首先实现自旋锁，自旋锁(spin lock)是最简单的锁，是用来实现互斥的工具。

我们称“访问共用资源的代码”为临界区。自旋锁的基本思想是定义一个共享变量`bolt`，`bolt`会被初始化为0。在线程进入临界区之前，即访问共享变量之前，都需要去检查`bolt`是否为0。如果`bolt`为0，那么这个线程就会将`bolt`设置为1，然后进入临界区。待线程离开临界区后，线程会将`bolt`设置为0。如果线程在检查`bolt`时，发现`bolt`为1，说明有其他线程在临界区中。此时这个线程就会一直在循环检查`bolt`的值（类似陀螺在原地旋转，所以被称为自旋），直到`bolt`为0，然后进入临界区。

从上面的描述中我们可以看到，同一时刻只能有一个线程在临界区中，并且线程的循环等待并不能保证有限等待的原则。因此在使用自旋锁的时候，我们假设了各个线程在临界区的时间是短暂的。

下面我们就根据上面的描述来实现自旋锁。

我们首先在`include/sync.h`文件下定义表示自旋锁的类`SpinLock`。

```cpp
#ifndef SYNC_H
#define SYNC_H

#include "os_type.h"

class SpinLock
{
private:
    // 共享变量
    uint32 bolt;
public:
    SpinLock();
    void initialize();
    // 请求进入临界区
    void lock();
    // 离开临界区
    void unlock();
};
#endif
```

我们在`src/kernel/sync.cpp`中实现之。

在使用`SpinLock`之前，我们需要对`SpinLock`的成员变量`bolt`进行初始化为0。

```cpp
SpinLock::SpinLock()
{
    initialize();
}

void SpinLock::initialize()
{
    // 将bolt初始化为 0
    bolt = 0;
}
```

由于我们常常将`SpinLock`定义为一个全局变量，而全局变量的构造函数在我们的操作系统实验中不会被自动调用。所以在使用`SpinLock`的时候，我们需要手动调用`SpinLock::initialize`。

接着，我们实现线程请求进入临界区的函数`SpinLock::lock`。

```cpp
void SpinLock::lock()
{
    uint32 key = 1;

    do
    {
        // 原子操作，将bolt的值设置为 1
        asm_atomic_exchange(&key, &bolt);
    } while (key);
}
```

注意：我们在实现`asm_atomic_exchange`的时候作了一个重要的假设，我们假设形式参数register指向的变量不是一个共享变量。只有满足这个条件的时候，`asm_atomic_exchange`才会是原子的。

当线程离开临界区的时候，我们简单地将bolt置为0即可。

```cpp
void SpinLock::unlock()
{
    bolt = 0;
}
```

接下来使用自旋锁解决消失的芝士汉堡问题。

在母亲制作芝士汉堡会为其加上锁，在晾完衣服并检查芝士汉堡后，才释放锁。

```cpp
void a_mother(void *arg)
{
    // 获取锁，进入临界区
    aLock.lock();
    
    int delay = 0;

    printf("mother: start to make cheese burger, there are %d cheese burger now\n", cheese_burger);
	 ...
    printf("mother: Oh, Jesus! There are %d cheese burgers\n", cheese_burger);
    
    // 释放锁，离开临界区
    aLock.unlock();
}
```

儿子回来后，无法打开锁，只能在原地等待母亲晾完衣服。

```cpp
void a_naughty_boy(void *arg)
{
    aLock.lock();
    printf("boy   : Look what I found!\n");
    // 吃掉所有的汉堡
    cheese_burger -= 10;
    aLock.unlock();
}
```

在使用`SpinLock`之前，我们需要初始化。

```cpp
void first_thread(void *arg)
{
    // 第1个线程不可以返回
	...
    aLock.initialize();
	...
}
```

我们编译运行，输出如下结果。

<img src="./images/OS_lab/lab6/spinlock_hamburger.png" width="700px" />

- 接下来实现信号量。

我们首先定义一个非负整数counter表示临界资源的个数。

当线程需要申请临界资源时，线程需要执行P操作。P操作会检查counter的数量，如果counter大于0，表示临界资源有剩余，那么就将一个临界资源分配给请求的线程；如果counter等于0，表示没有临界资源剩余，那么这个线程会被阻塞，然后挂载到信号量的阻塞队列当中。

当线程释放临界资源时，线程需要执行V操作。V操作会使counter的数量递增1，然后V操作会检查信号量内部的阻塞队列是否有线程，如果有，那么就将其唤醒。

从上面的描述可以看到，counter和阻塞队列是共享变量，需要实现互斥访问。目前，我们实现互斥工具只有SpinLock，因此，我们实际上使用SpinLock去实现信号量。

我们首先在`include/sync.h`下定义信号量`Semaphore`。

```cpp
class Semaphore
{
private:
    uint32 counter; // 临界资源的个数
    List waiting;   // 阻塞队列
    SpinLock semLock; // 互斥锁

public:
    Semaphore();
    void initialize(uint32 counter);
    void P();
    void V();
};
```

`Semaphore`的实现放置在`src/kernel/sync.cpp`下。

在使用`Semaphore`之前，我们需要对其进行初始化。

```cpp
Semaphore::Semaphore()
{
    initialize(0);
}

void Semaphore::initialize(uint32 counter)
{
    this->counter = counter;
    semLock.initialize();
    waiting.initialize();
}
```

我们实现P操作。

```cpp
void Semaphore::P()
{
    PCB *cur = nullptr;

    while (true)
    {
        // 请求进入临界区
        semLock.lock();

        // 检查是否有临界资源可分配
        if (counter > 0)
        {
            --counter;
            semLock.unlock();
            return;
        }

        // 没有临界资源可分配，线程被阻塞
        cur = programManager.running;
        waiting.push_back(&(cur->tagInGeneralList));
        cur->status = ProgramStatus::BLOCKED;

        // 释放锁，调度其他线程来执行
        semLock.unlock();
        programManager.schedule();
    }
}
```

然后我们实现V操作。

```cpp
void Semaphore::V()
{
    // 先加锁，释放资源
    semLock.lock();
    ++counter;

    // 然后检查是否有线程被阻塞
    if (waiting.size())
    {
        // 唤醒一个线程
        PCB *program = ListItem2PCB(waiting.front(), tagInGeneralList);
        waiting.pop_front();
        semLock.unlock();
        programManager.MESA_WakeUp(program);
    }
    else
    {
        semLock.unlock();
    }
}
```

线程的唤醒的实现在`src/kernel/program.cpp`下。

```cpp
void ProgramManager::MESA_WakeUp(PCB *program) {
    // 将线程状态设置为就绪态，然后放入就绪队列
    program->status = ProgramStatus::READY;
    readyPrograms.push_front(&(program->tagInGeneralList));
}
```

我们只要简单地将`program`的状态设置为就绪态，然后放入就绪队列即可。

现在，我们使用信号量来解决“消失的芝士汉堡”问题。

母亲使用了第二代锁来保护汉堡。

```cpp
void a_mother(void *arg)
{
    semaphore.P();
    int delay = 0;

	 ...
        
    printf("mother: Oh, Jesus! There are %d cheese burgers\n", cheese_burger);
    semaphore.V();
}
```

儿子回来后，发现汉堡被锁上了，等母亲喊他吃饭。

```cpp
void a_naughty_boy(void *arg)
{
    semaphore.P();
    printf("boy   : Look what I found!\n");
    // 吃掉所有芝士汉堡
    cheese_burger -= 10;
    semaphore.V();
}
```

在使用`Semaphore`之前，由于我们需要对`cheese_burger`进行互斥访问，我们需要将临界资源的数量初始化为1。

```cpp
void first_thread(void *arg)
{
    // 第1个线程不可以返回
	...
    semaphore.initialize(1);
	...
}
```

最后我们编译运行，输出如下结果。

<img src="./images/OS_lab/lab6/counter_hamburger.png" width="700px" />

#### 其他方式实现锁机制

使用 lock cmpxchg 实现 CAS 循环来替代 xchg。

```asm
global your_asm_atomic_exchange

; void your_asm_atomic_exchange(uint32 *register, uint32 *memory)
; 原子地交换 *register 和 *memory 的值
; 使用 lock cmpxchg 实现的 CAS 循环替代 xchg
your_asm_atomic_exchange:
    push ebp
    mov ebp, esp
    pushad

    mov esi, [ebp + 4 * 2]  ; register (key) 地址
    mov edi, [ebp + 4 * 3]  ; memory (bolt) 地址
    mov edx, [esi]          ; edx = *register (期望写入 *memory 的新值)

retry:
    mov eax, [edi]          ; eax = *memory 当前值 (expected)
    lock cmpxchg [edi], edx ; 原子 CAS：若 eax==[edi]，则 [edi]=edx，ZF=1
                            ;          若 eax!=[edi]，则 eax=[edi]，ZF=0
    jnz retry               ; CAS 失败则重试

    mov [esi], eax          ; *register = *memory 旧值

    popad
    pop ebp
    ret
```

`lock cmpxchg [edi], edx` 是一条真正的原子指令，它不可分割地完成"比较 EAX 与 [edi]，若相等则将 EDX 写入 [edi]"。通过 CAS 循环实现交换：先把 *key 的值存入 EDX 作为"想写入 bolt 的新值"，然后反复尝试将 EDX 写入 bolt（前提是 bolt 没有被其他人改变）。CAS 成功后，EAX 中保存的就是 bolt 的旧值，写回 key 即完成交换。
与原方案的 xchg 相比，`lock cmpxchg` 是一条完全原子的指令（对共享变量 [edi] 的读-改-写不可分割），不存在原方案中"先读 key 到 EAX，再 xchg"的两步问题。

修改 `SpinLock` 的实现，使用 `your_asm_atomic_exchange`。并在 include/asm_utils.h 中添加 `your_asm_atomic_exchange` 的声明。

```cpp
void SpinLock::lock()
{
    uint32 key = 1;
    do
    {
        your_asm_atomic_exchange(&key, &bolt);
    } while (key);
}
```

运行结果如下。

<img src="./images/OS_lab/lab6/other_bamburger.png" width="700px" />

### 任务二-生产者/消费者问题

#### 竞态条件

以一个简单的生产者/消费者问题为例，假设有一个共享缓冲区，生产者往缓冲区中插入项目，消费者从缓冲区中取出项目。如果生产者和消费者同时访问缓冲区，可能会出现竞态条件。

这里假设缓冲区的大小为2。一共有两个生产者，生产者每次生产2个项目。一共有三个消费者，消费者每次消费3个项目。

在 `setup.cpp` 里，我们定义了共享缓冲区、生产者线程和消费者线程，代码如下。在以下代码中，使用了一些技巧，增大了发生竞态条件的概率。

```cpp
// 共享缓冲区（用于演示竞态条件）
#define BUFFER_SIZE 2
int buffer[BUFFER_SIZE];
int buffer_count = 0;  // 当前缓冲区中的项目数量
int in = 0;           // 生产者插入位置
int out = 0;          // 消费者取出位置

// 统计信息
int total_produced = 0;
int total_consumed = 0;

void producer_thread(void *arg)
{
    int producer_id = *(int*)arg;
    
    // 每个生产者生产2个项目
    for (int i = 0; i < 2; i++) {
        // 检查缓冲区是否已满（但实际上可能因为竞态条件而错误判断）
        int observed_count = buffer_count;
        programManager.schedule(); // 强制切换，让检查与更新分离
        if (observed_count < BUFFER_SIZE) {
            // 生产一个项目（项目值 = 生产者ID * 10 + 序号）
            int item = producer_id * 10 + i;
            buffer[in] = item;
            in = (in + 1) % BUFFER_SIZE;
            buffer_count = observed_count + 1;  // 使用过期值更新，扩大竞态窗口
            total_produced++;
            
            printf("Producer %d: Produced item %d, buffer_count=%d\n", producer_id, item, buffer_count);
        } else {
            printf("Producer %d: Buffer full! Cannot produce item %d\n", producer_id, producer_id * 10 + i);
        }
        
        // 模拟生产时间
        int delay = 0xfffffff;
        while (delay) delay--;
    }
    
    printf("Producer %d: Finished production.\n", producer_id);
}

void consumer_thread(void *arg)
{
    int consumer_id = *(int*)arg;
    
    // 每个消费者尝试消费3次
    for (int i = 0; i < 3; i++) {
        // 检查缓冲区是否为空（但实际上可能因为竞态条件而错误判断）
        int observed_count = buffer_count;
        programManager.schedule(); // 强制切换，让检查与更新分离
        if (observed_count > 0) {
            // 消费一个项目
            int item = buffer[out];
            out = (out + 1) % BUFFER_SIZE;
            buffer_count = observed_count - 1;  // 使用过期值更新，扩大竞态窗口
            total_consumed++;
            
            printf("Consumer %d: Consumed item %d, buffer_count=%d\n", consumer_id, item, buffer_count);
        } else {
            printf("Consumer %d: Buffer empty! Cannot consume.\n", consumer_id);
        }
        
        // 模拟消费时间
        int delay = 0xfffffff;
        while (delay) delay--;
    }
    
    printf("Consumer %d: Finished consumption.\n", consumer_id);
}

void first_thread(void *arg)
{
    // 第1个线程不可以返回
    stdio.moveCursor(0);
    for (int i = 0; i < 25 * 80; ++i)
    {
        stdio.print(' ');
    }
    stdio.moveCursor(0);

    printf("=== Race Condition Demonstration: Producer-Consumer Problem ===\n");
    printf("Buffer size: %d\n", BUFFER_SIZE);
    printf("2 Producers, 2 Consumers, NO SYNCHRONIZATION!\n\n");

    // 初始化缓冲区
    for (int i = 0; i < BUFFER_SIZE; i++) {
        buffer[i] = 0;
    }
    buffer_count = 0;
    in = 0;
    out = 0;
    total_produced = 0;
    total_consumed = 0;

    // 创建生产者和消费者参数
    int prod1_id = 1, prod2_id = 2;
    int cons1_id = 1, cons2_id = 2;

    // 创建2个生产者线程和2个消费者线程
    programManager.executeThread(producer_thread, &prod1_id, "producer1", 1);
    programManager.executeThread(producer_thread, &prod2_id, "producer2", 1);
    programManager.executeThread(consumer_thread, &cons1_id, "consumer1", 1);
    programManager.executeThread(consumer_thread, &cons2_id, "consumer2", 1);

    // 等待所有线程完成
    int delay = 0xffffffff;
    while (delay) delay--;

    // 显示最终结果
    printf("\n=== Final Results ===\n");
    printf("Total produced: %d\n", total_produced);
    printf("Total consumed: %d\n", total_consumed);
    printf("Final buffer_count: %d\n", buffer_count);
    printf("Expected buffer_count: %d (produced - consumed)\n", total_produced - total_consumed);
    
    // 检查是否发生竞态条件
    if (buffer_count != (total_produced - total_consumed)) {
        printf("!!! RACE CONDITION DETECTED !!!\n");
        printf("Buffer count inconsistency due to lack of synchronization.\n");
    }
    
    // 检查缓冲区是否出现负数或超过容量的情况
    if (buffer_count < 0) {
        printf("!!! BUFFER UNDERFLOW DETECTED !!!\n");
        printf("Consumers consumed more than producers produced.\n");
    }
    if (buffer_count > BUFFER_SIZE) {
        printf("!!! BUFFER OVERFLOW DETECTED !!!\n");
        printf("Producers produced more than buffer capacity.\n");
    }

    asm_halt();
}
```

编译运行之后，结果如下。

<img src="./images/OS_lab/lab6/conflic.png" width="700px" />

可以看到，由于没有同步机制，最终结果与预期不符，发生了竞态条件。

#### 信号量解决办法

接下来使用信号量解决上述问题。

```cpp
// 共享缓冲区（用于演示竞态条件）
#define BUFFER_SIZE 2
int buffer[BUFFER_SIZE];
int buffer_count = 0;  // 当前缓冲区中的项目数量
int in = 0;           // 生产者插入位置
int out = 0;          // 消费者取出位置

// 统计信息
int total_produced = 0;
int total_consumed = 0;

// 信号量：empty(空槽), full(满槽), mutex(互斥)
Semaphore empty_sem;
Semaphore full_sem;
Semaphore mutex_sem;

void producer_thread(void *arg)
{
    int producer_id = *(int*)arg;
    
    // 每个生产者生产2个项目
    for (int i = 0; i < 2; i++) {
        empty_sem.P();
        mutex_sem.P();
        // 生产一个项目（项目值 = 生产者ID * 10 + 序号）
        int item = producer_id * 10 + i;
        buffer[in] = item;
        in = (in + 1) % BUFFER_SIZE;
        buffer_count++;
        total_produced++;
        printf("Producer %d: Produced item %d, buffer_count=%d\n", producer_id, item, buffer_count);
        mutex_sem.V();
        full_sem.V();
        
        // 模拟生产时间
        int delay = 0xfffffff;
        while (delay) delay--;
    }
    
    printf("Producer %d: Finished production.\n", producer_id);
}

void consumer_thread(void *arg)
{
    int consumer_id = *(int*)arg;
    
    // 每个消费者尝试消费3次
    for (int i = 0; i < 3; i++) {
        full_sem.P();
        mutex_sem.P();
        // 消费一个项目
        int item = buffer[out];
        out = (out + 1) % BUFFER_SIZE;
        buffer_count--;
        total_consumed++;
        printf("Consumer %d: Consumed item %d, buffer_count=%d\n", consumer_id, item, buffer_count);
        mutex_sem.V();
        empty_sem.V();
        
        // 模拟消费时间
        int delay = 0xfffffff;
        while (delay) delay--;
    }
    
    printf("Consumer %d: Finished consumption.\n", consumer_id);
}

void first_thread(void *arg)
{
    // 第1个线程不可以返回
    stdio.moveCursor(0);
    for (int i = 0; i < 25 * 80; ++i)
    {
        stdio.print(' ');
    }
    stdio.moveCursor(0);

    printf("=== Race Condition Demonstration: Producer-Consumer Problem ===\n");
    printf("Buffer size: %d\n", BUFFER_SIZE);
    printf("2 Producers, 2 Consumers, USE SEMAPHORES!\n\n");

    // 初始化缓冲区
    for (int i = 0; i < BUFFER_SIZE; i++) {
        buffer[i] = 0;
    }
    buffer_count = 0;
    in = 0;
    out = 0;
    total_produced = 0;
    total_consumed = 0;

    empty_sem.initialize(BUFFER_SIZE);
    full_sem.initialize(0);
    mutex_sem.initialize(1);

    // 创建生产者和消费者参数
    int prod1_id = 1, prod2_id = 2;
    int cons1_id = 1, cons2_id = 2;

    // 创建2个生产者线程和2个消费者线程
    programManager.executeThread(producer_thread, &prod1_id, "producer1", 1);
    programManager.executeThread(producer_thread, &prod2_id, "producer2", 1);
    programManager.executeThread(consumer_thread, &cons1_id, "consumer1", 1);
    programManager.executeThread(consumer_thread, &cons2_id, "consumer2", 1);

    // 等待所有线程完成
    int delay = 0xffffffff;
    while (delay) delay--;

    // 显示最终结果
    printf("\n=== Final Results ===\n");
    printf("Total produced: %d\n", total_produced);
    printf("Total consumed: %d\n", total_consumed);
    printf("Final buffer_count: %d\n", buffer_count);
    printf("Expected buffer_count: %d (produced - consumed)\n", total_produced - total_consumed);
    
    // 检查是否发生竞态条件
    if (buffer_count != (total_produced - total_consumed)) {
        printf("!!! INCONSISTENCY DETECTED !!!\n");
        printf("Buffer count inconsistency despite synchronization.\n");
    }

    asm_halt();
}
```

编译运行之后，结果如下。

<img src="./images/OS_lab/lab6/solve.png" width="700px" />

可以看到，使用信号量后，最终结果与预期一致，没有发生竞态条件。

### 任务三-哲学家就餐问题

通过一段代码，演示哲学家就餐问题，并使用信号量来解决该问题。

```cpp
// 哲学家就餐问题（5位哲学家，5把叉子）
#define PHIL_COUNT 5

// 信号量：叉子
Semaphore forks[PHIL_COUNT];

// 信号量：就餐区（最多允许N-1位哲学家同时就餐）
Semaphore room_limit;

// 统计信息：每个哲学家已经完成的就餐次数
int eat_count[PHIL_COUNT];

// 模拟延迟
static void short_delay()
{
    int delay = 0xfffffff;
    while (delay)
        --delay;
}

void philosopher(void *arg)
{
    // 获取哲学家ID
    int id = *(int *)arg;
    int left = id;
    int right = (id + 1) % PHIL_COUNT;

    // 假设每个哲学家就餐2次
    for (int i = 0; i < 2; ++i)
    {
        short_delay();

        room_limit.P();
        forks[left].P();
        forks[right].P();

        eat_count[id]++;
        printf("Philosopher %d: start eating (%d)\n", id, eat_count[id]);
        short_delay();
        printf("Philosopher %d: stop  eating (%d)\n", id, eat_count[id]);

        forks[right].V();
        forks[left].V();
        room_limit.V();
    }
}

void first_thread(void *arg)
{
    // 第1个线程不可以返回
    stdio.moveCursor(0);
    for (int i = 0; i < 25 * 80; ++i)
    {
        stdio.print(' ');
    }
    stdio.moveCursor(0);

    for (int i = 0; i < PHIL_COUNT; ++i)
    {
        forks[i].initialize(1);
        eat_count[i] = 0;
    }

    // 就餐区最多允许 N-1位哲学家同时就餐
    room_limit.initialize(PHIL_COUNT - 1);

    int ids[PHIL_COUNT];
    for (int i = 0; i < PHIL_COUNT; ++i)
    {
        ids[i] = i;
        // 创建哲学家线程，传入哲学家ID
        programManager.executeThread(philosopher, &ids[i], "philosopher", 1);
    }

    asm_halt();
}
```

运行结果如下。

<img src="./images/OS_lab/lab6/philo.png" width="700px" />

死锁场景示例（可能发生）：
5位哲学家同时入座，每人先拿起左手边的叉子，再等待右手边的叉子。
此时每个人都占有一把叉子并等待另一把，形成循环等待，所有线程都阻塞，系统进入死锁。

死锁原因：
1) 互斥：叉子一次只能被一人使用。
2) 占有且等待：拿到左叉后等待右叉。
3) 不可剥夺：已经拿到的叉子不会被强制释放。
4) 循环等待：哲学家之间形成首尾相接的等待环。

解决死锁的一种方案：
引入“最多允许N-1位哲学家同时入座”的限制，即在拿叉子前先获取一个计数信号量(room_limit)，使得至少有一位哲学家无法进入就餐区，从而打破循环等待，避免死锁。当最多只有4位哲学家在竞争5根筷子时，根据鸽巢原理，必然至少有一位哲学家能够同时获得左右两根筷子并开始进餐。他进餐完毕后会释放筷子，从而打破了所有哲学家相互等待的僵局。


## 实验七-内存管理

### 任务一-二级分页机制

#### 代码复现

开启二级分页机制的三个步骤：

1. 规划页目录表和页表位置并初始化内容
2. 将页目录表和页表地址写入 cr3 寄存器
3. 将 cr0 的 PG 位设置为 1

综合上述3步骤后便可以开启分页机制，我们下面使用代码实现。

我们在内存管理器`MemoryManager`加入开启分页机制的成员函数声明，代码放在`include/memeory.h`中，如下所示。

```cpp
class MemoryManager
{
	...
    // 开启分页机制
    void openPageMechanism();

};
```

然后我们在`src/kernel/memory.cpp`中实现这个函数，如下所示。

```cpp
void MemoryManager::openPageMechanism()
{
    // 页目录表指针
    int *directory = (int *)PAGE_DIRECTORY;
    //线性地址0~4MB对应的页表
    int *page = (int *)(PAGE_DIRECTORY + PAGE_SIZE);

    // 初始化页目录表
    memset(directory, 0, PAGE_SIZE);
    // 初始化线性地址0~4MB对应的页表
    memset(page, 0, PAGE_SIZE);

    int address = 0;
    // 将线性地址0~1MB恒等映射到物理地址0~1MB
    for (int i = 0; i < 256; ++i)
    {
        // U/S = 1, R/W = 1, P = 1
        page[i] = address | 0x7;
        address += PAGE_SIZE;
    }

    // 初始化页目录项

    // 0~1MB
    directory[0] = ((int)page) | 0x07;
    // 3GB的内核空间
    directory[768] = directory[0];
    // 最后一个页目录项指向页目录表
    directory[1023] = ((int)directory) | 0x7;

    // 初始化cr3，cr0，开启分页机制
    asm_init_page_reg(directory);

    printf("open page mechanism\n");
    
}
```

其中，常量定义在`include/os_constant.h`下，如下所示。

```cpp
#define PAGE_DIRECTORY 0x100000
```

第4-11行，我们打算将页目录表放在`1MB`处。在开启分页机制前，我们需要建立好内核所在地址的页目录表和页表，否则一旦置PG位=1，开启分页机制后，CPU就会出现寻址异常。由于我们的内核很小，我们假设内核只会放在0\~1MB的内存区域。

第13-20行，为了访问方便，对于0\~1MB的内存区域我们建立的是虚拟地址到物理地址的恒等映射，也就是说，虚拟地址和物理地址相同。这个时候，我们就要设置相应的页目录项和页表项。

首先，虚拟地址0\~1MB的二进制表示是0x00000000\~0x000fffff，其31-22位均为0，对应第0个页目录项。因此我们只需要初始化第0个页目录项和其对应的页表即可。第0个页目录项被放在页目录表之后，地址是`PAGE_DIRECTORY + PAGE_SIZE`。然后我们取21\~12位，范围从0x000\~0xfff，涉及256个页表项。由于我们希望线性地址经过翻译后的物理地址依然和线性地址相同。因此，这256个页表项分别指向物理页的第0页，第1页和第255页。

除了设置页表项对应的物理页地址和固定为0的位外，我们设置U/S，R/W和P位为1。

第24-29行，我们初始化页目录项。由于我们的0\~1MB的线性地址对应于第0个页目录项，我们用刚刚放入了256个页表项的页表作为第0个页目录项指向的页表。同样地，我们设置U/S，R/W和P位为1。

后面我们设置第768个页目录项和第0个页表项相同、设置最后一个页目录项指向页目录表，这是用于构造页目录项和页表项的虚拟地址而服务的。至于具体的构造方法，我们留到后面再来学习。

第32行，我们将页目录表的地址放入cr3寄存器，然后将cr0的PG位置1便可开启分页机制，如下所示。

```asm
asm_init_page_reg:
    push ebp
    mov ebp, esp

    push eax

    mov eax, [ebp + 4 * 2]
    mov cr3, eax ; 放入页目录表地址
    mov eax, cr0
    or eax, 0x80000000
    mov cr0, eax           ; 置PG=1，开启分页机制

    pop eax
    pop ebp

    ret
```

置PG=1后，开启分页机制，函数如果能正确返回，则说明CPU能够正确使用分页机制来寻址。

最后，我们在`src/kernel/setup.cpp`中开启分页机制。

```cpp
extern "C" void setup_kernel()
{
	...
    // 内存管理器
    memoryManager.openPageMechanism();
    memoryManager.initialize();

    // 创建第一个线程
	...
}
```

将上述代码编译运行后，得到结果如下。

<img src="./images/OS_lab/lab7/level_two_pages.png" width="600px" />

### 任务二-内存分配算法

Best-fit 的核心思路是：遍历所有空闲块，选择其中大小 ≥ count 且最小的那个。这样能最大程度减少内部碎片，把较大的空闲块留给后续更大的请求。

以下是 BitMap::allocate 实现。

```cpp
int BitMap::allocate(const int count)
{
    if (count == 0)
        return -1;

    int index = 0;
    int best_start = -1;
    int best_size = length + 1;  // 初始化为不可能的大值

    // 第一遍扫描：遍历所有空闲块，记录最佳候选
    while (index < length)
    {
        // 越过已分配的资源
        while (index < length && get(index))
            ++index;

        if (index == length)
            break;

        // 找到一个空闲块，测量其连续长度
        int start = index;
        int free_size = 0;
        while (index < length && !get(index))
        {
            ++free_size;
            ++index;
        }

        // 该块够用 且 比当前最优更小 → 更新最优候选
        if (free_size >= count && free_size < best_size)
        {
            best_start = start;
            best_size = free_size;
        }
    }

    // 没有找到任何满足需求的块
    if (best_start == -1)
        return -1;

    // 在最优位置分配
    for (int i = 0; i < count; ++i)
    {
        set(best_start + i, true);
    }

    return best_start;
}
```

使用给定样例进行测试，结果如下。

<img src="./images/OS_lab/lab7/best_fit_allocate.png" width="600px" />

### 任务三-虚拟页内存管理

#### 虚拟页内存分配

**第一步，从虚拟地址池中分配若干连续的虚拟页**。虚拟页的分配通过函数`allocateVirtualPages`来实现，如下所示。

```cpp
int allocateVirtualPages(enum AddressPoolType type, const int count)
{
    int start = -1;

    if (type == AddressPoolType::KERNEL)
    {
        start = kernelVrirtual.allocate(count);
    }

    return (start == -1) ? 0 : start;
}
```

由于我们没有实现用户进程，此时能够分配页内存的地址池只有内核虚拟地址池。因此，对于其他类型的地址池，一律返回0，即分配失败。

**第二步，对每一个虚拟页，从物理地址池中分配1页**。物理页的分配通过函数`allocatePhysicalPages`来实现，这里便不再赘述。我们的物理地址池有两个，用户物理地址池和内核物理地址池，因此在分配物理页的时候也应该区别对待。

**第三步，为虚拟页建立页目录项和页表项，使虚拟页内的地址经过分页机制变换到物理页内**。建立虚拟页到物理页的映射关系通过函数`connectPhysicalVirtualPage`来实现，如下所示。

```cpp
bool MemoryManager::connectPhysicalVirtualPage(const int virtualAddress, const int physicalPageAddress)
{
    // 计算虚拟地址对应的页目录项和页表项
    int *pde = (int *)toPDE(virtualAddress);
    int *pte = (int *)toPTE(virtualAddress);

    // 页目录项无对应的页表，先分配一个页表
    if(!(*pde & 0x00000001)) 
    {
        // 从内核物理地址空间中分配一个页表
        int page = allocatePhysicalPages(AddressPoolType::KERNEL, 1);
        if (!page)
            return false;

        // 使页目录项指向页表
        *pde = page | 0x7;
        // 初始化页表
        char *pagePtr = (char *)(((int)pte) & 0xfffff000);
        memset(pagePtr, 0, PAGE_SIZE);
    }

    // 使页表项指向物理页
    *pte = physicalPageAddress | 0x7;

    return true;
}

```

#### 虚拟页内存释放

**第一步，对每一个虚拟页，释放为其分配的物理页**。首先，由于物理地址池存放的是物理地址，为了释放物理页，我们要找到虚拟页对应的物理页的物理地址，如下所示。

```cpp
int MemoryManager::vaddr2paddr(int vaddr)
{
    int *pte = (int *)toPTE(vaddr);
    int page = (*pte) & 0xfffff000;
    int offset = vaddr & 0xfff;
    return (page + offset);
}
```

根据分页机制，一个虚拟地址对应的物理页的地址是存放在页表项中的。因此，我们先求出虚拟地址的页表项的虚拟地址，然后访问页表项，取页表项内容的31-12位就是物理页的物理地址，最后替换虚拟地址的31-12位即可得到虚拟地址对应的物理地址。

然后我们释放物理页。释放了物理页后，我们就要将虚拟页对应的页表项置0，这是为了防止在虚拟页释放后被再次寻址。

**第二步，释放虚拟页**。释放虚拟页的函数如下所示。

```cpp
void MemoryManager::releaseVirtualPages(enum AddressPoolType type, const int vaddr, const int count)
{
    if (type == AddressPoolType::KERNEL)
    {
        kernelVirtual.release(vaddr, count);
    }
}
```

使用给定样例进行测试，结果如下。

```cpp
void first_thread(void *arg)
{
    char *p1 = (char *)memoryManager.allocatePages(AddressPoolType::KERNEL, 100);
    char *p2 = (char *)memoryManager.allocatePages(AddressPoolType::KERNEL, 10);
    char *p3 = (char *)memoryManager.allocatePages(AddressPoolType::KERNEL, 100);

    printf("%x %x %x\n", p1, p2, p3);

    memoryManager.releasePages(AddressPoolType::KERNEL, (int)p2, 10);
    p2 = (char *)memoryManager.allocatePages(AddressPoolType::KERNEL, 100);

    printf("%x\n", p2);

    p2 = (char *)memoryManager.allocatePages(AddressPoolType::KERNEL, 10);
    
    printf("%x\n", p2);

    asm_halt();
}
```

<img src="./images/OS_lab/lab7/virtual_page_manage.png" width="600px" />

可以成功分配，给内核空间分配到的地址确实属于内核虚拟地址空间的范围。也可以成功释放。

### 任务四-LRU算法模拟

以下是 LRU 算法的实现，使用 CPP 语言，简单进行模拟。

```cpp
class Node {
public:
    int key;
    int val;
    Node* next;
    Node* prev;
    Node(): key(0), val(0), next(nullptr), prev(nullptr) {}
    Node(int k, int v): key(k), val(v), next(nullptr), prev(nullptr) {}
};

class LRU {
public:

    int capacity;
    int count;
    Node* dummy_head;
    Node* dummy_tail;
    unordered_map<int, Node*> mp;

    LRU(int capacity): capacity(capacity), count(0) {
        this->dummy_head = new Node(0, 0);
        this->dummy_tail = new Node(-1,-1);
        dummy_head->next = dummy_tail;
        dummy_head->prev = nullptr;
        dummy_tail->next = nullptr;
        dummy_tail->prev = dummy_head;
    }
    
    int get(int key) {
        
        // 寻找目标节点
        Node* tar_node = nullptr;
        if (mp.find(key) == mp.end()) {
            return -1;
        } else {
            tar_node = mp[key];
            tar_node->next->prev = tar_node->prev;
            tar_node->prev->next = tar_node->next;
        }

        // 在dummy_head后面进行插入
        Node* head = dummy_head->next;
        dummy_head->next = tar_node;
        tar_node->prev = dummy_head;
        tar_node->next = head;
        head->prev = tar_node;

        return tar_node->val;
    }
    
    void put(int key, int value) {
        
        // 准备好要插入的结点
        Node* new_node = nullptr;
        if (mp.find(key) == mp.end()) {
            new_node = new Node(key, value);
            mp[key] = new_node;
            count++;
        } else {
            new_node = mp[key];
            new_node->val = value;
            new_node->next->prev = new_node->prev;
            new_node->prev->next = new_node->next;
        }

        // 在dummy_head后面进行插入
        Node* head = dummy_head->next;
        dummy_head->next = new_node;
        new_node->prev = dummy_head;
        new_node->next = head;
        head->prev = new_node;

        // 超出缓存容量上限则要删除
        if (count > capacity) {
            Node* rubbish = dummy_tail->prev;
            rubbish->prev->next = dummy_tail;
            dummy_tail->prev = rubbish->prev;
            mp.erase(rubbish->key);
            delete rubbish;
            count--;
        }
    }
};
```