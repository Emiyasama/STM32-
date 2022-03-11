# STM32学习笔记（十四）

## 继电器与步进电机的原理与驱动

### 1. 继电器原理

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\继电器原理1.jpg" alt="继电器原理1" style="zoom: 25%;" />

如图，当线圈通电时衔铁被吸下，开关柄 A 与常开柄 C 连通

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\继电器原理2.jpg" alt="继电器原理2" style="zoom:25%;" />

- 线圈连通驱动电路ULN2003（内部是非门与放大电流器件）

- 驱动电路连接单片机的 IO 端口
- AC两端连接用电器电路

具体运行流程：

1. 单片机IO端口输出1时为打开继电器，输出0时为关闭继电器
2. 通过驱动电路后 1变为0, 0变为1
3. 驱动的输出口与继电器一端线圈相连
4. 线圈另一端连接5V电压，因此驱动输出 0 时线圈一端为 0V 另一端为 5V，线圈连通
5. 驱动输出 1 时线圈一端为 5V 另一端为 5V，线圈无电流
6. 线圈连通则用电器工作，线圈无电流则用电器不工作

因此，最后等效为：单片机输出为 1 时用电器工作，单片机输出为 0 时用电器不工作



### 2. 继电器驱动程序

懂了原理之后，驱动的原理就很简单了，只要控制单片机 IO 端口的高低电平即可控制继电器开关

在 relay.c 中函数如下：

```c
void RELAY_1(u8 c){ //继电器的控制程序（c=0继电器放开，c=1继电器吸合）
	GPIO_WriteBit(RELAYPORT,RELAY1,(BitAction)(c));//通过参数值写入接口
}
```

在 main.c 中主程序如下：

```c
while(1){
		if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A))RELAY_1(1); //当按键A按下时继电器1标志置位		
		if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_B))RELAY_1(0); //当按键B按下时继电器1标志置位		
		if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_C))RELAY_2(1); //当按键C按下时继电器2标志置位
		if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_D))RELAY_2(0); //当按键D按下时继电器2标志置位
	}
```



### 3. 步进电机原理

四步步进电机：

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\四步步进电机.jpg" alt="四步步进电机" style="zoom:25%;" />

八步步进电机：

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\八步步进电机.jpg" alt="八步步进电机" style="zoom:25%;" />

原理：与继电器同理，只要线圈通电就会吸引内部转子转动，以转动步骤分为四步与八步，如上图



### 4. 步进电机驱动程序

在 step_motor.c 中函数如下：

#### （1）步进电机初始化

```c
void STEP_MOTOR_Init(void){ //LED灯的接口初始化
	GPIO_InitTypeDef  GPIO_InitStructure; 	
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_GPIOB | RCC_APB2Periph_GPIOC | RCC_APB2Periph_GPIOD | RCC_APB2Periph_GPIOE, ENABLE); //APB2外设GPIO时钟使能      
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);//启动AFIO重映射功能时钟    
    GPIO_InitStructure.GPIO_Pin = STEP_MOTOR_A | STEP_MOTOR_B | STEP_MOTOR_C | STEP_MOTOR_D; //选择端口                        
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP; //选择IO接口工作方式       
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz; //设置IO接口速度（2/10/50MHz）    
	GPIO_Init(STEP_MOTOR_PORT, &GPIO_InitStructure);
	//必须将禁用JTAG功能才能做GPIO使用
	GPIO_PinRemapConfig(GPIO_Remap_SWJ_Disable, ENABLE);// 改变指定管脚的映射,完全禁用JTAG+SW-DP	
	STEP_MOTOR_OFF(); //初始状态是断电状态 			
}
```



#### （2）电机断电函数

因为电机不能一直通着电，所以每次单片机控制完之后需要给电机断电

```c
void STEP_MOTOR_OFF (void){//电机断电
	GPIO_ResetBits(STEP_MOTOR_PORT,STEP_MOTOR_A | STEP_MOTOR_B | STEP_MOTOR_C | STEP_MOTOR_D);//各接口置0
}
```



#### （3）电机转动函数（以4拍顺时针为例）

```c
void STEP_MOTOR_4R (u8 speed){//电机顺时针，4拍，速度快，力小
	GPIO_ResetBits(STEP_MOTOR_PORT,STEP_MOTOR_C | STEP_MOTOR_D);//0
	GPIO_SetBits(STEP_MOTOR_PORT,STEP_MOTOR_A | STEP_MOTOR_B);//1
	delay_ms(speed); //延时
	GPIO_ResetBits(STEP_MOTOR_PORT,STEP_MOTOR_A| STEP_MOTOR_D);//0
	GPIO_SetBits(STEP_MOTOR_PORT,STEP_MOTOR_B | STEP_MOTOR_C);//1
	delay_ms(speed); //延时
	GPIO_ResetBits(STEP_MOTOR_PORT,STEP_MOTOR_A | STEP_MOTOR_B);//0
	GPIO_SetBits(STEP_MOTOR_PORT,STEP_MOTOR_C | STEP_MOTOR_D);//1
	delay_ms(speed); //延时
	GPIO_ResetBits(STEP_MOTOR_PORT,STEP_MOTOR_B | STEP_MOTOR_C);//0
	GPIO_SetBits(STEP_MOTOR_PORT,STEP_MOTOR_A | STEP_MOTOR_D);//1
	delay_ms(speed); //延时
	STEP_MOTOR_OFF(); //进入断电状态，防电机过热
}
```



#### （4）main函数内主循环

```c
while(1){		
    	if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A))STEP_MOTOR_4R(3); //当按键A按下时步进电机4步右转		
		else if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_B))STEP_MOTOR_4L(3); //当按键B按下时步进电机4步左转		
		else if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_C))STEP_MOTOR_8R(3); //当按键C按下时步进电机8步右转
		else if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_D))STEP_MOTOR_8L(3); //当按键D按下时步进电机8步左转
		else STEP_MOTOR_OFF();//当没有按键时步进电机断电
	}
```



### 5. 一个小笔记

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\步进电机小tips1.png" alt="步进电机小tips1" style="zoom:25%;" />

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\步进电机小tips2.png" alt="步进电机小tips2" style="zoom:25%;" />



在 step_motor.h 中定义了一个全局变量 STEP

```c
extern u8 STEP;
```

**但是在 step_motor.c 中还需要再声明一次**

```C
u8 STEP;
```



