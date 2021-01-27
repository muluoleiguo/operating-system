# 实验13编写、应用中断例程

（1） 编写并安装int 7ch中断例程，功能为显示一个用0结束的字符串，中断例程安装在0:200处。
参数：（dh）=行号，（dl）=列号，（cl）=颜色，ds:si指向字符串首地址。
中断例程安装成功后，对下面的程序进行单步跟踪，尤其注意观察int、iret指令执行前后CS、IP和栈中的状态。

```assembly
assume cs:code
data segment
db "welcome to masm! ", 0
data ends
code segment
start: 
mov dh,10	 ;所在行数：11行
mov dl,10	 ;所在列数：11列
mov cl,2	 ;字符属性
mov ax,data
mov ds,ax
mov si,0	;入口参数ds：si指向字符串data
int 7ch		 ;调用9号子程序，显示字符串
mov ax,4c00h
int 21h
code ends
end. start

```



**程序分析：**这个与show_str程序基本类似。

   **装载程序源代码如下：**

```assembly
assume cs:code
code segment
start:     ;7cH中断例程的安装程序

           mov ax, cs
           mov ds, ax
           mov si, offset show_str ;将ds:si指向源地址（show_str的机器码）
           mov ax, 0000H
           mov es, ax
           mov di, 200H       ;将es:di指向目的地址（0:200H向量表中）
           mov cx, offset show_strend - offset show_str   ;设置传输长度
           cld            ;传输方向为正
           rep movsb      ;字节传输
           ;设置中断向量表，使7cH条目中断向量指向0000:200H
           mov ax, 0000H	;这真的有必要存在吗？es里面不就是0000h吗
           mov es, ax
           mov word ptr es:[7cH*4], 200H
           mov word ptr es:[7cH*4+2], 0000H          
           mov ax, 4c00H

           int 21H



;装载的例程：7cH
;功能：int 7cH实现按行和列及字符属性显示字符串功能
;入口参数：入口参数：dh-行数、dl-列数、cl-字符属性,ds:si指向data字符串
;返回值：无
show_str:  push dx
           push cx
           push si            ;将子程序用到的寄存器入栈
           mov ax, 0b800H
           mov es, ax         ;设置显示缓冲区内存段         
           mov ax, 0          ;（ax）= 0，防止高位不为零
           mov al, 160        ;0a0H-160字节/行
           mul dh             ;相对于0b800:0000第dh行偏移量
           mov bx, ax         ;将第（dh）行的偏移地址送入bx，bx代表行偏移
           mov ax, 0
           mov al, 2          ;列的标准偏移量是2个字节
           mul dl             ;同一行列的偏移量，尽量使用乘法，（al）=列偏移
           add bx, ax         ;最终获得偏移地址(bx)=506H
           mov di,0           ;将di作为每个字符的偏移量
           mov al, cl         ;将字符属性写入al中
           mov ch, 0          ;将cx高8位设置为0
   show:   mov cl, ds:[si]    ;将字符串单个字符读入cl中
           jcxz ok            ;判断字符串是否为零。
           mov es:[bx+di+0], cl   ;在显示缓冲区中写入字符
           mov es:[bx+di+1], al   ;在显示缓冲区中写入字符属性
           add di, 2
           inc si
           jmp short show   
       ok: pop si             ;字符串字符为0，结尾         
           pop cx             ;恢复寄存器
           pop dx
           iret               
show_strend:nop            ;代码段结尾，便于计算7cH例程的长度。   
code ends
end start
```







（2） 编写并安装int 7ch中断例程，功能为完成loop指令的功能。
参数：（cx）=循环次数，（bx）=位移。

以上中断例程安装成功后，对下面的程序进行单步跟踪，尤其注意观察int、iret指令执行前后CS、IP和栈中的状态。

```assembly
assume cs:code
code segment

start: 
mov ax,0b800h
mov es, ax
mov di,160*12
mov bx, offset s-offset se	;设置从标号se到标号s的转移位移
mov cx, 80
s : mov byte ptr es:[di], '!'
add di,2
int 7ch						;如果(cx)!=0,转移到标号s处
se: nop
mov ax,4c00h
int 21h

code ends
end start
```

