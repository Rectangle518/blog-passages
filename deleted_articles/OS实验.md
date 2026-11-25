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


## 实验五-内核线程

### 任务一-实现printf

首先学习如何在函数内部引用可变参数列表中的参数。

为了引用可变参数列表中的参数，我们需要用到`<stdarg.h>`头文件定义的一个变量类型`va_list`和三个宏`va_start`，`va_arg`，`va_end`，这三个宏用于获取可变参数列表中的参数，用法如下。

| 宏                                    | 用法说明                                                     |
| ------------------------------------- | ------------------------------------------------------------ |
| `va_list`                             | 定义一个指向可变参数列表的指针。                             |
| `void va_start(va_list ap, last_arg)` | 初始化可变参数列表指针`ap`，使其指向可变参数列表的起始位置，即函数的固定参数列表的最后一个参数`last_arg`的后面第一个参数。 |
| `type va_arg(va_list ap, type)`       | 以类型`type`返回可变参数，并使`ap`指向下一个参数。           |
| `void va_end(va_list ap)`             | 清零`ap`。      |

现在来实现一个`print_any_number_of_integers`，这是一个用来输出若干个整数的函数。函数的参数分为两部分，n 是可变参数的数量，...是可变参数，表示若干个待输出的整数。

```cpp
#include <iostream>
#include <stdarg.h>

void print_any_number_of_integers(int n, ...);

int main()
{
    print_any_number_of_integers(1, 213);
    print_any_number_of_integers(2, 234, 2567);
    print_any_number_of_integers(3, 487, -12, 0);
}

void print_any_number_of_integers(int n, ...)
{
    // 定义一个指向可变参数的指针parameter
    va_list parameter;
    // 使用固定参数列表的最后一个参数来初始化parameter
    // parameter指向可变参数列表的第一个参数
    va_start(parameter, n);

    for ( int i = 0; i < n; ++i ) {
        // 引用parameter指向的int参数，并使parameter指向下一个参数
        std::cout << va_arg(parameter, int) << " ";
    }
    
    // 清零parameter
    va_end(parameter);

    std::cout << std::endl;
}
```

这里考虑一下这几个宏的实现。

```cpp
// 计算类型 n 的大小，并向上取整到 sizeof(int) 的整数倍
#define _INTSIZEOF(n) ((sizeof(n) + sizeof(int) - 1) & ~(sizeof(int) - 1))

// 将 ap 指针指向可变参数列表的第一个参数
#define va_start(ap, v) (ap = (va_list)&v + _INTSIZEOF(v))

// 返回ap指向的，type类型的变量，并同时使ap指向下一个参数
#define va_arg(ap, type) (*(type *)((ap += _INTSIZEOF(type)) - _INTSIZEOF(type)))

// 清零 ap 指针
#define va_end(ap) (ap = (va_list)0)
```

接下来我们就可以开始实现`printf`函数了。`printf`函数的参数分为两部分，fmt 是格式化字符串，...是可变参数，表示若干个待输出的整数。`printf`首先找到fmt中的形如`%c,%d,%x,%s`对应的参数，然后用这些参数具体的值来替换`%c,%d,%x,%s`等，得到一个新的格式化输出字符串，这个过程称为fmt的解析。最后，printf将这个新的格式化输出字符串即可。然而，这个字符串可能非常大，会超过函数调用栈的大小。实际上，我们会定义一个缓冲区，然后对fmt进行逐字符地解析，将结果逐字符的放到缓冲区中。放入一个字符后，我们会检查缓冲区，如果缓冲区已满，则将其输出，然后清空缓冲区，否则不做处理。

在实现printf前，我们需要一个能够输出字符串的函数，这个函数能够正确处理字符串中的`\n`换行字符。

```cpp
int STDIO::print(const char *const str)
{
    int i = 0;

    for (i = 0; str[i]; ++i)
    {
        switch (str[i])
        {
        // 处理换行符
        case '\n':
            uint row;
            row = getCursor() / 80;
            // 如果当前行是最后一行，则向上滚动
            if (row == 24)
            {
                rollUp();
            }
            // 否则，光标移动到下一行
            else
            {
                ++row;
            }
            moveCursor(row * 80);
            break;
        // 处理其他字符：直接输出
        default:
            print(str[i]);
            break;
        }
    }

    return i;
}
```

