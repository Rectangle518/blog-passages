这里是对操作系统实验的过程记录，也是为了写实验报告而准备的一些笔记与草稿。

实验文档：https://gitee.com/chenzx67/sysu-2026-spring-operating-system/blob/main/README.md


## 实验二-实验入门

（！非常重要）先明确一下实验中需要理解的启动顺序：
加电 → BIOS → MBR → GRUB(bootloader) → 内核 → initramfs → 真正的根文件系统 → 系统启动完成


寄存器参考：

  | 0-31位 | 0-15位 | 8-15位 | 0-7位 |
  | ------ | ------ | ------ | ----- |
  | eax    | ax     | ah     | al    |
  | ebx    | bx     | bh     | bl    |
  | ecx    | cx     | ch     | cl    |
  | edx    | dx     | dh     | dl    |
  | esi    | si     | 无 | 无 |
  | edi    | di     | 无 | 无 |
  | esp    | sp     | 无 | 无 |
  | ebp    | bp     | 无 | 无 |

### 任务一-编写MBR程序

从 12*12 处开始输出我的学号，修改一个颜色之后，由于 12 * 80 + 12 = 972，据此确定输出位置。

```
org 0x7c00
[bits 16]
xor ax, ax ; eax = 0
; 初始化段寄存器, 段地址全部设为0
mov ds, ax
mov ss, ax
mov es, ax
mov fs, ax
mov gs, ax

; 初始化栈指针
mov sp, 0x7c00
mov ax, 0xb800
mov gs, ax

mov ah, 0x47
mov al, '2'
mov [gs:2 * 972], ax

mov al, '4'
mov [gs:2 * 973], ax

jmp $ ; 死循环

times 510 - ($ - $$) db 0
db 0x55, 0xaa
```

执行以下命令：

```bash
nasm -f bin mbr.asm -o mbr.bin
qemu-img create hd.img 10m
dd if=mbr.bin of=hd.img bs=512 count=1 seek=0 conv=notrunc
qemu-system-i386 -hda hd.img -serial null -parallel stdio 
```

<img src="./images/OS_lab/lab2/task1_学号.png" width="600px">

### 任务二-实模式中断

1. 设置光标位置为 (8, 8)

使用的是实模式中断int 10h，功能号ah设置为02h，设置行和列均为8，然后调用即可。

```
mov ah, 0x02
mov bh, 0x00
mov dh, 0x08
mov dl, 0x08

int 0x10
```

执行以下命令：

```bash
nasm -f bin task2_1.asm -o task2_1.bin
dd if=task2_1.bin of=hd.img bs=512 count=1 seek=0 conv=notrunc
qemu-system-i386 -hda hd.img -serial null -parallel stdio 
```

<img src="./images/OS_lab/lab2/task2_设置光标.png" width="600px">

2. 从 (8, 8) 开始打印学号

功能号ah设置为0x0e，显示一个字符后光标前移，写了一个循环来进行输出。

```
    mov ah, 0x02
    mov bh, 0x00
    mov dh, 0x08
    mov dl, 0x08

    int 0x10

    mov si, student_id

print_id_loop:

    mov al, [si]
    inc si
    cmp al, 0
    je print_id_done

    mov bh, 0x00
    mov bl, 0x0f
    mov cx, 0x01

    mov ah, 0x0e
    int 0x10

    jmp print_id_loop

print_id_done:

    jmp $ ; 死循环

student_id db '24325196', 0 ; 学号
```

执行以下命令：

```bash
nasm -f bin task2_2.asm -o task2_2.bin
dd if=task2_2.bin of=hd.img bs=512 count=1 seek=0 conv=notrunc
qemu-system-i386 -hda hd.img -serial null -parallel stdio 
```

<img src="./images/OS_lab/lab2/task2_输出学号.png" width="600px">

3. 实现键盘回显

调用int 16h的0号功能，读取键盘输入并放入al寄存器，并调用int 10h的0x0e进行显示并移动光标。

```
    mov ah, 0x02
    mov bh, 0x00
    mov dh, 0x00
    mov dl, 0x00

    int 0x10

input_loop:

    mov ah, 0x00
    int 0x16

    cmp al, 0x1b ; ESC
    je end_input

    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x0f
    int 0x10

    jmp input_loop

end_input:
    jmp $ ; 死循环
```

执行以下命令：

```bash
nasm -f bin task2_3.asm -o task2_3.bin
dd if=task2_3.bin of=hd.img bs=512 count=1 seek=0 conv=notrunc
qemu-system-i386 -hda hd.img -serial null -parallel stdio 
```

<img src="./images/OS_lab/lab2/task2_键盘回显.png" width="600px">

