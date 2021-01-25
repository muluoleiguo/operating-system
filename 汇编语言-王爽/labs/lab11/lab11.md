# 实验11 编写子程序

编写一个子程序，将包含任意字符，以0结尾的字符串中的小写字母转变成大写字母，描述如下。

名称：letterc

功能：将以0结尾的字符串中的小写字母转变成大写字母

参数：ds:si指向字符串首地址

应用举例：

```assembly
assume cs:codesg
datasg segment 
   db "Beginner's All-purpose Symbolic Instruction Code.",0 
datasg ends
codesg segment
begin: mov ax, datasg
       mov ds, ax
       mov si, 0    
       call letterc
       mov ax, 4c00H
       int 21H

letterc:   

           

codesg ends
end begin
```

注意需要进行转化的是字符串中的小写字母a〜z,而不是其他字符

程序分析：

1. 在数据段定义了一个字符串，注意这个字符串不光是有大小写字符，还有其他字符，并以0结尾。（在C语言中字符串的存储也是按照0字符来结尾的，以前我们介绍了，这个是方便判断字符串是否到结尾处。）老样子，在子程序中，首先将子程序预计使用到的变量入栈保存。ax和si吧。

2. 判断是否是结尾？也就是遇到0.

3. 判断字符串中的字符是否是小写字母？因为小写字母‘a’~‘z’是连续的ASCII码

4. 如果是小写字母使用and指令转换成大写的。

5. 这个子程序只是将该字符串的小写字母转换成了大写字母，但是调用它后没有任何的显示结果的，因为它只是个单纯的转换。

6. 怎样测试结果？我们原来应该设计过显示字符串的子程序，可以使用把它显示在屏幕上；也可以使用debug来查看。这需要修改下这个子程序。



```assembly
assume cs:codesg
datasg segment 
   db "Beginner's All-purpose Symbolic Instruction Code.",0 
   db 50 dup (0)               ;在字符串结尾增加50字节的空间，并初始化为0
datasg ends
codesg segment
begin: mov ax, datasg
       mov ds, ax		;初始化ds
       mov si, 0    	;ds：si指向字符串的首地址
        mov di, 50      ;ds：di指向结果检测字符串的首地址
       call letterc		;调用子程序
       mov ax, 4c00H
       int 21H

letterc:   
		   push ax
           push si        ;保存寄存器变量的值
           push di
   comp:   mov al, [si]   ;将字符读入al中
           cmp al, 0      ;检测字符是否为0，字符串结尾检测
           je exit        ;如果为0，跳转到exit标号，退出
           cmp al, 'a'    ;与小写字母a比较
           jb next_char   ;如果小于字符a，跳转到下个字符          
           cmp al, 'z'    ;与小写字母z比较
           ja next_char   ;如果大于字符z，跳转到下个字符           
           and al, 11011111B  ;条件都不满足，开始转换为大写字母
  next_char:                ;下一个字符，递增si，跳转到comp标号
   		   mov [di], al       ;将转换后字符写入到结果内存单元中。
           inc di             ;新增
           inc si
           jmp short comp      
   exit:   pop di     ;新增
   		   pop si
           pop ax     
           ret         
codesg ends
end begin
```

