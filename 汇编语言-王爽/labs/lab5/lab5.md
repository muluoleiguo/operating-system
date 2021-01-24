# 编写、调试具有多个段的程序

这个实验内容稍微有一些多

1. 将下面的程序编译、连接，用Debug加载、跟踪，然后回答问题。

```assembly
assume cs:code,ds:data,ss:stack
data segment
	dw 0123h, 0456h, 0789h,Oabch,Odefh,Ofedh,Ocbah, 0987h
data ends
stack segment
	dw 0,0,0,0,0,0,0,0
stack ends
code segment
start: 	mov ax,stack
		mov ss,ax
        mov sp,16
        mov ax,data
        mov ds,ax
        push ds:[0]
        push ds:[2]
        pop ds:[2]
        pop ds:[0]
        mov ax,4c00h
        int 21h
code ends
end start

```

① CPU执行程序，程序返回前，data段中的数据为多少？
② CPU执行程序，程序返回前，cs=、ss=、ds=
③ 设程序加载后，code段的段地址为X,则data段的段地址为 , stack段的段地址为   。