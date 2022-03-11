# STM32学习笔记（二）

## 1.核心板电路

电路各部分：

- **单片机最小系统电路**：包括单片机，主晶振，起振电容，RC复位电路
- **USB转串口电路**（CH340芯片）：负责将USB协议信号转换成单片机能处理的USTART串口通信
- **ASP自动下载电路**：负责检测串口数据，实现自动下载功能
- **MicroUSB 接口**：连接电脑，为核心板提供5V电源输入和串口通信
- **电源电路**：为核心板提供5V和3.3V稳定电压
- **功能电路**：含有LED、按键、蜂鸣器、RTC走时等附加功能



各元件作用：

- 滤波电容：去除电源电压的波动干扰
- 起振电容：帮助晶体振荡器稳定的工作
- 继电器：核心板的电源总开关
- RTC晶振：为单片机内部的RTC时钟提供32.768kHz频率的时钟基准



## 2.点亮LED灯(用库文件)

PS：需要先链接sys.h文件

```c
#include "sys.h"
```

### 关于初始化：

```c
LED_Init();
```

所有I/O端口使用前必须初始化

初始化内容包括：

- 输入还是输出（接口工作模式）
- 端口号（比如：GPIO_PIN_  1）
- 输出速率（2/10/50MHz）比如：GPIO_Speed_20MHz
- 启动GPIO端口 
- 设置IO端口组

```c
GPIO_Init(GPIOB, &GPIO_InitStructure);
```

即把GPIO_InitStructure结构体里设置的数据赋值给GPIOB端口，然后进行初始化。

###  方法一：GPIO_WriteBit函数

核心语句：

```c
while(1){
	GPIO_WriteBit(LEDPORT,LED1,(BitAction)(1)) //LED1端口输出高电平
    GPIO_WriteBit(LEDPORT,LED2,(BitAction)(0)) //LED2端口输出低电平
}
```

这里使用了库函数：GPIO_WriteBit函数

**参数1**：使用哪一组端口

**参数2**：使用这一组里的第几号端口

**参数3**： BitAction是一个枚举变量，在括号中设置的值可赋值给这个枚举变量。

这里参数3也可以直接换成**Bit_RESET**或者**Bit_SET**，分别表示清零和置1

详见**STM32固件库使用手册**P120

### 方法二：GPIO_ResetBits函数

```c
GPIO_ResetBits(GPIOA,GPIO_Pin_10);
```

即把GPIOA组的GPIO_Pin_10端口清零

### 方法三：GPIO_SetBits函数

```C
GPIO_SetBits(GPIOA,GPIO_Pin_11);
```

即把GPIOA组的GPIO_Pin_11端口置1

###  方法四：GPIO_Write函数

```c
GPIO_Write(GPIOA,0x0001); //直接把GPIOA这一组的端口变量都赋值为1
GPIO_Write(GPIOB,0x0000); //直接把GPIOB这一组的端口变量都赋值为0
```

不建议使用这种全部赋值的方法



### PS：直接操作寄存器法

需要多层宏定义嵌套，然后最底层的定义变量直接对寄存器地址进行操作。一般板子厂家会提供。



## 3.LED闪灯程序

主要就是利用延时函数，在开灯和关灯之间加入延时即可看到闪灯效果。

延时函数如下：

```c
delay_us();    //微秒级延时函数，括号里即可填入延时时间
delay_ms();    //毫秒级延时函数，括号里即可填入延时时间
```

### 方法一：普通延时法

#### 微秒级延时函数：

```c
void delay_us(u16 time)
{    
   u16 i=0;  
   while(time--)
   {
      i=10;  //自己定义
      while(i--) ;    
   }
```

#### 毫秒级延时函数：

```c
void delay_ms(u16 time)
{    
   u16 i=0;  
   while(time--)
   {
      i=12000;  //自己定义
      while(i--) ;    
   }
}
```

即利用for循环，一直倒计时计数，直到为0跳出循环往下执行

**优点**：比较简单，容易理解与使用

**缺点**：不是很精准

### 方法二：SysTick库函数

 通过系统的滴答定时器完成，可以精密延时

滴答计时器：本质是倒计时计时器 

SysTick库函数详见**STM32固件库使用手册**P238

#### 第一步：重计数初值

这里涉及到单片机的主频（AHB_INPUT）

**主频**：指计数为多少时为1ms