我们实现的printf比较简单，只能解析如下参数。

| 符号 | 含义             |
| ---- | ---------------- |
| %d   | 按十进制整数输出 |
| %c   | 输出一个字符     |
| %s   | 输出一个字符串   |
| %x   | 按16进制输出     |

按照前面描述的过程，printf的实现如下。

```cpp
int printf(const char *const fmt, ...)
{
    // 缓冲区配置
    const int BUF_LEN = 32;          // 缓冲区大小，选择32字节是为了减少内核内存使用
    char buffer[BUF_LEN + 1];        // 主输出缓冲区，+1为字符串终止符'\0'预留空间
    char number[33];                 // 数字转换缓冲区，33字节足够存储32位整数的任何进制表示
    
    // 局部变量声明
    int idx;                         // 缓冲区当前写入位置索引（0 ~ BUF_LEN-1）
    int counter;                     // 已成功输出的字符计数器
    va_list ap;                      // 可变参数列表指针
    
    // 初始化可变参数访问
    va_start(ap, fmt);               // 初始化ap，使其指向fmt后的第一个可变参数
    idx = 0;                         // 从缓冲区起始位置开始
    counter = 0;                     // 输出字符数初始为0
    
    // 主解析循环：逐字符处理格式字符串
    for (int i = 0; fmt[i]; ++i)     // 遍历格式字符串直到遇到'\0'
    {
        // 情况1：普通字符（非格式说明符）
        if (fmt[i] != '%')
        {
            // 将普通字符添加到缓冲区，如果缓冲区满则自动输出
            counter += printf_add_to_buffer(buffer, fmt[i], idx, BUF_LEN);
        }
        // 情况2：遇到格式说明符起始字符'%'
        else
        {
            i++;                     // 跳过'%'，查看下一个字符确定格式类型
            
            // 边界检查：如果'%'是字符串的最后一个字符，则终止解析
            if (fmt[i] == '\0')
            {
                break;               // 格式字符串意外结束，退出循环
            }
            
            // 根据格式字符类型进行相应处理
            switch (fmt[i])
            {
            case '%':                // 转义情况：输出一个'%'字符
                counter += printf_add_to_buffer(buffer, fmt[i], idx, BUF_LEN);
                break;
                
            case 'c':                // 字符输出：%c
                // 从可变参数列表中获取一个字符（C中字符参数被提升为int）
                counter += printf_add_to_buffer(buffer, va_arg(ap, int), idx, BUF_LEN);
                break;
                
            case 's':                // 字符串输出：%s
                // 特殊处理：先输出缓冲区中已有的内容
                buffer[idx] = '\0';   // 终止当前缓冲区内容
                idx = 0;              // 重置缓冲区索引
                counter += stdio.print(buffer);  // 输出缓冲区内容
                
                // 直接输出整个字符串（可能很长，超过缓冲区大小）
                counter += stdio.print(va_arg(ap, const char *));
                break;
                
            case 'd':                // 十进制整数输出：%d
            case 'x':                // 十六进制整数输出：%x
            {
                // 从可变参数列表中获取一个整数
                int temp = va_arg(ap, int);
                
                // 处理负数（仅对十进制格式）
                if (temp < 0 && fmt[i] == 'd')
                {
                    // 输出负号
                    counter += printf_add_to_buffer(buffer, '-', idx, BUF_LEN);
                    temp = -temp;    // 转换为正数以便后续处理
                }
                
                // 将整数转换为字符串表示
                // itos函数：integer to string，返回转换后字符串的长度
                // 参数：number-输出缓冲区，temp-要转换的整数，进制（10或16）
                temp = itos(number, temp, (fmt[i] == 'd' ? 10 : 16));
                
                // 逐个输出数字字符
                for (int j = 0; number[j]; ++j)
                {
                    counter += printf_add_to_buffer(buffer, number[j], idx, BUF_LEN);
                }
                break;
            }
            
            // 注意：此处没有default分支，不支持其他格式字符
            }
        }
    }
    
    // 循环结束后处理：输出缓冲区中剩余的内容
    buffer[idx] = '\0';               // 终止缓冲区字符串
    counter += stdio.print(buffer);   // 输出剩余内容
    
    // 返回总共输出的字符数
    return counter;
}

```

