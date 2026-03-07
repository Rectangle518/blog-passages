对于《并行程序设计导论》第三章的笔记：使用MPI进行分布式内存编程。

目录如下：
- [基础知识](#基础知识)
  - [分布式内存系统](#分布式内存系统)
  - [第一个MPI程序](#第一个mpi程序)
  - [常用函数](#常用函数)
    - [MPI\_Init](#mpi_init)
    - [MPI\_Finalize](#mpi_finalize)
    - [MPI\_Comm\_size](#mpi_comm_size)
    - [MPI\_Comm\_rank](#mpi_comm_rank)
    - [MPI\_Send](#mpi_send)
    - [MPI\_Recv](#mpi_recv)
  - [注意事项](#注意事项)
    - [是否阻塞](#是否阻塞)
    - [不可超越](#不可超越)
    - [局部变量](#局部变量)
- [集合通信](#集合通信)
  - [MPI\_Reduce](#mpi_reduce)
  - [MPI\_Allreduce](#mpi_allreduce)
  - [MPI\_Bcast](#mpi_bcast)
  - [MPI\_Scatter](#mpi_scatter)
  - [MPI\_Gather](#mpi_gather)
  - [MPI\_Allgather](#mpi_allgather)
- [派生数据类型](#派生数据类型)


## 基础知识

### 分布式内存系统

这是一个分布式内存系统：

<img src="./images/mpi_learn/分布式内存系统.png" width="600" />

在消息传递程序中，通常将运行在一个“核-内存”对上的程序称为一个**进程**。两个进程之间可以通过调用函数来进行通信。

### 第一个MPI程序

使用MPI写一段hello world程序：

```c
#include<stdio.h>
#include<string.h>
#include<mpi.h>

const int MAX_STRING = 100;

int main(void) {
    char greeting[MAX_STRING];
    int comm_sz; // 进程数
    int my_rank; // 进程编号

    MPI_Init(NULL, NULL);
    MPI_Comm_size(MPI_COMM_WORLD, &comm_sz);
    MPI_Comm_rank(MPI_COMM_WORLD, &my_rank);

    if (my_rank != 0) {
        sprintf(greeting, "Greetings from process %d of %d", my_rank, comm_sz);
        MPI_Send(greeting, strlen(greeting)+1, MPI_CHAR, 0, 0, MPI_COMM_WORLD);
    } else {
        printf("Greetings from process %d of %d\n", my_rank, comm_sz)
        for (int q = 1; q < comm_sz; q++) {
            MPI_Recv(greeting, MAX_STRING, MPI_CHAR, q, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
            printf("%s\n", greeting);
        }
    }

    MPI_Finalize();
    return 0;
}
```

其中，`<mpi.h>`头文件包括了MPI函数的原型、宏定义、类型定义等，还包括了编译MPI程序所需要的全部定义与声明。MPI定义的所有标识符都由`MPI_`开始，下划线后的第一个字母大写；MPI定义的宏与常量的所有字母都大写。

使用mpicc进行编译，然后使用mpiexec启动4个进程运行：
```bash
mpicc mpi_hello.c -o mpi_hello
mpiexec -np 4 ./mpi_hello

# 运行结果：
Greetings from process 0 of 4
Greetings from process 1 of 4
Greetings from process 2 of 4
Greetings from process 3 of 4
```

### 常用函数

接下来介绍一些函数。

#### MPI_Init
```c
int MPI_Init(
    int* argc_p
    char*** argv_p
);
```
参数`argc_p`和`argv_p`分别是指向`main`函数的参数`argc`和`argv`的指针。当程序不使用这些参数时，可以传入`NULL`。这个函数是为了告知MPI进行所有必要的初始化设置，例如为消息缓冲区分配存储空间、为进程指定进程号等。

#### MPI_Finalize
```c
int MPI_Finalize();
```
这个函数是为了告知MPI程序已经完成，可以释放所有资源。

#### MPI_Comm_size
```c
int MPI_Comm_size(
    MPI_Comm comm,
    int* comm_sz_p
);
```
参数`comm`是一个**通信子**，可取值为`MPI_COMM_WORLD`，表示用户启动的所有进程所组成的通信子。`size_p`是一个指向整数的指针，用于存储进程数。
> 通信子：一组可以互相发送信息的进程的集合。

#### MPI_Comm_rank
```c
int MPI_Comm_rank(
    MPI_Comm comm,
    int* my_rank_p
);
```
参数`comm`是一个通信子，`my_rank_p`是一个指向整数的指针，用于存储进程号。

#### MPI_Send
```c
int MPI_Send(
    void* msg_buf_p,
    int msg_size,
    MPI_Datatype msg_type,
    int dest,
    int tag,
    MPI_Comm communicator
);
```
参数`msg_buf_p`是指向包含消息内容的内存块的指针，`msg_size`是要发送的数据元素个数，`msg_type`是数据元素类型，`dest`是目标进程的进程号，`tag`是消息标签（用于区分消息），`comm`是通信子。

#### MPI_Recv
```c
int MPI_Recv(
    void* msg_buf_p,
    int buf_size,
    MPI_Datatype buf_type,
    int source,
    int tag,
    MPI_Comm communicator,
    MPI_Status* status_p
);
```
参数`msg_buf_p`是指向消息内容内存块的指针，`buf_size`是缓冲区大小，`buf_type`是数据元素类型，`source`是源进程的进程号，`tag`是消息标签，`comm`是通信子，`status_p`是一个指向`MPI_Status`结构体的指针。
> `status_p`参数的作用：在接收操作完成后，返回关于这条消息的元数据。特别是在使用`MPI_ANY_SOURCE`或`MPI_ANY_TAG`时，这个参数就是用来填补这些信息空白的。

### 注意事项

#### 是否阻塞

`MPI_Send`函数是否阻塞取决于MPI的具体实现。一个典型的实现方式是，规定一个消息的“截止”大小，如果消息大小超过这个值，则发送操作是阻塞的；否则，发送进程将缓冲这个消息，并使发送函数立刻返回。`MPI_Send`函数返回时，消息不一定已经到达接收方，但是发送缓冲区（即`msg_buf_p`所指的区域）已经可以安全复用。

相比之下，`MPI_Recv`函数是阻塞的，即接收进程会一直等待，直到收到一个匹配的消息。因此，需要注意可能出现的进程悬挂问题，即一个进程因为等不到匹配的消息而一直阻塞。

#### 不可超越

MPI要求消息是不可超越的，即如果进程p给进程r发送了两条消息，那么第一条消息必须在第二条消息之前可用。如果进程p和进程q都给进程r发送消息，那么这两条消息的顺序是不确定的。

#### 局部变量

在MPI编程中，局部变量是指**只在当前进程中有效**的变量。如果变量在所有进程中都有效，则称为全局变量。

## 集合通信

涉及通信子中所有进程的通信函数称为集合通信；而`MPI_Send`和`MPI_Recv`函数通常称为点对点通信，只涉及两个进程。

### MPI_Reduce

先介绍常用的`MPI_Reduce`函数：
```c
int MPI_Reduce(
    const void *input_data_p, // [In]  发送缓冲区地址（每个进程的数据）
    void *output_data_p,      // [Out] 接收缓冲区地址（仅在 root 进程有效）
    int count,                // [In]  每个进程中数据的个数
    MPI_Datatype datatype,    // [In]  数据类型 (如 MPI_INT, MPI_DOUBLE)
    MPI_Op operator,          // [In]  归约操作符 (如 MPI_SUM, MPI_MAX)
    int dest_process,         // [In]  根进程的 Rank ID (结果存到这里)
    MPI_Comm comm             // [In]  通信域 (如 MPI_COMM_WORLD)
);
```

使用示例：假设我们有 4 个进程，每个进程都有一个数字`local_value = rank + 1`，即 1, 2, 3, 4。我们要计算总和 (1+2+3+4=10)，并让进程 0 知道结果。

```c
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);

    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    // 1. 准备本地数据
    int local_value = rank + 1;
    int global_sum = 0; // 用于存储结果 (只在 root 进程有意义)

    printf("Process %d has value: %d\n", rank, local_value);

    // 2. 执行归约
    // 参数解释:
    // &local_value : 发送地址 (每个进程都有自己的值)
    // &global_sum  : 接收地址 (只有 root 会收到结果)
    // 1            : 数据个数 (1个整数)
    // MPI_INT      : 数据类型
    // MPI_SUM      : 操作 (求和)
    // 0            : Root 进程 ID (结果给进程 0)
    // MPI_COMM_WORLD : 通信域
    MPI_Reduce(&local_value, &global_sum, 1, MPI_INT, MPI_SUM, 0, MPI_COMM_WORLD);

    // 3. 检查并输出结果
    if (rank == 0) {
        printf("Root Process %d: The total sum is %d\n", rank, global_sum);
    } else {
        // 非 root 进程这里的 global_sum 未被 MPI 修改，保持初始值 0
        // 注意：不要依赖非 root 进程中的 output_data_p 内容
        printf("Process %d: I don't know the total sum.\n", rank);
    }

    MPI_Finalize();
    return 0;
}

// 运行结果：
// Process 0 has value: 1
// Process 1 has value: 2
// Process 2 has value: 3
// Process 3 has value: 4
// Root Process 0: The total sum is 10
// Process 1: I don't know the total sum.
// Process 2: I don't know the total sum.
// Process 3: I don't know the total sum.
```

一些注意事项：
- 在通信子的所有进程中必须调用相同的集合通信函数，且参数必须相容。
- 参数`output_data_p`只在用于接收的进程有效，其他进程的`output_data_p`值不会被使用，但仍然要传入。
- 集合通信函数不使用标签，而是根据通信子和调用的顺序来进行匹配。

### MPI_Allreduce

`MPI_Allreduce`函数与`MPI_Reduce`函数类似，但会将结果广播给所有进程。
参数列表也类似，只是去掉了`dest_process`参数，因为所有进程都会得到结果。
```c
int MPI_Allreduce(
    const void *input_data_p, // [In]  发送缓冲区地址（每个进程的数据）
    void *output_data_p,      // [Out] 接收缓冲区地址（仅在 root 进程有效）
    int count,                // [In]  每个进程中数据的个数
    MPI_Datatype datatype,    // [In]  数据类型 (如 MPI_INT, MPI_DOUBLE)
    MPI_Op operator,          // [In]  归约操作符 (如 MPI_SUM, MPI_MAX)
    MPI_Comm comm             // [In]  通信域 (如 MPI_COMM_WORLD)
);
```

### MPI_Bcast

`MPI_Bcast`函数用于广播数据，即一个进程将数据发送给通信子中的所有其他进程。
```c
int MPI_Bcast(
    void *data_p,           // [In/Out] 数据缓冲区地址
    int count,              // [In]  数据个数
    MPI_Datatype datatype,  // [In]  数据类型 (如 MPI_INT, MPI_DOUBLE)
    int source_proc,        // [In]  源进程的 Rank ID (谁发数据)
    MPI_Comm comm           // [In]  通信域 (如 MPI_COMM_WORLD)
);
```

### MPI_Scatter

`MPI_Scatter`函数是用于数据分发的集合通信函数，可以将根进程中的一个大数组切分成若干个小块，分别发送给不同的进程。
```c
int MPI_Scatter(
    const void *send_buf_p,    // [In]  发送缓冲区 (仅在 Root 进程有效)
    int send_count,            // [In]  发送给每个进程的数据个数 (注意：不是总数！)
    MPI_Datatype send_type,    // [In]  发送数据的类型
    void *recv_buf_p,          // [Out] 接收缓冲区 (所有进程都有效)
    int recv_count,            // [In]  接收的数据个数 (通常等于 send_count)
    MPI_Datatype recv_type,    // [In]  接收数据的类型 (通常等于 send_type)
    int src_proc,              // [In]  源进程的 Rank ID
    MPI_Comm comm              // [In]  通信域
);
```

注意：`MPI_Scatter`要求总数据量必须能被进程数整除，即每个进程分到的数据量必须相等。

### MPI_Gather

`MPI_Gather`函数是`MPI_Scatter`的逆操作，用于将多个进程的数据收集到根进程。它根据进程的`rank`，将分布在各个进程中的数据块拼接成一个大数组，然后发送给根进程。
```c
int MPI_Gather(
    void *send_buf_p,          // [In]  发送缓冲区地址 (所有进程有效)
    int send_count,            // [In]  每个进程发送的数据个数
    MPI_Datatype send_type,    // [In]  发送数据类型
    void *recv_buf_p,          // [Out] 接收缓冲区地址 (仅在 Root 进程有效)
    int recv_count,            // [In]  每个进程贡献的数据个数 (注意：不是总数！)
    MPI_Datatype recv_type,    // [In]  接收数据类型 (通常与 send_type 相同)
    int dest_proc,             // [In]  根进程 Rank ID (数据收集到这里)
    MPI_Comm comm              // [In]  通信域
);
```

### MPI_Allgather

`MPI_Allgather`函数是`MPI_Gather`的扩展，它将所有进程的数据收集到所有进程中。每个进程都会收到其他所有进程的数据。
```c
int MPI_Gather(
    void *send_buf_p,          // [In]  发送缓冲区地址 (所有进程有效)
    int send_count,            // [In]  每个进程发送的数据个数
    MPI_Datatype send_type,    // [In]  发送数据类型
    void *recv_buf_p,          // [Out] 接收缓冲区地址 (仅在 Root 进程有效)
    int recv_count,            // [In]  每个进程贡献的数据个数 (注意：不是总数！)
    MPI_Datatype recv_type,    // [In]  接收数据类型 (通常与 send_type 相同)
    MPI_Comm comm              // [In]  通信域
);
```

## 派生数据类型

在MPI中，发送和接收的时间开销往往很大，因此需要尽量减少通信次数，将信息一次发送而非分开发送。
MPI提供了三种手段来整合可能需要多条消息的数据：
1. 不同通信函数中的`count`参数；
2. 派生数据类型；
3. MPI中的`MPI_Pack`和`MPI_Unpack`函数。

MPI的基本数据类型假设数据在内存中是连续存放的，而我们实际上经常需要发送在内存中不连续的数据，这时就需要派生数据类型。

我们可以使用`MPI_Type_create_struct`函数来创建由基本数据类型所组成的派生数据类型，该函数可以指定数据的布局和偏移量。

```c
int MPI_Type_create_struct(
    int count,                          // 块的数量 (结构体成员个数)
    int array_of_blocklengths[],        // 每个块的元素个数
    MPI_Aint array_of_displacements[],  // 每个块的位移 (字节)
    MPI_Datatype array_of_types[],      // 每个块的基本类型
    MPI_Datatype *new_type_p            // 输出：新构造的类型
);
```

下面是派生数据类型的一个完整示例，它创建了一个包含三个成员的结构体类型，并使用`MPI_Send`和`MPI_Recv`函数进行发送和接收。

```c
#include <mpi.h>
#include <stdio.h>

typedef struct {
    int id;
    double x, y, z;
    char tag[10];
} Particle;

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    
    int rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    Particle my_particle = {101, 1.5, 2.5, 3.5, "test"};
    Particle recv_particle;

    if (rank == 0) {
        // 1. 准备参数
        int counts[3] = {1, 3, 10};
        
        MPI_Aint displacements[3];
        MPI_Datatype types[3] = {MPI_INT, MPI_DOUBLE, MPI_CHAR};

        // 2. 获取真实的内存位移 (使用 offsetof 或 MPI_Get_address)
        // 方法 A: 使用标准库 offsetof (需 #include <stddef.h>)
        // displacements[0] = offsetof(Particle, id);
        // displacements[1] = offsetof(Particle, x);
        // displacements[2] = offsetof(Particle, tag);
        
        // 方法 B: MPI 推荐方式 (更通用，处理复杂对象)
        Particle dummy;
        MPI_Get_address(&dummy.id, &displacements[0]);
        MPI_Get_address(&dummy.x, &displacements[1]);
        MPI_Get_address(&dummy.tag, &displacements[2]);

        // 转换为相对位移 (相对于第一个元素)
        MPI_Aint base_addr = displacements[0];
        for(int i=0; i<3; i++) {
            displacements[i] -= base_addr;
        }

        // 3. 创建类型
        MPI_Datatype particle_type;
        MPI_Type_create_struct(3, counts, displacements, types, &particle_type);

        // 4. 【重要】提交类型
        MPI_Type_commit(&particle_type);

        // 5. 发送 (count=1，因为我们要发一个完整的 Particle)
        MPI_Send(&my_particle, 1, particle_type, 1, 0, MPI_COMM_WORLD);

        // 6. 释放类型
        MPI_Type_free(&particle_type);
        
        printf("Rank 0 sent particle.\n");
    } 
    else if (rank == 1) {
        // 接收方也需要构造相同的类型 (或者直接用 MPI_BYTE 接收，但不推荐跨平台)
        // 为了演示，接收方也构造一次 (实际生产中通常会广播类型或硬编码)
        int counts[3] = {1, 3, 10};
        MPI_Aint displacements[3];
        MPI_Datatype types[3] = {MPI_INT, MPI_DOUBLE, MPI_CHAR};
        
        Particle dummy;
        MPI_Get_address(&dummy.id, &displacements[0]);
        MPI_Get_address(&dummy.x, &displacements[1]);
        MPI_Get_address(&dummy.tag, &displacements[2]);
        MPI_Aint base_addr = displacements[0];
        for(int i=0; i<3; i++) displacements[i] -= base_addr;

        MPI_Datatype particle_type;
        MPI_Type_create_struct(3, counts, displacements, types, &particle_type);
        MPI_Type_commit(&particle_type);

        // 接收
        MPI_Recv(&recv_particle, 1, particle_type, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        
        printf("Rank 1 received: ID=%d, Pos=(%f,%f,%f), Tag=%s\n", 
               recv_particle.id, recv_particle.x, recv_particle.y, recv_particle.z, recv_particle.tag);
        
        MPI_Type_free(&particle_type);
    }

    MPI_Finalize();
    return 0;
}
```