### 任务三-汇编

大体描述一下思路。对于if逻辑，将a1的值放入一个寄存器，根据多次比较的结果判断是否跳转即可；对于while逻辑，同样是根据比较的结果判断是否跳转，每次记得使循环变量自减；对于函数调用，主要注意压栈和出栈的处理。

先使用以下命令安装相应环境：

```bash
sudo apt install gcc-multilib g++-multilib
```

在assignment/student.asm中编写汇编代码，并使用`make run`命令运行：

```
%include "head.include"

    call your_if
    call your_while
    jmp student_function_end

your_if:
    mov eax, [a1]
    cmp eax, 12
    jl .case1 
    cmp eax, 24
    jl .case2
    shl eax, 4
    jmp .done
.case1:
    mov edx, 0
    mov ebx, 2
    idiv ebx
    inc eax
    jmp .done
.case2:
    mov ebx, 24
    sub ebx, eax
    imul eax, ebx
.done:
    mov [if_flag], eax
    ret

your_while:
.loop:
    mov eax, [a2]
    cmp eax, 12
    jl .end
    call my_random
    mov ecx, [a2]
    sub ecx, 12
    mov edx, [while_flag]
    mov [edx + ecx], al
    dec dword [a2]
    jmp .loop
.end:
    ret

%include "end.include"

your_function:
    push esi
    mov esi, [your_string]
.print_loop:
    xor eax, eax
    mov al, [esi]
    test al, al
    je .print_done
    push eax
    call print_a_char
    add esp, 4
    inc esi
    jmp .print_loop
.print_done:
    pop esi
    ret
```

<img src="./images/OS_lab/lab2/task3.png" width="600px">


## 实验三-从实模式到保护模式

### LBA方式读写硬盘

主硬盘分配的端口地址是0x1f0~0x1f7，这些端口的功能如下。

我们这里使用的是LBA28（28表示使用28位来表示逻辑扇区的编号）的方式读取硬盘。

| 端口地址 | 功能 |
| :---: | :---: |
| 0x1f0 | 数据端口，有16位 |
| 0x1f1 | 错误寄存器，记录错误类型 |
| 0x1f2 | 读取的扇区数量 |
| 0x1f3 | 逻辑扇区的0~7位 |
| 0x1f4 | 逻辑扇区的8~15位 |
| 0x1f5 | 逻辑扇区的16~23位 |
| 0x1f6 | 7位和5位为1，6位决定CHS/LBA，4位决定主/从硬盘，低4位为逻辑扇区的最后4位 |
| 0x1f7 | 状态/命令寄存器，写入0x20请求硬盘读，随后可以读取状态 |

### 保护模式

在80286及以后，保护模式的引入使得内存地址改为32位，程序至少可以访问到 4GB 的内存空间。保护模式的段寄存器中存储的是段选择子，段选择子中存储的是段描述符在GDT（一个段描述符数组）的索引，段描述符中存储的是段的基地址、段界限、段属性等信息。

在BIOS加电启动后，我们需要在实模式下的MBR中编写代码加载bootloader，然后在bootloader中实现从实模式到保护模式的跳转。

### 任务一-加载bootloader

#### 复现例一

使用已经提供好的 bootloader.asm 和 mbr.asm，以及 makefile 文件进行编译，生成 bootloader.bin 和 mbr.bin，并将 mbr 和 bootloader 分别写入对应的扇区，然后使用 qemu 进行加载运行。

makefile 文件内容如下，可以编译并写入镜像，然后执行。

```makefile
run:
	@qemu-system-i386 -hda hd.img -serial null -parallel stdio 
build:
	@nasm -f bin mbr.asm -o mbr.bin
	@nasm -f bin bootloader.asm -o bootloader.bin
	@dd if=mbr.bin of=hd.img bs=512 count=1 seek=0 conv=notrunc
	@dd if=bootloader.bin of=hd.img bs=512 count=5 seek=1 conv=notrunc
clean:
	@rm *.bin
```

```bash
执行：
make build
make run
```

<img src="./images/OS_lab/lab3/example-1.png" width="600px" />

#### 改用CHS读取

先介绍一下从 LBA 到 CHS 的转换公式。
设 H 为总磁头数，S 为每磁道扇区数，LBA 为逻辑块地址，则：
(1) 柱面号(C) = LBA // (S * H)
(2) 磁头号(H) = (LBA % (S * H)) // S
(3) 扇区号(S) = (LBA % S) + 1

修改 mbr.asm 文件的 asm_read_hard_disk 函数，使用 CHS 读取扇区，修改后的代码如下。