接下来测试这个printf函数。

```cpp
#include "asm_utils.h"
#include "interrupt.h"
#include "stdio.h"

// 屏幕IO处理器
STDIO stdio;
// 中断管理器
InterruptManager interruptManager;


extern "C" void setup_kernel()
{
    // 中断处理部件
    interruptManager.initialize();
    // 屏幕IO处理部件
    stdio.initialize();
    interruptManager.enableTimeInterrupt();
    interruptManager.setTimeInterrupt((void *)asm_time_interrupt_handler);
    //asm_enable_interrupt();
    printf("print percentage: %%\n"
           "print char \"N\": %c\n"
           "print string \"Hello World!\": %s\n"
           "print decimal: \"-1234\": %d\n"
           "print hexadecimal \"0x7abcdef0\": %x\n",
           'N', "Hello World!", -1234, 0x7abcdef0);
    //uint a = 1 / 0;
    asm_halt();
}
```

<img src="./images/OS_lab/lab5/初步实现的printf.png" width="800px" />

接下来在当前的 `printf` 函数中添加对科学计数法的格式化输出支持。在之前的代码中添加一个 case 分支来处理科学计数法格式化输出。

```cpp
case 'e':
    temp = va_arg(ap, int);

    if (temp < 0)
    {
        counter += printf_add_to_buffer(buffer, '-', idx, BUF_LEN);
        temp = -temp;
    }

    // 将整数用科学计数法表示
    int_to_scientific(number, temp);

    for (int j = 0; number[j]; ++j)
    {
        counter += printf_add_to_buffer(buffer, number[j], idx, BUF_LEN);
    }
    break;
```

然后实现 `int_to_scientific` 函数，将整数转换为科学计数法表示的字符串。

```cpp
void int_to_scientific(char *number, int num) {
    // 处理0的特殊情况
    if (num == 0) {
        number[0] = '0';
        number[1] = 'e';
        number[2] = '0';
        number[3] = '\0';
        return;
    }

    int idx = 0;
    int exponent = 0;
    int temp = num;
    int divisor = 1;

    // 计算数字位数和对应的除数
    while (temp >= 10) {
        temp /= 10;
        divisor *= 10;
        exponent++;
    }

    // 写入整数部分（第一位）
    temp = num / divisor;
    number[idx++] = temp + '0';
    number[idx++] = '.';

    // 写入小数部分（剩余数字）
    temp = num;
    int current_divisor = divisor;
    while (current_divisor >= 1) {
        int digit = (temp / current_divisor) % 10;
        if (current_divisor != divisor) {  // 跳过第一位数字
            number[idx++] = digit + '0';
        }
        current_divisor /= 10;
    }

    // 写入指数部分
    number[idx++] = 'e';
    number[idx++] = '+';
    
    // 写入指数值
    if (exponent >= 10) {
        number[idx++] = (exponent / 10) + '0';
        number[idx++] = (exponent % 10) + '0';
    } else {
        number[idx++] = exponent + '0';
    }

    number[idx] = '\0';
}
```

用同样的方式进行测试，结果如图。

<img src="./images/OS_lab/lab5/实现科学计数法.png" width="800px" />

### 任务二-线程的实现

#### 用户线程与内核线程

用户线程缺点：
- 单线程阻塞导致整个进程挂起；
- 调度器只能感知进程级，时钟中断无法影响线程级执行流；
- 进程内时间片再分配导致效率抵消。

内核线程优点：
- 线程与进程同等被调度，多线程进程获得更多处理器资源（例：4线程+1进程=5个独立执行流，4线程进程占80%的CPU）
- 单线程阻塞不影响同进程其他线程
- 系统调用开销相比提速可忽略

#### 线程的描述