**程序分析：**

（1）编写一个7cH的中断例程，实现loop语句的功能。通过以上实例，我们发现int 7cH实现的功能就是loop指令的功能。如果（cx）不等于0，跳转到s标号继续执行。并且调用一次int 7cH，（cx）=（cx）-1（每次）.

（2）7cH的入口参数：cx、bx（我们规定的），cx代表计数器，bx代表了相对位移。因为中断例程不是返回到操作系统，故它必然包含iret指令，返回到调用那一点。

（3）回忆loop指令，它的相对位移是：从标号s（按照本例子来说）- loop指令下面那个指令的首地址（也就是se标号的地址）。由于int 7cH就等价于loop指令，那么它入口参数也需要相对的位移，这个相对位移就是（bx）=offset s – offset se。这里se标号与前面2个例程用途不一样了，它不用作计算例程的长度了，它代表了se标号的地址，用于计算相对位移了。

（4）我们来看看例程中，怎样实现将（cx）=（cx）-1（每次）；跳转到s标号处，直到（cx）=0。跳转到s标号，肯定是修改了ip了。至于cs也应该修改，虽然它在一个段内。产生的结果是：（ip）=s标号的ip，（cs）=s标号的cs。

（5）在调用7cH例程时，cs和ip都压栈了，cs值是s标号的cs，ip的值应该是标号se的偏移地址。那么在栈中，有了s标号的cs了，偏移地址是se的偏移地址。怎样把计算出s的偏移地址，这个例程的任务就完成了。

（6）我们可以利用iret指令。回忆iret指令，（执行iret指令，用汇编描述的过程是：pop ip；pop cs；popf，与我们引发中断过程中CPU执行的入栈过程（pushf；push cs；push ip）正好是相对应的。）popf弹栈状态寄存器值，我们不讨论了。这里主要讨论ip和cs，cs肯定是s标号的cs了，我们现在想法把栈中的ip值（栈中的值为se标号的值）修改为offset se +（bx）那么它肯定就是s标号的地址值。最后在弹栈，实现（cs）=s标号的cs，（ip）=s标号的ip了。

（7）回忆栈结构，此例中是使用的系统自动的栈（这个我喜欢，但也不要蒙圈了。）ss代表栈的段地址。bp默认是偏移地址，也就是说ss：[bp]就代表了栈空间的bp单元。sp是栈顶的指针。在8086中，一个栈的基本单元是2个字节的。以上分析都有了清晰的认识后，我们来看看这个例程代码：

（8）例程代码：

```assembly
lp:    push bp        ;将bp这个ss栈的偏址保存
       mov bp, sp     ;将当前栈顶指针值送入到bp
       dec cx         ;调用一次7cH，（cx）-1
       jcxz lpret     ;与（cx）值判断，如果为0，跳转到lpret标号
       add [bp+2], bx  ;修改ss栈中的从栈顶向下第2个单元的值
lpret: pop bp         ;恢复bp值
       iret           ;返回到调用处。
```

程序分析：

【1】因为ss和bp是配套使用的，[bp]代表ss栈中单元，bp也就是ss的偏移地址。push bp目的是保护bp的值，因为下面将会用到bp变量。

【2】mov bp, sp将栈顶指针sp送入到bp；此时栈中的数据有（从栈顶开始依次向下），bp的值（刚压入的），ip值（int 7cH调用时的，应该是se标号的值），cs值（int 7cH调用时的，应该和s标号的值一样的），flag值（int 7cH调用时的，标志寄存器的值，这个我们不考虑）。【3】dec cx; 我们为了实现loop的功能，在例程中应该是执行一次循环（调用一次7cH，）：cx的值就减1。

【4】jcxz lpret      ;与（cx）值判断，如果为0，跳转到lpret标号，