```asm
asm_read_hard_disk:                           
; 从硬盘读取一个逻辑扇区

; 参数列表
; ax=逻辑扇区号0~15位
; cx=逻辑扇区号16~28位
; ds:bx=读取出的数据放入地址

; 返回值
; bx=bx+512

    mov ch, 0x00 ; 柱面号是0
    mov dh, 0x00 ; 磁头号是0
    mov cl, al   
    inc cl       ; cl=扇区号
    mov dl, 0x80 ; 驱动器号
    push ax      ; 保存寄存器ax
    mov al, 0x01 ; 读取1个扇区

    mov ah, 0x02 ; BIOS中断13h的功能号
    int 0x13     ; 调用BIOS中断
    add bx, 512
    pop ax       ; 恢复寄存器ax
    ret
```

结果输出与例一相同，这里不再重复贴图。

### 任务二-进入保护模式

利用已经提供好的 bootloader.asm 和 mbr.asm 文件，包括 makefile 和 gdbinit 文件等。执行 make build 命令进行编译，然后执行 make debug 命令启动 gdb 进行调试。

其中，makefile 内容如下。
在 build 命令下，将 bootloader.asm 和 mbr.asm 分别编译成一个可重定位文件，并使用 -g 参数加上 debug 信息；然后为可重定位文件指定起始地址，分别链接生成可执行文件 xxx.symbol 和 xxx.bin 文件。
在 debug 命令下，启动 qemu 进行调试，间隔一秒之后在另一个终端窗口启动 gdb 进行调试（会执行 gdbinit 文件）。

```
run:
	@qemu-system-i386 -hda hd.img -serial null -parallel stdio 
debug:
	@qemu-system-i386 -s -S -hda hd.img -serial null -parallel stdio &
	@sleep 1
	@gnome-terminal -e "gdb -q -x gdbinit"
build:
	@nasm -g -f elf32 mbr.asm -o mbr.o
	@ld -o mbr.symbol -melf_i386 -N mbr.o -Ttext 0x7c00
	@ld -o mbr.bin -melf_i386 -N mbr.o -Ttext 0x7c00 --oformat binary

	@nasm -g -f elf32 bootloader.asm -o bootloader.o
	@ld -o bootloader.symbol -melf_i386 -N bootloader.o -Ttext 0x7e00
	@ld -o bootloader.bin -melf_i386 -N bootloader.o -Ttext 0x7e00 --oformat binary

	@dd if=mbr.bin of=hd.img bs=512 count=1 seek=0 conv=notrunc
	@dd if=bootloader.bin of=hd.img bs=512 count=5 seek=1 conv=notrunc
clean:
	@rm -fr *.bin *.o *.symbol
```

接下来在 gdb 中进行调试，依次验证进入保护模式的四个步骤。

步骤一：准备 GDT 并用 lgdt 指令加载 GDTR 信息

<img src="./images/OS_lab/lab3/GDT内容.png" width="500px" />

查看地址 0x8800 附近的内容，可以看到 GDT 的内容。

步骤二：开启第21根地址线（A20）

<img src="./images/OS_lab/lab3/开启A20地址线.png" width="500px" />

在图中可以看到 A20=1 输出，表明第21根地址线已经成功开启。

步骤三：开启cr0的保护模式标志位

<img src="./images/OS_lab/lab3/cr0寄存器.png" width="500px" />

可以看到 cr0 寄存器的值为 0x00000011，最低位是1，表明保护模式标志位开启。

步骤四：远跳转进入保护模式

此时，jmp 指令将 CODE_SELECTOR 送入 cs 寄存器，将 protect_mode_begin + LOADER_START_ADDRESS 送入 eip 寄存器，进入保护模式。

<img src="./images/OS_lab/lab3/远跳转.png" width="300px" />


## 实验四-中断

### 任务一-C/C++与汇编混合编程

C/C++函数调用规则：
- 如果函数有参数，那么参数从右向左依次入栈。
- 如果函数有返回值，返回值放在eax中。
- 放置于栈的参数一般使用ebp来获取。

#### 汇编语言调用C函数

声明方式：
```
// 在汇编代码中声明这个函数来自外部
extern function_from_C
extern function_from_Cpp

// 如果是C++，需要加上C关键字
extern "C" void function_from_Cpp();
```

当我们需要在汇编代码中调用函数 `function_from_C`，调用的形式是 `function_from_C(1,2)`，此时的汇编代码如下。

```asm
push 2         ; arg2
push 1         ; arg1
call function_from_C 
add esp, 8      ; 清除栈上的参数
; call指令返回后，函数的返回值被放在了eax中
```

#### C调用汇编函数