我们创建的线程的状态有5个，分别是创建态、运行态、就绪态、阻塞态和终止态。我们使用一个枚举类型`ProgramStatus`来描述线程的5个状态。

```c++
enum ProgramStatus
{
    CREATED,
    RUNNING,
    READY,
    BLOCKED,
    DEAD
};
```

线程的组成部分线程各自的栈，状态，优先级，运行时间，线程负责运行的函数，函数的参数等，这些组成部分被集中保存在一个结构中——PCB(Process Control Block)，如下所示。

```cpp
struct PCB
{
    int *stack;                      // 栈指针，用于调度时保存esp
    char name[MAX_PROGRAM_NAME + 1]; // 线程名
    enum ProgramStatus status;       // 线程的状态
    int priority;                    // 线程优先级
    int pid;                         // 线程pid
    int ticks;                       // 线程时间片总时间
    int ticksPassedBy;               // 线程已执行时间
    ListItem tagInGeneralList;       // 线程队列标识
    ListItem tagInAllList;           // 线程队列标识
};

struct ListItem
{
    ListItem *previous;
    ListItem *next;
};
```

#### PCB的分配

声明一个程序管理类`ProgramManager`，用于线程和进程的创建和管理。

```cpp
class ProgramManager
{
    // ...
};
```

在创建线程之前，我们需要向内存申请一个PCB。我们将一个PCB的大小设置为4096个字节，也就是一个页的大小。本来我们PCB的分配是通过页内存管理来实现的，类似于`malloc`和`free`。但是，我们并没有实现基于二级分页机制的内存管理，或者说我们现在还没有引入内存分页的概念。为了解决PCB的内存分配问题，我们实际上是在内存中预留了若干个PCB的内存空间来存放和管理PCB，如下所示。

```cpp
// PCB的大小，4KB。
const int PCB_SIZE = 4096;         
// 存放PCB的数组，预留了MAX_PROGRAM_AMOUNT个PCB的大小空间。
char PCB_SET[PCB_SIZE * MAX_PROGRAM_AMOUNT]; 
// PCB的分配状态，true表示已经分配，false表示未分配。
bool PCB_SET_STATUS[MAX_PROGRAM_AMOUNT];     

// 分配一个PCB
PCB *ProgramManager::allocatePCB()
{
    for (int i = 0; i < MAX_PROGRAM_AMOUNT; ++i)
    {
        // 如果PCB未分配，则分配PCB
        if (!PCB_SET_STATUS[i])
        {
            // 设置PCB的分配状态为已分配，并返回PCB的指针
            PCB_SET_STATUS[i] = true;
            return (PCB *)((int)PCB_SET + PCB_SIZE * i);
        }
    }

    return nullptr;
}

// 释放一个PCB
void ProgramManager::releasePCB(PCB *program)
{
    // 计算PCB在PCB_SET数组中的索引，并设置其分配状态为未分配
    int index = ((int)program - (int)PCB_SET) / PCB_SIZE;
    PCB_SET_STATUS[index] = false;
}
```

#### 线程的创建

这里给出线程管理类`ProgramManager`的代码。

