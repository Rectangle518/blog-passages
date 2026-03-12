这里是对操作系统实验的过程记录，也是为了写实验报告而准备的一些笔记与草稿。

实验文档：https://gitee.com/chenzx67/sysu-2026-spring-operating-system/blob/main/README.md

目录如下：

- [实验二-实验入门](#实验二-实验入门)
  - [任务一-编写MBR程序](#任务一-编写mbr程序)
  - [任务二-实模式中断](#任务二-实模式中断)
  - [任务三-汇编](#任务三-汇编)

## 实验二-实验入门

### 任务一-编写MBR程序

使用mbr.asm从12*12的位置开始打印学号：

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

### 任务二-实模式中断

1. 设置光标位置为 (8, 8)

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

2. 从 (8, 8) 开始打印学号：

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

3. 实现键盘回显

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

### 任务三-汇编

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