声明方式：

```
// 在汇编代码中声明某个函数为global
global function_from_asm

// 在C代码中声明这个函数来自外部
extern void function_from_asm();

// 如果是C++，需要加上C关键字
extern "C" void function_from_asm();
```

当我们需要在C代码中调用函数 `function_from_asm` 时，使用 `int ret = function_from_asm(1,2);` 即可。

此时，我们实现的汇编函数 `function_from_asm` 必须要遵循C/C++的函数调用规则才可以被正常调用。
一个遵循了C/C++的函数调用规则的汇编函数如下所示。

```asm
function_from_asm:
	push ebp
	mov ebp, esp
	
	; 下面通过ebp引用函数参数
	; [ebp + 4 * 0]是之前压入的ebp值
	; [ebp + 4 * 1]是返回地址
	; [ebp + 4 * 2]是arg1
	; [ebp + 4 * 3]是arg2
	; 返回值需要放在eax中
	
	... 
	
	pop ebp
	ret
```

调用者按照 从右到左 的顺序压入参数：
​push 2​ (参数 arg2)
​push 1​ (参数 arg1)

然后执行 `​call function_from_asm`​。
call 指令会自动将 返回地址（即 call 指令下一条指令的地址）压入栈中。
此时，在进入被调用函数 `function_from_asm` 之前，栈顶指针 esp 指向刚刚压入的 返回地址。

#### 运行结果

使用 makefile 编译运行，结果如下。

<img src="./images/OS_lab/lab4/C与汇编混合编程.png" width="500px" />


### 任务二-内核的加载

文件目录如下：

```
├── build
│   └── makefile
├── include
│   ├── asm_utils.h
│   ├── boot.inc
│   ├── os_type.h
│   └── setup.h
├── run
│   ├── gdbinit
│   └── hd.img
└── src
    ├── boot
    │   ├── bootloader.asm
    │   ├── entry.asm
    │   └── mbr.asm
    ├── kernel
    │   └── setup.cpp
    └── utils
        └── asm_utils.asm
```

修改 `asm_utils.asm` 代码，输出我的学号。

进入 build 目录，利用已有的 makefile 进行编译并运行，结果如下。

<img src="./images/OS_lab/lab4/加载内核输出学号.png" width="800px" />

### 任务三-中断的处理

先梳理一下执行过程，由 mbr 加载 bootloader，bootloader 加载内核，内核执行 setup.cpp 中的 `setup_kernel()` 方法，`setup_kernel()` 会调用 `interruptManager.initialize()` 方法，在这个方法中先设置IDTR，然后再初始化256个中断描述符，将每个中断的处理函数都指向我们自定义的默认处理函数。随后 `setup_kernel()` 方法会触发一个除0异常，此时会调用我们自定义的默认处理函数。

<img src="./images/OS_lab/lab4/触发未处理的中断.png" width="800px" />

```cpp
void InterruptManager::initialize()
{
    // 初始化IDT
    IDT = (uint32 *)IDT_START_ADDRESS;

	// 设置IDTR
    asm_lidt(IDT_START_ADDRESS, 256 * 8 - 1);

    for (uint i = 0; i < 256; ++i)
    {
		// 将每个中断的处理函数都指向我们自定义的默认处理函数
        setInterruptDescriptor(i, (uint32)asm_interrupt_empty_handler, 0);
    }
}

void InterruptManager::setInterruptDescriptor(uint32 index, uint32 address, byte DPL)
{
    // 中断描述符的低32位
    IDT[index * 2] = (CODE_SELECTOR << 16) | (address & 0xffff);
    // 中断描述符的高32位
    IDT[index * 2 + 1] = (address & 0xffff0000) | (0x1 << 15) | (DPL << 13) | (0xe << 8);
}
```

以下是 `asm_lidt` 的代码。

```asm
asm_lidt:
    push ebp                ; 保存旧的基址指针
    mov ebp, esp            ; 设置新的栈帧基址
    push eax                ; 保存eax寄存器，遵循调用约定
    
    ; 获取limit参数（第2个参数）
    mov eax, [ebp + 4 * 3]  ; ebp+12：第3个32位参数（limit）
    mov [ASM_IDTR], ax      ; 将limit存入ASM_IDTR的低16位
    
    ; 获取start参数（第1个参数）
    mov eax, [ebp + 4 * 2]  ; ebp+8：第2个32位参数（start）
    mov [ASM_IDTR + 2], eax ; 将start存入ASM_IDTR的偏移2处（16位对齐）
    
    ; 执行LIDT指令
    lidt [ASM_IDTR]         ; 从ASM_IDTR加载到IDTR寄存器
    
    pop eax                 ; 恢复eax寄存器
    pop ebp                 ; 恢复旧的基址指针
    ret                     ; 返回调用者

ASM_IDTR dw 0               ; 定义16位的limit字段
         dd 0               ; 定义32位的base字段

```