```cpp
class ProgramManager
{
public:
    List allPrograms;   // 所有状态的线程/进程的队列
    List readyPrograms; // 处于ready(就绪态)的线程/进程的队列
    PCB *running;       // 当前执行的线程
public:
    ProgramManager();
    void initialize();

    // 创建一个线程并放入就绪队列
    // function：线程执行的函数
    // parameter：指向函数的参数的指针
    // name：线程的名称
    // priority：线程的优先级
    // 成功，返回pid；失败，返回-1
    int executeThread(ThreadFunction function, void *parameter, const char *name, int priority);

    // 分配一个PCB
    PCB *allocatePCB();
    // 归还一个PCB
    // program：待释放的PCB
    void releasePCB(PCB *program);

    void schedule();
};

// 构造函数：调用initialize函数
ProgramManager::ProgramManager()
{
    initialize();
}

// 初始化函数：初始化所有队列和PCB的分配状态
void ProgramManager::initialize()
{
    // 初始化队列
    allPrograms.initialize();
    readyPrograms.initialize();
    running = nullptr;

    // 初始化PCB的分配状态
    for (int i = 0; i < MAX_PROGRAM_AMOUNT; ++i)
    {
        PCB_SET_STATUS[i] = false;
    }
}

// 创建一个线程并放入就绪队列
int ProgramManager::executeThread(ThreadFunction function, void *parameter, const char *name, int priority)
{
    // 关中断，防止创建线程的过程被打断
    bool status = interruptManager.getInterruptStatus();
    interruptManager.disableInterrupt();

    // 分配一页作为PCB
    PCB *thread = allocatePCB();

    if (!thread)
        return -1;

    // 初始化分配的页
    memset(thread, 0, PCB_SIZE);

    for (int i = 0; i < MAX_PROGRAM_NAME && name[i]; ++i)
    {
        thread->name[i] = name[i];
    }

    // 设置线程状态，优先级，时间片，pid等信息
    thread->status = ProgramStatus::READY;
    thread->priority = priority;
    thread->ticks = priority * 10;
    thread->ticksPassedBy = 0;
    thread->pid = ((int)thread - (int)PCB_SET) / PCB_SIZE;

    // 线程栈
    thread->stack = (int *)((int)thread + PCB_SIZE);
    thread->stack -= 7;
    thread->stack[0] = 0;
    thread->stack[1] = 0;
    thread->stack[2] = 0;
    thread->stack[3] = 0;
    thread->stack[4] = (int)function;
    thread->stack[5] = (int)program_exit;
    thread->stack[6] = (int)parameter;

    allPrograms.push_back(&(thread->tagInAllList));
    readyPrograms.push_back(&(thread->tagInGeneralList));

    // 恢复中断
    interruptManager.setInterruptStatus(status);

    return thread->pid;
}
```

#### 线程的调度

先实现一个最简单的时间片轮转调度算法。

修改之前的处理时钟中断函数，如下所示。

```cpp
extern "C" void c_time_interrupt_handler()
{
    PCB *cur = programManager.running;

    // 若当前线程还有配额
    if (cur->ticks)
    {
        // 减少当前线程的配额
        --cur->ticks;
        // 增加当前线程的已执行配额
        ++cur->ticksPassedBy;
    }
    // 若当前线程的配额用完，进行调度
    else
    {
        programManager.schedule();
    }
}

void ProgramManager::schedule()
{
    // 获取中断状态，关闭中断
    bool status = interruptManager.getInterruptStatus();
    interruptManager.disableInterrupt();

    // 若没有就绪线程，则恢复原本中断状态，直接返回
    if (readyPrograms.size() == 0)
    {
        interruptManager.setInterruptStatus(status);
        return;
    }

    // 若当前线程仍在运行
    if (running->status == ProgramStatus::RUNNING)
    {
        // 修改状态为就绪，放回就绪队列，重新分配配额
        running->status = ProgramStatus::READY;
        running->ticks = running->priority * 10;
        readyPrograms.push_back(&(running->tagInGeneralList));
    }
    // 若当前线程已经死亡，直接释放
    else if (running->status == ProgramStatus::DEAD)
    {
        releasePCB(running);
    }

    // 从就绪队列中取出一个线程，设置为运行状态
    ListItem *item = readyPrograms.front();
    PCB *next = ListItem2PCB(item, tagInGeneralList);
    PCB *cur = running;
    next->status = ProgramStatus::RUNNING;
    running = next;
    readyPrograms.pop_front();

    // 切换上下文
    asm_switch_thread(cur, next);

    // 恢复中断状态
    interruptManager.setInterruptStatus(status);
}
```

#### 实现任务二

在 PCB 的类中添加一个属性`parentPid`，表示该线程的父线程ID。

然后在 `executeThread` 函数中，将父线程ID设置为当前running的线程ID，表明即将被运行的线程是由当前running线程创建的。

```cpp
thread->parentPid =  (programManager.running) ? programManager.running->pid : -1;
```

在 `setup.cpp` 里，先运行第一个进程(pid = 0)，然后由它创建第二个和第三个进程。

