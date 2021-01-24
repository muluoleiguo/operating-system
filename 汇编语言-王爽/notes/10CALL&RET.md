call和ret指令都是转移指令，它们都修改IP,或同时修改CS和IP。它们经常被共同用来实现子程序的设计。



## 10.1 ret 和 retf

ret指令用**栈**中的数据，修改IP的内容，从而实现近转移；
retf指令用**栈**中的数据，修改CS和IP的内容，从而实现远转移。

CPU执行ret指令时，进行下面两步操作：

`(1) (IP)=((ss)*16+(sp))`
`(2) (sp)=(sp)+2`

CPU执行retf指令时，进行下面4步操作：
`(1) (IP)=((ss)*16+(sp))`
`(2) (sp)=(sp)+2`
`(3) (CS)=((ss)*16+(sp))`
`(4) (sp)=(sp)+2`

可以看出，如果我们用汇编语法来解释ret和retf指令，则:
CPU执行ret指令时，相当于进行：
pop IP
CPU执行retf指令时，相当于进行：
pop IP
pop CS



下面的程序中，ret指令执行后，(IP)=0, CS:IP指向代码段的第一条指令

```assembly
assume cs:code
stack segment
	db 16 dup (0)
stack ends
code segment
mov ax,4c00h
int 21h
start:
mov ax, stack
mov ss, ax
mov sp, 16
mov ax, 0
push ax
mov bx, 0
ret
code ends
end start
```



## 10.2 call 指令

CPU执行call指令时，进行两步操作：
(1) 将当前的IP或CS和IP压入栈中；
(2) 转移。

call指令不能实现短转移，除此之外，call指令实现转移的方法和jmp指令的原理相同，



## 10.3依据位移进行转移的call指令

call标号(将当前的IP压栈后，转到标号处执行指令)
CPU执行此种格式的call指令时，进行如下的操作：

1. (sp)=(sp)-2
   ((ss)*16+(sp))=(IP)
2. (IP)=(IP)+16 位位移。

16位位移=标号处的地址-call指令后的第一个字节的地址；
16位位移的范围为-32768〜32767，用补码表示；
16位位移由编译程序在编译时算出。



CPU执行“call标号”时，相当于进行：
push IP
jmp near ptr 标号



# 10.4 转移的目的地址在指令中的call指令

前面讲的call指令，其对应的机器指令中并没有转移的目的地址，而是相对于当前IP的转移位移。

"call far ptr标号”实现的是段间转移。



CPU执行此种格式的call指令时，进行如下的操作。

1. `(sp)=(sp)-2`
   `((ss)*16+(sp))=(CS)
   (sp)=(sp)-2
   ((ss)* 16+(sp))=(IP)`
2. (CS)=标号号所在段的段地址
   (IP)=标号在段中的偏移地址

CPU执行“call far ptr 标号”时，相当于进行

push CS
push IP
jmp far ptr 标号


## 10.5转移地址在寄存器中的call指令

指令格式：call 16 位 reg
功能：
(sp)=(sp)-2
((ss)* 16+(sp))=(IP)
(IP)=(16 位 reg)

相当于

push IP
jmp 16 位 reg



## 10.6转移地址在内存中的call指令

转移地址在内存中的call指令有两种格式。

1. call word ptr内存单元地址

相当于

push IP
jmp word ptr内存单元地址



2. call dword ptr内存单元地址

相当于

push CS
push IP
jmp dword ptr内存单元地址



## 10.7 call和ret的配合使用



下面程序返回前，bx中的值是多少？

```assembly
assume cs:code
code segment
start: 
mov ax,1
mov cx,3
call s
mov bx, ax ; (bx)=?
mov ax,4c00h
int 21h
s: 
add ax,ax
loop s
ret
code ends
end start

```

1. CPU将call s指令的机器码读入，IP指向了 call s后的指令mov bx,ax,然后CPU执行call s指令，将当前的IP值(指令mov bx,ax的偏移地址)压栈，并将IP的值改变为标号s处的偏移地址；
2. CPU从标号s处开始执行指令，loop循环完毕后，(ax)=8;
3.  CPU将ret指令的机器码读入，IP指向了 ret指令后的内存单元，然后CPU执行ret指令，从栈中弹出一个值(即call s先前压入的mov bx,ax指令的偏移地址)送入IP中。则CS:IP指向指令mov bx,ax；
4. CPU从mov bx,ax开始执行指令，直至完成。





