# STM32学习笔记（五）

## 超级终端串口控制程序

### 0.1  声明全局变量的方法

1.  .c文件里，函数外，正常声明

```C
u16 USART1_RX_STA
```

2. 然后.h文件中用extern定义全局变量

```c
extern u16 USART1_RX_STA
```

USART1_RX_STA

- 是**串口状态变量**
- 最高两位，反映状态
- 后面几位，用于计数，控制数据存入数组的位置



### 0.2  打开串口中断

在串口初始化（USART_Init）函数中的USART_ITConfig函数的参数3改为**ENABLE**



### 1. 串口中断服务程序

负责接收多个字符的一次输入

```c
//USART1串口中断函数
void USART1_IRQHandler(void){ 
	//局部变量Res，外面还有一个全局变量USART1_RX_STA
    u8 Res;
    //通过标志位检测是否是接收中断
	if(USART_GetITStatus(USART1, USART_IT_RXNE) != RESET){	      
        //把这一次接收到的16进制数据存入Res
		Res =USART_ReceiveData(USART1);
        //把用户输入的值实时反映回给用户
		printf("%c",Res); 
        //判断USART1_RX_STA的最高位是否为0，为0则接收未完成，if成立
		if((USART1_RX_STA&0x8000)==0){			
            //判断第二最高位是否为1，为1时if成立
			if(USART1_RX_STA&0x4000){//即收到了0x0d
				if(Res!=0x0a)USART1_RX_STA=0;
				else USART1_RX_STA|=0x8000;	
			}
            else{//第二最高位为0时(即没收到0x0d) 	
                //如果输入是回车(0x0d)则把USART1_RX_STA第二位置1 
				if(Res==0x0d)USART1_RX_STA|=0x4000;
                //如果输入不是回车
				else{
                    //把当前输入的字符Res存入数组中，由USART1_RX_STA控制存入的位置
					USART1_RX_BUF[USART1_RX_STA&0X3FFF]=Res ; 
                    //数组位置USART1_RX_STA递增
					USART1_RX_STA++;	
                    //当USART1_RX_STA超过我们设定的数组的容量时，此值清零
					if(USART1_RX_STA>(USART1_REC_LEN-1))USART1_RX_STA=0; 
				    }		 
			  }
		}   		 
	} 
} 
```

**知识点：**

1. &符号指按位与，因此判断USART1_RX_STA&0x8000是否等于0即能等价于判断USART1_RX_STA的最高位是否是0，是0时接收未完成，if成立
2. 同理 if (USART1_RX_STA&0x4000) 是判断第二最高位是否是1
3. 0x0d指回车，如果接收到的数据Res为回车时，则USART1_RX_STA|=0x4000，即把USART1_RX_STA的第二最高位置1  
4. 如果此次输入不是回车，这种把数据存入USART1_RX_BUF数组，由USART1_RX_STA来控制存入数组的位置
5. 当USART1_RX_STA超过我们设定的数组的容量时，此值清零



**程序流程：**

1. 接收中断，此字符存入Res，把此字符打印回给用户
2. 判断最高位相当于判断这个超级终端是否还在工作，只要还开着就可以一直接收，接收就没完成
3. 判断第二最高位相当于判断此次是否为回车，若不是回车则一次存入数组
4. 若是回车则证明本次输入完毕，STA置1
5. 要知道按下一次回车相当于发了两个数据：0x0d和0x0a（回车和换行）
6. 当STA置1后下一个字符的判断路线就换成【收到了0x0d】
7. 此后再判断0x0d后的这个字符是否是0x0a，如果不是证明接收错误，重新开始：STA=0
8. 如果是0x0a，则接收完成，最高位置1



### 2.主函数

0. 在main函数里主循环外设STA为0xC000

即一上电STA就是以及回车了的状态，这样能使我们一上电就能看到选项提示

1. 判断前两位是都为1，即接收完毕

```c
if(USART1_RX_STA&0xC000){ 
}
```

2. 判断STA从第三位开始的数据（计数值）

```c
if((USART1_RX_STA&0x3FFF)==0){
}
```

若此值为0证明什么数据都没输入，仅按了回车

此时用printf输出选

3. 判断是否输入了2个字符，同时判断第一个字符是否是字符1，第二个字符是否为字符y

即输入1y时，把LED1置1

```c
 else if((USART1_RX_STA&0x3FFF)==2 && USART1_RX_BUF[0]=='1' && USART1_RX_BUF[1]=='y'){ 
     			//把LED1置1
				GPIO_SetBits(LEDPORT,LED1); 
				printf("1y -- LED1灯已经点亮\r\n");
			}
```

4. 判断是否为1n，是则把LED1清0

```c
else if((USART1_RX_STA&0x3FFF)==2 && USART1_RX_BUF[0]=='1' && USART1_RX_BUF[1]=='n'){
				GPIO_ResetBits(LEDPORT,LED1); 
				printf("1n -- LED1灯已经熄灭\r\n");
			}
```

5. 输入为2y时，把LED2置1
6. 输入为2n时，把LED2清0
7. 最后所有做完后需要把STA清0

