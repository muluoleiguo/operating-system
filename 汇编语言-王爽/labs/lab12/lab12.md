# 实验12 编写0号中断的处理程序

编写0号中断的处理程序，使得在除法溢出发生时，在屏幕中间显示字符串“divide error!”，然后返回到DOS。
要求：仔细跟踪调试，在理解整个过程之前，不要进行后面课程的学习。

```assembly
assume cs:code
code segment
start:
mov ax,cs
mov ds,ax
mov si,offset do0
mov ax,0
mov es,ax
mov di,200h
mov cx,offset do0end-offset do0
cld
rep movsb
;设置中断向量
mov word ptr es:[0*4],200h
mov word ptr es:[0*4+2],0

mov ax,4c00h
int 21h
do0:
jmp short do0start
    db "overflow!"
do0start:
;move data address to ds register
mov ax,cs
mov ds,ax			;设置ds : si指向字符串
mov si,202h

;display area is 0xb8000h - 0xbffffh, move data to this area
mov ax,0b800h ;with hex, must a 0 before b800h
mov es,ax
mov di,12*160+36*2	;设置es:di指向显存空间的中间位置

mov cx,9
s:
mov al,[si]
mov es:[di],al
inc si
add di,2
loop s

mov ax,4c00h
int 21h
do0end:nop
code ends
end start

```

