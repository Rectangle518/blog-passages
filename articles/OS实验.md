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

mov al, '3'
mov [gs:2 * 974], ax

mov al, '2'
mov [gs:2 * 975], ax

mov al, '5'
mov [gs:2 * 976], ax

mov al, '1'
mov [gs:2 * 977], ax

mov al, '9'
mov [gs:2 * 978], ax

mov al, '6'
mov [gs:2 * 979], ax

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

mov ah, 0x02
mov bh, 0x00
mov dh, 0x08
mov dl, 0x08

int 0x10

times 510 - ($ - $$) db 0
db 0x55, 0xaa
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
org 0x7c00
[bits 16]

start:
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

times 510 - ($ - $$) db 0
db 0x55, 0xaa
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
org 0x7c00
[bits 16]

start:
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

times 510 - ($ - $$) db 0
db 0x55, 0xaa
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