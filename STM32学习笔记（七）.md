# STM32学习笔记（七）

## RCC设置程序分析

### 1.内部时钟树图

![image-20220211150426815](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220211150426815.png)

- OSC_OUT脚和OSC_IN是**外部时钟振荡器**输入与输出
- 左上角HSI是**内部时钟振荡器**
- 黄线左边的SYSCLK是**主频**（最大为72MHz）
- 输出主频的选择器SW，输入有三个：HSI、PLLCLK、HSE



#### 输出主频的四种方法：

1. 选择器选择HSI，即内部时钟振荡器，输出8MHz
2. 选择器选择HSE，即外部时钟振荡器，输出8MHz
3. 主频选择器选择PLLCLK，选择器PLLSRC选择连接PLLXTPRE，即外部时钟振荡器经过两个选择器来到PLLMUL倍频器，通过设置不同的倍频系数控制输出的主频
4. 主频选择器选择PLLCLK，选择器PLLSRC选择连接上方内部时钟振荡器，即内部时钟振荡器通过频率减半后经过选择器到达倍频器，通过设置不同的倍频系数控制输出的主频



### 2.RCC时钟的设置函数

```c
void RCC_Configuration(void){ //RCC时钟的设置  
	ErrorStatus HSEStartUpStatus;   
	RCC_DeInit();              /* RCC system reset(for debug purpose) RCC寄存器恢复初始化值*/   
	RCC_HSEConfig(RCC_HSE_ON); /* Enable HSE 使能外部高速晶振*/   
	HSEStartUpStatus = RCC_WaitForHSEStartUp(); /* Wait till HSE is ready 等待外部高速晶振使能完成*/   
	if(HSEStartUpStatus == SUCCESS){   
		/*设置PLL时钟源及倍频系数*/   
		RCC_PLLConfig(RCC_PLLSource_HSE_Div1, RCC_PLLMul_9); //RCC_PLLMul_x（枚举2~16）是倍频值。当HSE=8MHZ,RCC_PLLMul_9时PLLCLK=72MHZ   
		/*设置AHB时钟（HCLK）*/   
		RCC_HCLKConfig(RCC_SYSCLK_Div1); //RCC_SYSCLK_Div1——AHB时钟 = 系统时钟(SYSCLK) = 72MHZ（外部晶振8HMZ）   
		/*注意此处的设置，如果使用SYSTICK做延时程序，此时SYSTICK(Cortex System timer)=HCLK/8=9MHZ*/   
		RCC_PCLK1Config(RCC_HCLK_Div2); //设置低速AHB时钟（PCLK1）,RCC_HCLK_Div2——APB1时钟 = HCLK/2 = 36MHZ（外部晶振8HMZ）   
		RCC_PCLK2Config(RCC_HCLK_Div1); //设置高速AHB时钟（PCLK2）,RCC_HCLK_Div1——APB2时钟 = HCLK = 72MHZ（外部晶振8HMZ）   
		/*注：AHB主要负责外部存储器时钟。APB2负责AD，I/O，高级TIM，串口1。APB1负责DA，USB，SPI，I2C，CAN，串口2，3，4，5，普通TIM */  
		FLASH_SetLatency(FLASH_Latency_2); //设置FLASH存储器延时时钟周期数   
		/*FLASH时序延迟几个周期，等待总线同步操作。   
		推荐按照单片机系统运行频率：
		0—24MHz时，取Latency_0；   
		24—48MHz时，取Latency_1；   
		48~72MHz时，取Latency_2*/   
		FLASH_PrefetchBufferCmd(FLASH_PrefetchBuffer_Enable); //选择FLASH预取指缓存的模式，预取指缓存使能   
		RCC_PLLCmd(ENABLE);	//使能PLL
		while(RCC_GetFlagStatus(RCC_FLAG_PLLRDY) == RESET); //等待PLL输出稳定   
		RCC_SYSCLKConfig(RCC_SYSCLKSource_PLLCLK); //选择SYSCLK时钟源为PLL
		while(RCC_GetSYSCLKSource() != 0x08); //等待PLL成为SYSCLK时钟源   
	}  
	/*开始使能程序中需要使用的外设时钟*/   
//	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1 | RCC_APB2Periph_GPIOA | RCC_APB2Periph_GPIOB |   
//	RCC_APB2Periph_GPIOC | RCC_APB2Periph_GPIOD | RCC_APB2Periph_GPIOE, ENABLE); //APB2外设时钟使能      
//	RCC_APB1PeriphClockCmd(RCC_APB1Periph_USART2, ENABLE); //APB1外设时钟使能  
//	RCC_APB1PeriphClockCmd(RCC_APB1Periph_USART3, ENABLE);   
//	RCC_APB2PeriphClockCmd(RCC_APB2Periph_SPI1, ENABLE);   	 
//	RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);    
} 
```