```assembly
; 源程序 内存中的情况(假设程序从内存1000 : 0处装入)
assume cs:code
stack segment
db 8 dup (0)		;1000:0000 00 00 00 00 00 00 00 00

db 8 dup (0)		;1000:0008 00 00 00 00 00 00 00 00
stack ends
code segment
start:
mov ax, stack
mov ss,ax
mov sp,16
mov ax,1000
call s				;1001:000B E8 05 00
mov ax,4c00h
int 21h
s:
add ax,ax
ret					;1001:0015 C3
code ends
end start

```

主要执行过程

1. 前3条指令执行后，栈的情况如下

1000:0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ss:sp

2. call指令读入后，(IP)=000EH, CPU指令缓冲器中的代码为：E8 05 00；

CPU执行E8 05 00,首先，栈中的情况变为：

1000:0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 0E(ss:sp) 00

然后，(IP)=(IP)+0005=0013H

3. CPU从cs:0013H处(即标号s处)开始执行。

4. ret指令读入后：

(IP)=0016H, CPU指令缓冲器中的代码为：C3
CPU执行C3,相当于进行pop IP,执行后，栈中的情况为：

1000:0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 0E 00 ss:sp

(IP)=000EH

5. CPU回到cs:000EH处(即call指令后面的指令处)继续执行

可以写一个具有一定功能的程序段，我们称其为子程序， 在需要的时候，甩call指令转去执行。可是执行完子程序后，如何让CPU接着call指令向下执行？ call指令转去执行子程序之前，call指令后面的指令的地址将存储在栈中，所以可在子程序的后面使用ret指令，用栈中的数据设置IP的值，从而转到call指令后面的代码处继续执行。

框架如下

```assembly
assume cs:code
code segment
main:
;...
call sub1		;调用子程序sub1
;...
mov ax,4c00h
int 21h
sub1:			;子程序sub1开始
;...
call sub2		;调用子程序sub2
;...
ret				;子程序sub1返回
sub2:			;子程序sub2开始
;...
ret				;子程序返回
code ends
end main
```



## 10.8 mul 指令

mul是乘法指令，

1. 两个相乘的数：两个相乘的数，要么都是8位，要么都是16位。如果是8位， 一个默认放在AL中，另一个放在8位reg或内存字节单元中；如果是16位，一个默认在AX中，另一个放在16位reg或内存字单元中。
2. 结果：如果是8位乘法，结果默认放在AX中；如果是16位乘法，结果高位默认在DX中存放，低位在AX中放。

格式如下：
mul reg
mul内存单元

内存单元可以用不同的寻址方式给出

mul word, ptr [bx+si+8 ]
含义：

`(ax)=(ax)*((ds)*16+(bx)+(si)+8)结果的低 16 位。`
`(dx)=(ax)*((ds)* 16+(bx)+(si)+8)结果的高 16 位。`



## 10.9模块化程序设计

call与ret指令共同支持了汇编语言编程中的模块化设计



## 10.10参数和结果传递的问题

子程序一般都要根据提供的参数处理一定的事务，处理后，将结果(返回值)提供给调用者

设计一个子程序，可以根据提供的N,来计算N的3次方。

这里面就有两个问题：
(1) 将参数N存储在什么地方？
(2) 计算得到的数值，存储在什么地方？

很显然，可以用寄存器来存储，可以将参数放到bx中；因为子程序中要计算`N*N*N`,可以使用多个mul指令，为了方便，可将结果放到dx和ax中。子程序如下。

```assembly
;说明：计算N的3次方
;参数：(bx)=N
;结果：(dx: ax) =N^3

cube:
mov ax,bx
mul bx
mul bx
ret

```



计算data段中第一组数据的3次方，结果保存在后面一组dword单元中

```assembly
assume cs:code
data segment
dw 1,2,3,4,5,6,7,8
dd 0,0,0,0,0,0,0,0
data ends

```

```assembly
code segment
start:
mov ax,data
mov ds,ax
mov si,0		;ds : si指向第一组word单元
mov di,16		;ds : di指向第二组dword单元
mov cx,8
s: 
mov bx,[si]
call cube
mov [di],ax
mov [di].2,dx
add si,2		;ds : si指向下一个word单元
add di,4		;ds : di指向下一个dword单元
loop s

mov ax,4c00h
int 21h
cube: 
mov ax,bx
mul bx
mul bx
ret
code ends
end start

```