```cpp
void third_thread(void *arg) {
    printf("(parentPid: %d) pid %d name \"%s\": Hello World!\n", 
        programManager.running->parentPid, programManager.running->pid, programManager.running->name);
    while(1) {

    }
}
void second_thread(void *arg) {
    printf("(parentPid: %d) pid %d name \"%s\": Hello World!\n", 
        programManager.running->parentPid, programManager.running->pid, programManager.running->name);
}
void first_thread(void *arg)
{
    // 第1个线程不可以返回
    printf("pid %d name \"%s\": Hello World!\n", programManager.running->pid, programManager.running->name);
    if (!programManager.running->pid)
    {
        programManager.executeThread(second_thread, nullptr, "second thread", 1);
        programManager.executeThread(third_thread, nullptr, "third thread", 1);
    }
    asm_halt();
}
```

运行结果见下图。

<img src="./images/OS_lab/lab5/添加父进程ID.png" width="800px" />

### 任务三-线程调度切换的秘密

先修改 `gdbinit` 文件，在 `first_thread` 和 `second_thread` 处打断点，然后使用gdb进行调试。

<img src="./images/OS_lab/lab5/第一个线程.png" width="500px" />

<img src="./images/OS_lab/lab5/第二个线程.png" width="500px" />

在gdb调试输出中，观察线程切换前后的寄存器状态变化。

#### **1. 通用寄存器状态**

**first_thread (第一个线程)时：**
- `eax=0x21dc0`, `ecx=0x21dc0`, `edx=0x0`, `ebx=0x0`
- `esi=0x0`, `edi=0x0`, `ebp=0x0`
- `esp=0x22db8` (栈指针)
- `eip=0x204da` (程序计数器，指向first_thread入口)

**second_thread (第二个线程)时：**
- `eax=0x22dc0`, `ecx=0x1`, `edx=0x0`, `ebx=0x0`
- `esi=0x0`, `edi=0x0`, `ebp=0x0`
- `esp=0x23db8` (栈指针改变了！)
- `eip=0x204a9` (程序计数器，指向second_thread入口)

#### **2. 重要的变化**

**栈指针(ESP)变化：**
- 线程1: `ESP=0x22db8`
- 线程2: `ESP=0x23db8`

**差值计算：**
```
0x23db8 - 0x22db8 = 0x1000 = 4096字节 = 4KB
```

两个线程的栈所存放的数据大小相同，所以差值正好是4KB，说明它们位于不同的PCB中。

#### **3. 新线程初始状态**

两个线程在开始时都有：
- `ebx=0`, `esi=0`, `edi=0`, `ebp=0`
- 这是因为新线程的栈初始化时，这些寄存器被设置为0
- 只有`eax`和`ecx`有非零值，可能是参数或临时值

#### **4. 段寄存器一致性**

两个线程的段寄存器完全相同：
- `CS=0x20` (代码段)
- `DS=0x8`, `ES=0x8` (数据段)
- `SS=0x10` (栈段)
- `GS=0x18` (视频内存段)

这说明：
- 所有内核线程共享相同的内存空间
- 没有进程隔离，只有线程栈的隔离
- 符合内核线程的设计：共享内核地址空间

#### **5. EFLAGS寄存器**

- 线程1: `EFLAGS=0x212`
- 线程2: `EFLAGS=0x206`

中断标志(IF)都为1，说明中断是开启的。

#### **调度过程还原**

**调度到second_thread**：
   - 时钟中断触发`c_time_interrupt_handler`
   - first_thread的ticks减少（可能已为0）
   - `ProgramManager::schedule()`被调用
   - first_thread状态改为READY，放回就绪队列
   - second_thread从就绪队列头部取出
   - `asm_switch_thread(firstThread, secondThread)`执行
   - 保存first_thread的ESP到`firstThread->stack`
   - 恢复second_thread的ESP从`secondThread->stack`

### 任务四-调度算法的实现

先实现先来先服务算法，即按照线程创建的顺序进行调度。