#### （1）第一句：

```c
ErrorStatus HSEStartUpStatus;
```

这是个枚举定义，枚举变量是ErrorStatus，定义如下：

```
typedef enum {ERROR = 0, SUCCESS = !ERROR} ErrorStatus;
```

即ERROR为0，其他的为SUCCESS

因此，此语句含义为声明 **HSEStartUpStatus** 为 ErrorStatus 类型的枚举

#### （2）第二句：

```c
RCC_DeInit(); 
```

RCC寄存器初始化函数

#### （3）第三句：

```c
RCC_HSEConfig(RCC_HSE_ON); /* Enable HSE 使能外部高速晶振*/ 
```

开启外部高速晶振

#### （4）第四句：

```c
HSEStartUpStatus = RCC_WaitForHSEStartUp(); /* Wait till HSE is ready 等待外部高速晶振使能完成*/   
```

利用固件库函数 RCC_WaitForHSEStartUp() 读出外部高速晶振是否正常开启，并赋值给 HSEStartUpStatus

#### （5）第五句：

```c
if(HSEStartUpStatus == SUCCESS){}
```

进行判断，即外部晶振开启后才会执行if中的语句

#### （6）if内部：设置PLL时钟源及倍频系数

```c
/*设置PLL时钟源及倍频系数*/   
RCC_PLLConfig(RCC_PLLSource_HSE_Div1, RCC_PLLMul_9); 
//RCC_PLLMul_x（枚举2~16）是倍频值。当HSE=8MHZ,RCC_PLLMul_9时PLLCLK=72MHZ   
```

- RCC_PLLSource_HSE_Div1的意思是采用外部晶振的（不除以2的）方式输送给PLL
- RCC_PLLMul_9指倍频器频率，倍频有2-16倍，我们选择的是9倍

#### （7）if内部：设置AHB时钟（HCLK）

```c
/*设置AHB时钟（HCLK）*/   
RCC_HCLKConfig(RCC_SYSCLK_Div1); 
//RCC_SYSCLK_Div1——AHB时钟 = 系统时钟(SYSCLK) = 72MHZ（外部晶振8HMZ）   
/*注意此处的设置，如果使用SYSTICK做延时程序，此时SYSTICK(Cortex System timer)=HCLK/8=9MHZ*/
```

- 采用固件库函数RCC_HCLKConfig(）

- 参数为AHB的分频系数：Div1指除以1倍，即不分频，Div2指除以2，分成1/2的频率（范围为1-512Div）

#### （8）if内部：对APB1和APB2分频

```c
RCC_PCLK1Config(RCC_HCLK_Div2); 
//设置低速AHB时钟（PCLK1）,RCC_HCLK_Div2——APB1时钟 = HCLK/2 = 36MHZ（外部晶振8HMZ）   
RCC_PCLK2Config(RCC_HCLK_Div1); 
//设置高速AHB时钟（PCLK2）,RCC_HCLK_Div1——APB2时钟 = HCLK = 72MHZ（外部晶振8HMZ）   
```

**注：**AHB主要负责外部存储器时钟。APB2负责AD，I/O，高级TIM，串口1。APB1负责DA，USB，SPI，I2C，CAN，串口2，3，4，5，普通TIM

#### （9）if内部：对FLASH分频

推荐分频如下：

按照单片机系统运行频率：

​		0—24MHz时，取Latency_0；   
​		24—48MHz时，取Latency_1；   
​		48~72MHz时，取Latency_2

```c
FLASH_SetLatency(FLASH_Latency_2); //设置FLASH存储器延时时钟周期数   
		/*FLASH时序延迟几个周期，等待总线同步操作。   
		推荐按照单片机系统运行频率：
		0—24MHz时，取Latency_0；   
		24—48MHz时，取Latency_1；   
		48~72MHz时，取Latency_2*/   
```



```c
FLASH_PrefetchBufferCmd(FLASH_PrefetchBuffer_Enable); //选择FLASH预取指缓存的模式   
```

此处选开启预取缓存，提高运行速度

#### （10）开启锁相环倍频器

```c
RCC_PLLCmd(ENABLE);	//使能PLL
```

然后等待PLL输出稳定

```c
while(RCC_GetFlagStatus(RCC_FLAG_PLLRDY) == RESET); //等待PLL输出稳定   
```

#### （11）选择SYSCLK时钟源

```c
RCC_SYSCLKConfig(RCC_SYSCLKSource_PLLCLK); //选择SYSCLK时钟源为PLL
```

然后等待PLL成为SYSCLK时钟源