## 10.11批量数据的传递

寄存器的数量终究有限，我们不可能简单地用寄存器来存放多个需要传递的数据。

对于返回值，也有同样的问题。

我们将批量数据放到内存中，然后将它们所在内存空间的首地址放在寄存器中，传递给需要的子程序。

设计一个子程序，功能：将一个全是字母的字符串转化为大写。

这个子程序需要知道两件事，字符串的内容和字符串的长度。因为字符串中的字母可能很多，所以不便将整个字符串中的所有字母都直接传递给子程序。但是，可以将字符串在内存中的首地址放在寄存器中传递给子程序。因为子程序中要用到循环，我们可以用loop指令，而循环的次数恰恰就是字符串的长度。出于方便的考虑，可以将字符串的长度放到cx中。

```assembly
capital: 
and byte ptr [si] , 11011111b ;将ds : si所指单元中的字母转化为大写
inc si 							;ds : si指向下一个单元
loop capital
ret

```



```assembly
assume cs:code
data segment
db 'conversation'
data ends
code segment
start:
mov ax,data
mov ds,ax
mov si,0 		;ds:si指向字符串（批量数据）所在空间的首地址
mov cx, 12 		;cx存放字符串的长度
call capital
mov ax,4c00h
int 21h

capital:
and byte ptr [si], 11011111b
inc si
loop capital
ret
code ends
end start

```



## 10.12寄存器冲突的问题

设计一个子程序，功能：将一个全是字母，以o结尾的字符串，转化为大写

程序要处理的字符串以o作为结尾符，这个字符串可以如下定义：
db 'conversation ', 0

子程序可以依次读取每个字符进行检测，如果不是0,就进行大写的转化；如果是0,就结束处理。由于可通过检测0而知道是否已经处理完整个字符串，所以子程序可以不需要字符串的长度作为参数。可以用jcxz来检测0。

```assembly
;说明：将一个全是字母，以0结尾的字符串，转化为大写
;参数：ds:si指向字符串的首地址
;结果：没有返回值
capital:
mov cl,[si]
mov ch,0
jcxz ok						;如果(cx)=0,结束；如果不是0，处理
and byte ptr [si],11011111b	;将ds:si所指单元中f字母转化为大写
inc si						;ds:si指向下一个单元
jmp short capital
ok:ret

```



将data段中的字符串全部转化为大写。

```assembly
assume cs:code
data segment
db ’word',0
db 'unix',0
db 'wind',0
db 'good',0
data ends

```



所有字符串的长度都是5（算上结尾符0），使用循环，重复调用子程序capital,完成对4个字符串的处理。

这个程序在思想上完全正确，但在细节上却有些**错误**，把错误找出来。

```assembly
code segment
start: 
mov ax,data
mov ds, ax
mov bx, 0
mov cx, 4
s: 
mov si, bx
call capital
add bx, 5
loop s
mov ax,4c00h
int 21h
capital:
mov cl,[si]
mov ch,0
jcxz ok
and byte ptr [si],11011111b
inc si
jmp short capital
ok:ret
code ends
end start

```

分析： 问题在于CX的使用，主程序要使用CX记录循环次数，可是子程序中也使用了 CX,在执行子程序的时候，CX中保存的循环计数值被改变，使得主程序的循环出错。

**子程序中使用的寄存器，很可能在主程序中也要使用，造成了寄存器使用上的冲突**

两个方案

1. 编写调用子程序的程序时，注意子程序中有没有用到会产生冲突的寄存器，如果有，调用者使用别的寄存器；

2. 在编写子程序的时候，不要使用会产生冲突的寄存器。

其实这两个方案都是不可行的



解决这个问题的简捷方法是，**在子程序的开始将子程序中所有用到的寄存器中的内容都保存起来**，在子程序返回前再恢复。可以用**栈**来保存寄存器中的内容。

以后，我们编写子程序的标准框架如下：

子程序开始：子程序中使用的寄存器入栈
子程序内容
子程序中使用的寄存器出栈
返回(ret、 retf)

改进一下子程序capital的设计:

```assembly
capital: 
push ex
push si
change: 
mov Cl,[si]
mov ch, 0
jcxz ok
and byte ptr [si],11011111b
inc si
jmp short change
ok: 
pop si
pop cx
ret

```