```cpp
void ProgramManager::FCFS_schedule()
{
    bool status = interruptManager.getInterruptStatus();
    interruptManager.disableInterrupt();

    // 就绪队列为空，无需调度
    if (readyPrograms.size() == 0)
    {
        interruptManager.setInterruptStatus(status);
        return;
    }

    // FCFS关键：只有当前线程终止(DEAD)时才调度
    // 如果当前线程还在运行(RUNNING)，说明它是主动让出或阻塞
    // 对于纯FCFS，运行态的线程不会主动让出，只会在结束时调度
    if (running->status == ProgramStatus::DEAD)
    {
        releasePCB(running);
    }
    else if (running->status == ProgramStatus::RUNNING)
    {
        // FCFS：运行中的线程不因时间片耗尽而被换下
        // 但如果线程主动阻塞，仍需处理
        // 这里将当前线程放回就绪队列（处理阻塞场景）
        running->status = ProgramStatus::READY;
        readyPrograms.push_back(&(running->tagInGeneralList));
    }

    // 从就绪队列头部取出下一个线程（先来先服务）
    ListItem *item = readyPrograms.front();
    PCB *next = ListItem2PCB(item, tagInGeneralList);
    PCB *cur = running;
    next->status = ProgramStatus::RUNNING;
    running = next;
    readyPrograms.pop_front();

    asm_switch_thread(cur, next);

    interruptManager.setInterruptStatus(status);
}
```

修改一下 `program_exit()` 函数，使其在进程结束时调用 `FCFS_schedule()`。

然后重新编写测试程序 `setup.cpp` ，创建多个线程，观察调度结果。

```cpp
#include "asm_utils.h"
#include "interrupt.h"
#include "stdio.h"
#include "program.h"
#include "thread.h"

STDIO stdio;
InterruptManager interruptManager;
ProgramManager programManager;

void first_thread(void *arg)
{
    printf("pid %d name \"%s\": step 1\n",
           programManager.running->pid, programManager.running->name);
    // FCFS：这个线程会一直运行直到结束
    // 不会被时钟中断抢占
    printf("pid %d name \"%s\": step 2\n",
           programManager.running->pid, programManager.running->name);
    printf("pid %d name \"%s\": finished!\n",
           programManager.running->pid, programManager.running->name);
    // 线程结束后，program_exit 会调用 FCFS_schedule 调度下一个线程
}

void second_thread(void *arg)
{
    printf("pid %d name \"%s\": step 1\n",
           programManager.running->pid, programManager.running->name);
    printf("pid %d name \"%s\": step 2\n",
           programManager.running->pid, programManager.running->name);
    printf("pid %d name \"%s\": finished!\n",
           programManager.running->pid, programManager.running->name);
}

void third_thread(void *arg)
{
    printf("pid %d name \"%s\": step 1\n",
           programManager.running->pid, programManager.running->name);
    printf("pid %d name \"%s\": step 2\n",
           programManager.running->pid, programManager.running->name);
    printf("pid %d name \"%s\": finished!\n",
           programManager.running->pid, programManager.running->name);
}

extern "C" void setup_kernel()
{
    interruptManager.initialize();
    interruptManager.enableTimeInterrupt();
    interruptManager.setTimeInterrupt((void *)asm_time_interrupt_handler);

    stdio.initialize();
    programManager.initialize();

    // 按顺序创建三个线程
    int pid1 = programManager.executeThread(first_thread, nullptr, "first thread", 1);
    int pid2 = programManager.executeThread(second_thread, nullptr, "second thread", 2);
    int pid3 = programManager.executeThread(third_thread, nullptr, "third thread", 3);

    if (pid1 == -1 || pid2 == -1 || pid3 == -1)
    {
        printf("can not execute thread\n");
        asm_halt();
    }

    // 启动第一个线程
    ListItem *item = programManager.readyPrograms.front();
    PCB *firstThread = ListItem2PCB(item, tagInGeneralList);
    firstThread->status = RUNNING;
    programManager.readyPrograms.pop_front();
    programManager.running = firstThread;
    asm_switch_thread(0, firstThread);

    asm_halt();
}
```

<img src="./images/OS_lab/lab5/先来先服务.png" width="800px" />