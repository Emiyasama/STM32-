# STM32学习笔记（十一）

## 旋转编码器原理与驱动

### 一、 一些前提知识

1. 在原理图中可知旋转编码器P18和模拟摇杆P17是共有三个串口的，因此要使用P18时需要把P17的跳帽断开
2. 旋转编码器内部相当于有三个**微动开关**，按下时K1闭合，旋转时K2和K3先后闭合，可通过闭合的先后顺序判断旋转方向
3. 因为内部是机械式的微动开关，因此需要程序去抖

相关原理图如下：

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\旋转编码器原理图.jpg" style="zoom:33%;" />

​                                                                              图1 旋转编码器原理图

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\旋转编码器与串口.png" style="zoom:33%;" />

​                                                                     图2 旋转编码器对应串口 

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\旋转编码器引脚.jpg" style="zoom:33%;" />

​                                                                  图3 旋转编码器对应引脚图（放大）

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\旋转编码器内部等效图.jpg" style="zoom:33%;" />

​                                                                          图4  旋转编码器内部等效图

### 二、 分析程序

加入encoder.h与encoder.c

#### 1. main函数

```c
int main (void){//主程序
	u8 a=0,b=0,c=0x01;
    
	RCC_Configuration(); //系统时钟初始化 
	RTC_Config();  //RTC初始化
	ENCODER_Init(); //旋转编码器初始化
	TM1640_Init(); //TM1640初始化
    /**************************显示初始化的数值********************************/
	TM1640_display(0,a/10);        //a十位
	TM1640_display(1,a%10);        //a个位
	TM1640_display(2,20);          //息屏
	TM1640_display(3,20);          //息屏
	TM1640_display(4,20);          //息屏
	TM1640_display(5,20);          //息屏
	TM1640_display(6,20);          //息屏
	TM1640_display(7,20);          //息屏
    /**************************显示初始化的数值********************************/
    
	while(1){
		b=ENCODER_READ();	//读出旋转编码器值	
		if(b==1){a++;if(a>99)a=0;}         //b=1证明向右转，示值递增，当a>99即100时输出的是a=0
		if(b==2){if(a==0)a=100;a--;}       //b=2证明向左转，如果递减前a已经为0，证明下次再减就是-1了，此时另a=100再执行递减，则能保证a=0后的下一个数为99
		if(b==3)a=0;        //b=3证明旋转器按下，此时示值清零
		if(b!=0){           //b=0时指旋转器没操作，因此b不为0时指有旋转器的操作（即数值有改变）
			TM1640_display(0,a/10); //显示数值
			TM1640_display(1,a%10);
		}

//		TM1640_led(c); //与TM1640连接的8个LED全亮
//		c<<=1; //数据左移 流水灯
//		if(c==0x00)c=0x01; //8个灯显示完后重新开始
//		delay_ms(150); //延时
	}
}
```



#### 2. encoder.c中旋转编码器对应函数

对应函数包括：

- 旋转编码器初始化函数
- 读出旋转编码器值函数

#####  (1) encoder.h 配置

```c
#include "sys.h"
#include "delay.h"
#define ENCODER_PORT_A	GPIOA		//定义IO接口组
#define ENCODER_L	GPIO_Pin_6	//定义IO接口
#define ENCODER_D	GPIO_Pin_7	//定义IO接口

#define ENCODER_PORT_B	GPIOB		//定义IO接口组
#define ENCODER_R	GPIO_Pin_2	//定义IO接口


void ENCODER_Init(void);//初始化
u8 ENCODER_READ(void);
```

##### (2) encoder.c 内容

###### 旋转编码器初始化函数

```c
void ENCODER_Init(void){ //接口初始化
	GPIO_InitTypeDef  GPIO_InitStructure; //定义GPIO的初始化枚举结构	
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA|RCC_APB2Periph_GPIOB|RCC_APB2Periph_GPIOC,ENABLE);       
    GPIO_InitStructure.GPIO_Pin = ENCODER_L | ENCODER_D; //选择端口号                        
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU; //选择IO接口工作方式 //上拉电阻       
	GPIO_Init(ENCODER_PORT_A,&GPIO_InitStructure);	

    GPIO_InitStructure.GPIO_Pin = ENCODER_R; //选择端口号                        
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU; //选择IO接口工作方式 //上拉电阻       
	GPIO_Init(ENCODER_PORT_B,&GPIO_InitStructure);				
}
```

