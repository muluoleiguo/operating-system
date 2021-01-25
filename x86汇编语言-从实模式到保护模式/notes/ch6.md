# 第 6 章 相同的功能，不同的代码



```assembly
 ;代码清单6-1
         ;文件名：c06_mbr.asm
         ;文件说明：硬盘主引导扇区代码
         ;创建日期：2011-4-12 22:12 
      
         jmp near start
         
  mytext db 'L',0x07,'a',0x07,'b',0x07,'e',0x07,'l',0x07,' ',0x07,'o',0x07,\
            'f',0x07,'f',0x07,'s',0x07,'e',0x07,'t',0x07,':',0x07
  number db 0,0,0,0,0
  
  start:
         mov ax,0x07c0                  ;设置数据段基地址 
         mov ds,ax						;ds寄存器一般保存数据段基地址
         
         mov ax,0xb800                  ;设置附加段基地址 
         mov es,ax						;这里附加段指向显存位置，存放在es寄存器中
         
         cld							;将方向标志位DF清零，以指示传送是负方向的，与此相对应的指令是std
         mov si,mytext					;movsw指令原始数据串需要存放在ds:si位置，目的地址为es:di,因为ds目前指示的是当前代码段基地址地址，因此只要把偏移mytext存入si寄存器即可                
         mov di,0						;当前es指示显存起始位置，因此只要把偏移0存入di即可
         mov cx,(number-mytext)/2       ;实际上等于 13,cx作为计数器，每进行一次rep指令cx-1
         rep movsw						;一次传送一个字（两个字节）
     
         ;得到标号所代表的偏移地址
         mov ax,number					;此代码目的旨在显示number的偏移地址
         
         ;计算各个数位
         mov bx,ax					   ;bx指向当前number偏移地址
         mov cx,5                      ;循环次数 
         mov si,10                     ;除数 
  digit: 
         xor dx,dx						;dx（被除数高16位）清零
         div si							;除法
         mov [bx],dl                   ;保存数位	;为什么这里不用加0x7c00了？
         inc bx 						;bx自加1，指向下一个内存单元，number一共定义了5个字节内存单元
         loop digit
         
         ;显示各个数位
         mov bx,number 
         mov si,4                      
   show:
         mov al,[bx+si]					;从后往前显示
         add al,0x30
         mov ah,0x04
         mov [es:di],ax
         add di,2
         dec si
         jns show						;上一条指令符号位为SF=0（结果为非负）时跳转
         
         mov word [es:di],0x0744

         jmp near $						;无限循环

  times 510-($-$$) db 0					;512字节减去之后两个db指令=510字节，$当前指令偏移，$$当前代码段起始位置，填充字节0（db 0）
                   db 0x55,0xaa

```