比如：主频是72MHz，则计数72次为1ms

```c
#define AHB_INPUT 72    //定义主频
u32 us        //定义延时时间长度，即us
SysTick->LOAD = AHB_INPUT * us;  //即把主频乘以延时时间长度，然后这个值赋值给滴答定时器SysTick里的时间计时器LOAD里
```

#### 第二步：清空定时器的计数器

```c
SysTick->VAL = 0x00;
```

没有这句的话在主程序多次使用延时函数时会出现延时不准确的bug

#### 第三步：打开定时器

```C
SysTick->CTRL = 0X00000005;
```

#### 第四步：等待计数到0

```C
while(!(SysTick->CTRL&0X00010000));
```

当 SysTick->CTRL 的值不为0时，一直进入while的循环

直到 SysTick->CTRL 的值为0是，才跳出循环，往下执行

#### 第五步：关闭计时器

```c
SysTick->CTRL = 0X00000004;
```

#### 完整延时函数：

微秒级延时函数：

```c
#define AHB_INPUT 72  
void delay_us(u32 us){
    //重装计数初值
	SysTick->LOAD = AHB_INPUT * us; 
	//清空定时器的计数器
    SysTick->VAL = 0x00;
    //打开定时器
	SysTick->CTRL = 0X00000005;
    //等待计数到0
	while(!(SysTick->CTRL&0X00010000));
	//关闭定时器
    SysTick->CTRL = 0X00000004;
}
```

毫秒级延时函数：

```c
void delay_ms(u16 ms){
	while(ms-- != 0){
		delay_us(1000);    //调用1000us的延时
	}
}
```



## 4.LED呼吸灯

即实现亮度逐渐提升以及亮度逐渐降低

#### **如何显示不同的亮度？**

**本质是**控制灯点亮的延迟时间**t1**和灯熄灭的延迟时间**t2**

**t1**占空比 **K = t1/(t1+t2)**

前提：延时时间足够短，以达到视觉暂留效果

- 当 K 为50%时，LED亮度达到最大
- 当 K 变小时，灯变暗
- 当 K 变大时，灯变亮

#### 主程序如下：

```c
int main (void){//主程序
	//定义需要的变量
	u8 MENU;
	u16 t,i;
	//初始化程序
	RCC_Configuration(); //时钟设置
	LED_Init();
	//设置变量的初始值
	MENU = 0;
	t = 1;
	//主循环
	while(1){
		//菜单0
		if(MENU == 0){ //变亮循环
			for(i = 0; i < 10; i++){
				GPIO_WriteBit(LEDPORT,LED1,(BitAction)(1)); //LED1接口输出高电平1
				delay_us(t); //延时t
				GPIO_WriteBit(LEDPORT,LED1,(BitAction)(0)); //LED1接口输出低电平0
				delay_us(501-t); //延时501微秒-t
			}
			t++;
			if(t==500){
				MENU = 1;
			}
		}
		//菜单1
		if(MENU == 1){ //变暗循环
			for(i = 0; i < 10; i++){
				GPIO_WriteBit(LEDPORT,LED1,(BitAction)(1)); //LED1接口输出高电平1
				delay_us(t); //延时t
				GPIO_WriteBit(LEDPORT,LED1,(BitAction)(0)); //LED1接口输出低电平0
				delay_us(501-t); //延时501微秒-t
			}
			t--;
			if(t==1){
				MENU = 0;
			}
		}		
	}
}

```

结构分析：

1. 最外面套了一个while循环，判断条件是1，说明整个程序会**一直不断循环**
2. 每一次大循环，都会判断MENU的值，为0则进入变亮循环，为1则进入变暗循环
3. 每确定一次t就会确定一次亮度，为了让某一亮度显示出来，让人眼感受到，我们需要把这一个亮度保持一段时间：通过for循环，当 i 递增到9时才切换到下一个 t 值（亮度值）
4. 在变亮循环中，因为需要变亮，即逐渐增大 t1 占空比，所以 t 递增，用 t++
5. 在变暗循环中，因为需要变暗，即逐渐减小 t1 占空比，所以 t 递减，用 t--
6. 最后当变亮循环到达极限（t==500）时，就要开始跳到变暗循环（MENU=1）
7. 同理当变暗循环到达极限（t==1）时，就要开始跳到变亮循环（MENU=0）



















  