【5】add [bp+2], bx 由于栈结构栈顶是低地址，我们想修改从栈顶向下第2个单元，也就是bp+2（此时bp=sp）；那么[bp+2]就代表了栈中从栈顶开始的第2个单元。正好，这个单元就是iret弹栈的ip的值。此时[bp+2]值是offset se，再加上一个相对偏移量bx，此时的[bp+2]=offset se+(bx)；也就是offset s的值。

【6】pop bp        ;恢复bp值

【7】iret    ；pop ip ，pop cs ，popf

**装载程序代码如下：**

```assembly
assume cs:code
code segment
start:     ;7cH中断例程的安装程序
           mov ax, cs
           mov ds, ax
           mov si, offset lp  ;将ds:si指向源地址（captial的机器码）
           mov ax, 0000H
           mov es, ax
           mov di, 200H       ;将es:di指向目的地址（0:200H向量表中）
           mov cx, offset lpend - offset lp   ;设置传输长度
           cld            ;传输方向为正
           rep movsb      ;字节传输
          ;设置中断向量表，使7cH条目中断向量指向0000:200H
           mov ax, 0000H
           mov es, ax
           mov word ptr es:[7cH*4], 200H
           mov word ptr es:[7cH*4+2], 0000H         
           mov ax, 4c00H
           int 21H


;装载的例程：7cH
;功能：int 7cH实现和loop指令相同的功能
;入口参数：cx计数器、bx相对地址偏移量
;返回值：无   

lp:        push bp        ;将bp这个ss栈的偏址保存
           mov bp, sp     ;将当前栈顶指针值送入到bp
           dec cx         ;调用一次7cH，（cx）-1
           jcxz lpret     ;与（cx）值判断，如果为0，跳转到lpret标号
           add [bp+2], bx  ;修改ss栈中的从栈顶向下第2个单元的值
lpret:     pop bp         ;恢复bp值
           iret           ;返回到调用处。  
lpend: nop            ;代码段结尾，便于计算7cH例程的长度。   
code ends
end start
```









(3) 下面的程序，分别在屏幕的第2、4、 6、8行显示4句英文诗，补全程序。

```assembly
assume cs:code
code segment
    s1: db 'Good,better,best,','$'
    s2: db 'Never let it rest,','$'
    s3: db 'Till good is better,','$'
    s4: db 'And better,best.','$'
     s: dw offset s1, offset s2, offset s3, offset s4
   row:  db 2, 4, 6, 8
start :
       mov ax, cs
       mov ds, ax              ;将ds也指向了cs段
       mov bx, offset s    ;(bx)=s标号地址
       mov si, offset row      ;（si）=row标号的地址。
       mov cx, 4               ;计数器置为4,4行字符串
ok:    ;在DOS窗口设置光标的位置， 
       mov bh,0            ;BIOS中的10h中断例程的入口参数设置，bh（页号）=0  mov dh, [si]          ;入口参数：dh（行号）=（si）
       mov dl, 0               ;入口参数：dl（列数）=0
       mov ah, 2               ;10h例程中的2号子程序，功能：设置光标位置。
       int 10h                 ;调用中断例程
      ;开始显示字符串。调用21h例程，9号子程序
       mov dx,[bx]             ;入口参数：dx=（bx），每个字符串的首地址。
       mov ah,9            ;dos系统中21h例程中的9号子程序
       int 21h                 ;调用中断例程，功能：显示字符串（以$结尾的）
       inc si                  ;si按字节定义的。每次增量是1个字节。
       add bx,2            ;bx是按照字定义的，每次增量是2个字节。
       loop ok
       mov ax,4c00h
       int 21h
code ends
end start    
```

程序分析：

【1】首先我们发现4个字符串都定义在了code段中了，并且都以 '$' 结尾。int 21H显示字符串需要以 `$`结尾

【2】不要被大量的标号迷惑，它们更清楚的指向了内存单元的地址。

【3】注意一共调用了3次中断例程。置光标、显示字符串、安全退出程序。