以下是中断处理函数的代码。

```asm
; 定义错误信息字符串
ASM_UNHANDLED_INTERRUPT_INFO db 'Unhandled interrupt happened, halt...'
                             db 0  ; 字符串结束符（NULL）

; 自定义的默认处理函数
asm_unhandled_interrupt:
    cli                     ; 清除中断标志，禁用中断
    
    ; 设置字符串输出参数
    mov esi, ASM_UNHANDLED_INTERRUPT_INFO  ; esi指向字符串首地址
    xor ebx, ebx            ; ebx清零，用作屏幕显示位置偏移（列号×2）
    mov ah, 0x03            ; 设置显示属性：黑底青字（文本模式属性）
    
.output_information:
    ; 检查是否到达字符串结尾
    cmp byte[esi], 0        ; 比较当前字符是否为0（字符串结束符）
    je .end                 ; 如果是0，跳转到结束
    
    ; 输出一个字符到屏幕
    mov al, byte[esi]       ; al = 当前字符
    mov word[gs:bx], ax     ; 将字符和属性写入显存
                           ; gs:bx指向显存位置，ax=属性(ah)+字符(al)
    
    ; 准备下一个字符
    inc esi                 ; 指向下一个字符
    add ebx, 2              ; 显存位置前进2字节（字符+属性）
    jmp .output_information ; 继续输出下一个字符
    
.end:
    jmp $                   ; 无限循环，系统挂起

```

接下来自定义一个除0异常的处理函数，输出一个字符串。
首先在 `asm_utils.h` 中声明并编写这个函数，定义好这个函数所要输出的字符串，将这个函数声明为global。

```asm
global asm_divide_by_zero

ASM_DIVIDE_BY_ZERO_INFO db 'Divide by zero happened   --luyy86'
                        db 0

; void asm_divide_by_zero()
asm_divide_by_zero:
    cli
    mov esi, ASM_DIVIDE_BY_ZERO_INFO
    xor ebx, ebx
    mov ah, 0x03
.output_information:
    cmp byte[esi], 0
    je .end
    mov al, byte[esi]
    mov word[gs:bx], ax
    inc esi
    add ebx, 2
    jmp .output_information
.end:
    jmp $
```

接下来在 `interrupt.cpp` 中设置除0异常的处理函数。具体来讲，修改 `interruptManager.initialize()` 方法，由于除0异常的中断号为0，所以将第0个中断描述符的地址设置为 `asm_divide_by_zero`。

```cpp
void InterruptManager::initialize()
{
    // 初始化IDT
    IDT = (uint32 *)IDT_START_ADDRESS;
    asm_lidt(IDT_START_ADDRESS, 256 * 8 - 1);

	// 将除0异常的处理函数设置为 asm_divide_by_zero
    setInterruptDescriptor(0, (uint32)asm_divide_by_zero, 0);

	// for循环的起始值改为 1
    for (uint i = 1; i < 256; ++i)
    {
        setInterruptDescriptor(i, (uint32)asm_unhandled_interrupt, 0);
    }

}
```

最后在 `asm_utils.h` 中添加声明。

```cpp
extern "C" void asm_divide_by_zero();
```

<img src="./images/OS_lab/lab4/自定义除0异常函数.png" width="800px" />

### 任务四-时钟中断

修改 `interrupt.cpp` 中的 `c_time_interrupt_handler()` 时钟中断处理函数，在每次时钟中断时，输出的字符串相比于上次输出时，向右移动一个字符位置，当字符串到达屏幕最右侧时，换行并重新从屏幕最左侧开始输出。

```cpp
extern "C" void c_time_interrupt_handler()
{
    static int position = 0;
    const char* str = "24325196     luyy86     ";
    int str_len = 24; // 字符串长度
    
    // 清空屏幕
    for (int i = 0; i < 80; ++i)
    {
        stdio.print(0, i, ' ', 0x07);
    }

    // 移动光标到(0,0)
    stdio.moveCursor(0);
    
    // 显示跑马灯效果
    for (int i = 0; i < 80; ++i)
    {
        // 计算当前要显示的字符位置
        int char_pos = (position + i) % str_len;
        stdio.print(str[char_pos]);
    }
    
    // 更新位置，实现滚动效果
    position = (position + 1) % str_len;
}
```

<img src="./images/OS_lab/lab4/跑马灯效果.png" width="800px" />
