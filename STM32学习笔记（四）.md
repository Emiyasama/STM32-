# STM32学习笔记（四）

## 1.USART驱动程序

####  了解个接口： 

1. PA9/USART1_TX：单片机向电脑端发送数据（引脚30）
2. PA10/USART1_RX：单片机接收来自电脑端的信息（引脚31）



#### 设置波特率：

- 程序中的串口初始化的**波特率参数**和电脑利用软件接收时设置的**波特率参数**要一样，才能通信

- 设置的波特率数字越大通信速度越快
- 波特率的值不可随便取，标准值有：600、1200、2400、4800、9600、14400、19200、115200



#### 电脑端接收工具：DYS串口助手

1. 设置端口号
2. 设置波特率
3. 发送模式和接收模式可选数值或字符
4. 点击打开端口即可接收到单片机发来的信息



#### 发送与接收模式

- 数值模式：发送/接收到的信息以16进制形式显示
- 字符模式：发送/接收到的信息以ASCII形式显示



## 2.USART发送程序

 ![image-20220123151744859](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220123151744859.png)

发送方法如下：

### 方法一：

```c
while(1){
	//发送方法1
    USART_SendData(USART1 , 0x55); 
    while(USART_GetFlagStatus(USART1, USART_FLAG_TC)==RESET);
}
```

#### USART_SendData函数

- 参数1：哪个串口
- 参数2：可设置一个8位16进制的数据，或者设置一个 ‘字符‘

#### USART_GetFlagStatus函数

作用：检查发送中断标志位。只有当信息发送完成后程序才能往下执行

当上一个信息发送完毕后寄存器中相应标志位会置1，如果不检查上一个信息是否发送完成，就有可能导致上一个信息还未发送完就发下一个信息了，会导致信息重叠错乱

函数参数：

1. 哪个串口

2. 要检查判断的标志位

   

### 方法二：

```C
printf("STM32F103");
```

用法与c语言相同

<img src="C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220123234439715.png" alt="image-20220123234439715" style="zoom: 67%;" />

**PS：**默认printf函数是用在USART1上的，当需要用在别的串口时，可以在usart.h中修改以下这条宏定义

```c
#define USART_n USART1
//作用：定义使用printf函数的串口，同一时间只能用在一个串口上
//其他串口要是用USART_printf函数
```

 

### 方法三：

```c
USART1_printf("STM32 &d %d",a,b);
```

其实跟方法二相同，但这种方法可以使用多个串口，然后每个串口的printf函数都要在usart.c封装好，初始化完



## 3.USART接收程序

两种接收方式：查询接收方式、中断接收方式

### 查询接收方式：

#### （1）修改串口初始化函数

在usart.c中的USART1_Init函数中的最后此处的函数修改成如下样子

```c
USART_ITConfig(USART1,USART_IT_RXNE,DISABLE)
//这里最后一个参数控制了中断功能是否打开
```

#### （2）主循环内容

```c
if(USART_GetFlagStatus(USART1,USART_FLAG_RXNE) != RESET){  
			a =USART_ReceiveData(USART1);
}
```

1. USART_GetFlagStatus是检查标志位的函数，USART_FLAG_RXNE即代表我们检查的标志位是判断是否接收到电脑端发来的信息，收到则置1
2. 当接收到电脑端信息时执行if语句中的内容
3. USART_ReceiveData是接收参数里指定的串口的数据

**查询程序优缺点：**

- 优点：简单
- 缺点：缺乏实时性，因为主循环函数要执行完很多条代码才再循环一次，此时才会再次查询标志位，此时可能已经过去了很久了



### 串口中断接收方式

#### （1）修改串口初始化函数

```c
USART_ITConfig(USART1,USART_IT_RXNE,ENABLE)
//这里最后一个参数控制了中断功能是否打开
```

#### （2）中断函数

```c
void USART1_IRQHandler(void){ 
	u8 a;
    //判断这种中断是不是接收中断
	if(USART_GetITStatus(USART1, USART_IT_RXNE) != RESET){ 
        //接收来自USART1的信息
		a =USART_ReceiveData(USART1);
		printf("%c",a); 	  
	} 
} 
```



**中断接收方法优点**：

- 能在接收到信息时就跳到中断函数，然后一直处理传输来的数据，直到执行完——实时性强



## 4.USART控制程序

### （1）添加头文件与初始化

添加LED、按键、蜂鸣器的头文件和初始化

### （2）查询接收方式

记得把串口初始化调为DISABLE

1. 检查标志位
2. 通过switch-case结构分别设置输入为字符0或1时的情况
3. 输入输出模式调成字符
4. case '0'时通过GPIO_WriteBit函数调节LED变暗，并用printf函数返回0
5. case '1'时通过GPIO_WriteBit函数调节LED变亮，并用printf函数返回1

**要注意**：printf里不要打中文，不然会乱码



### （3）按键控制部分

```c
//判断KEY1是否按下
if(!GPIO_ReadInputDataBit(KEYPORT,KEY1)){
    //延时
	delay_ms(20);
    //再次判断KEY1是否按下
	if(!GPIO_ReadInputDataBit(KEYPORT,KEY1)){ 
        //等待松开
		while(!GPIO_ReadInputDataBit(KEYPORT,KEY1));  
		printf("KEY1 "); 
	}
}		
```

KEY2部分跟KEY1一样，可以粘贴复制改一下



## 5.超级终端HyperTerminal

### （1）新建连接

1. 文件 - 新建连接
2. 设置串口 - 设置波特率 - 编码选择GB2312 - 确定

### （2）断开连接

在左上角的COM即可断开

**要注意**：下载程序时要断开超级终端

### （3）拓展功能：转义字符

![image-20220124005817802](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220124005817802.png)

### （4）修改超级终端颜色

```c
printf("\033[1;40;32m 这里写文字");
```

1. 这里\033是转义序列的开始
2. 后面接一个[，不需要右半边
3. 第1个数字1是一些字体的设置，以分号；分隔
4. 第2个数字40是背景颜色
5. 第3个数字是前景，就是字的颜色
6. 最后是小写m结尾表示结束

**第一个数字意义**：

![image-20220124010553174](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220124010553174.png)

**第2、3个数字意义**：

![image-20220124010725499](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220124010725499.png)



### （5）输出完后改回颜色

```c
printf("\033[1;40;32m 这里写文字 \033[0m");
```

即以\033[0m结尾