主要是设置ENCODER的L、D、R三个串口，对应的是K2、K1、K3



###### 读出旋转编码器值函数

```c
u8 KUP;//旋钮锁死标志（1为锁死）
u16 cou;

u8 ENCODER_READ(void){ //接口初始化
	u8 a;//存放按键的值
	u8 kt;
	a=0;
	if(GPIO_ReadInputDataBit(ENCODER_PORT_A,ENCODER_L))KUP=0;	//判断旋钮是否解除锁死
	if(!GPIO_ReadInputDataBit(ENCODER_PORT_A,ENCODER_L)&&KUP==0){ //判断是否旋转旋钮，同时判断是否有旋钮锁死
		delay_us(100);
		kt=GPIO_ReadInputDataBit(ENCODER_PORT_B,ENCODER_R);	//把旋钮另一端电平状态记录
		delay_ms(3); //延时
		if(!GPIO_ReadInputDataBit(ENCODER_PORT_A,ENCODER_L)){ //去抖
			if(kt==0){ //用另一端判断左或右旋转
				a=1;//右转
			}else{
				a=2;//左转
			}
			cou=0; //初始锁死判断计数器
			while(!GPIO_ReadInputDataBit(ENCODER_PORT_A,ENCODER_L)&&cou<60000){ //等待放开旋钮，同时累加判断锁死
				cou++;KUP=1;delay_us(20); //
			}
		}
	}
	if(!GPIO_ReadInputDataBit(ENCODER_PORT_A,ENCODER_D)&&KUP==0){ //判断旋钮是否按下  
		delay_ms(20);
		if(!GPIO_ReadInputDataBit(ENCODER_PORT_A,ENCODER_D)){ //去抖动
			a=3;//在按键按下时加上按键的状态值
			//while(ENCODER_D==0);	等等旋钮放开
		}
	}
	return a;
} 
```

程序流程：

1. 判断Left微动开关K2电平，为高电平1时为未按下，此时判断旋钮并未卡死，KUP=0
2. 再判断Left微动开关K2标志位的非，即K2按下且没卡死时执行以下操作（到第7步）
3. 短暂延时以表礼貌（doge）
4. 读取Right微动开关K3电平，存入kt
5. 再延时3ms（>2ms即可）判断K2是否按下，按下则执行以下操作（到第6步）
6. 判断kt值，若为0则K3按下，为右转a=1，反之为左转a=2
7. 然后等待K2松开，松开后往下执行
8. 判断K1按钮是否按下，按下则a=3
9. 最后返回a
10. 重新进入主循环，状态值a清零



**关于卡死问题**：

即可能按键按下后并未弹回高电平，此时程序可能会一直死循环在【等待K2松开】这一步，导致无法return a，因此要引入卡死标志KUP和卡死判断计数器cou，使每次都能顺利返回a



解决流程如下：

1. 最开始就判断K2是否为高电平1，为1时才能使KUP重新清零
2. KUP=0才能进入左右转判断与下按判断
3. 在【等待K2松开】这一步添加另一判断标准：卡死判断计数器cou>60000时也可跳出循环
4. 在【等待K2松开】这一步循环中另KUP=1
5. 跳出【等待K2松开】循环后 return a 并重新进入主循环
6. 重新进入主循环时会再次判断K2是否为高电平1，若是卡死，则KUP无法清0，则一直无法进入左右转判断与下按判断，而直接执行 return a = 0 （初始值）
7. 问题解决！



**关于旋转编码器扫描延时导致无法精准反应问题：**

1. 在主循环中即调用了**读出旋转编码器函数**又调用了**delay函数**
2. 则每次读出旋转编码器后都要delay一大段时间才能完成一次主循环，再次进入主循环第二次调用旋转编码器的时间就会大大增加
3. 因此在两次调用旋转编码器之间的时间内的操作将无法反应出来



解决思路：

- 思路1：把读出旋转编码函数套进delay函数的循环中，则每次delay时都会不断调用读出旋转编码函数
- 思路2：把读出旋转编码函数套进中断函数当中，只要一触发就可改变旋转编码器的状态值



**附录：**一些关于读出旋转编码器函数的理解图

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\读出旋转编码器理解图1.jpg" style="zoom:33%;" />

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\读出旋转编码器理解图2.jpg" style="zoom: 50%;" />
