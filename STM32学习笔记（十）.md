# STM32学习笔记（十）

## 数码管原理与驱动程序

### 数码管原理

我们此处使用了一个数码管驱动芯片，芯片原理图如右：        <img src="F:\电设赛资料\原理图设计Multisim与相关基础\单片机相关\数码管驱动芯片.png" style="zoom:25%;" />

而数码管出输入口如右：                                                                  <img src="F:\电设赛资料\原理图设计Multisim与相关基础\单片机相关\数码管输出口图.png" style="zoom:25%;" />                    

我们在这两者之间采用了一个跳线控制连接与否：          <img src="F:\电设赛资料\原理图设计Multisim与相关基础\单片机相关\数码管跳线.png" style="zoom:25%;" />

**注意重点**：

1. 数码管输入为TIM1640_DIN（数据输入）和TIM1640_SCLK（时钟）
2. 驱动芯片中的GR1-GR8控制的为8位数码管的第几位数码管
3. 驱动芯片中的GR9为LED1-LED8的负极
4. 驱动芯片中的SEG1-SEG8控制的为数码管八段位置中的的第几段条状LED灯，
5. 以及，SEG1-SEG8同时是LED1-LED8的正极



### 主程序：

```c
	u8 c=0x01;
	while(1){
		if(RTC_Get()==0){ //读出RTC时间
			TM1640_display(0,rday/10);	//天
			TM1640_display(1,rday%10+10);
			TM1640_display(2,rhour/10); //时
			TM1640_display(3,rhour%10+10);
			TM1640_display(4,rmin/10);	//分
			TM1640_display(5,rmin%10+10);
			TM1640_display(6,rsec/10); //秒
			TM1640_display(7,rsec%10);

			TM1640_led(c); //与TM1640连接的8个LED全亮
			c<<=1; //数据左移 流水灯
			if(c==0x00)c=0x01; //8个灯显示完后重新开始
			delay_ms(125); //延时
		}
	}
```

1. TM1640_display的参数1指：八位数码管的第【参数1】+1位

2. **rday%10+10中的+10表此位数码管的右下角小数点点亮**
3. 对于单片机来说0x01指给**SEG8-SEG1**这八个端口分别设置为0,0,0,0,0,0,0,1
4. 而c<<=1; 指数据左移，即LED1-8依次被设为1，因此也依次点亮，实现流水灯功能
5. 最终效果为8位数码管上显示RTC时钟的时间（在日历程序中可以改），而下方LED显示流水灯效果
6. 若想关闭某一位数码管的显示，则把TM1640_display的参数2调为20，如下

```c
TM1640_display(7,20);  //让第8位数码管熄灭
```

7. 若想让LED全部点亮，则把TM1640_led(c)的参数c为0xff，即二进制数为11111111



### 驱动程序

即TM1640.c中自设的函数

#### TM1640.h中的定义如下

```c
#define TM1640_GPIOPORT	GPIOA	//定义IO接口
#define TM1640_DIN	GPIO_Pin_12	//定义IO接口
#define TM1640_SCLK	GPIO_Pin_11	//定义IO接口

void TM1640_Init(void);//初始化
void TM1640_led(u8 date);//
void TM1640_display(u8 address,u8 date);//
void TM1640_display_add(u8 address,u8 date);//不推荐
```

#### TM1640.c如下：

1. 添加头文件

```c
#include "TM1640.h"
#include "delay.h"
```

2. 关于通信速度

```c
#define DEL  1   //宏定义 通信速率（默认为1，如不能通信可加大数值）
```

约小通信约快，最小为1，忘大调可调2、3等整数

一般通信速度过快易出现通信不稳定

3. 地址模式设置

一般分为**固定地址模式**和**自动加一模式**

- 固定地址模式：参数1为地址，参数2为要输入的数
- 自动加一模式：只有参数1为当前地址要输入的数，每次调用完地址自动加1

4. 显示亮度的设置

```c
//显示亮度的设置
//#define TM1640MEDO_DISPLAY  0x88   //宏定义 亮度  最小
//#define TM1640MEDO_DISPLAY  0x89   //宏定义 亮度
//#define TM1640MEDO_DISPLAY  0x8a   //宏定义 亮度
//#define TM1640MEDO_DISPLAY  0x8b   //宏定义 亮度
#define TM1640MEDO_DISPLAY  0x8c   //宏定义 亮度（推荐）
//#define TM1640MEDO_DISPLAY  0x8d   //宏定义 亮度
//#define TM1640MEDO_DISPLAY  0x8f   //宏定义 亮度 最大
```

而当亮度为0x80时，数码管即为关闭的亮度，我们可以宏定义标记一下这个的意思，如下：

```c
#define TM1640MEDO_DISPLAY_OFF  0x80   //宏定义 亮度 关
```

5. 底层通信协议函数

- TM1640_start()
- TM1640_stop()
- TM1640_write()

不用学内容，暂时会用就好

6. 初始化函数TM1640_Init(void)

- 设置端口、工作方式、接口速度

- 把TM1640_DIN和TM1640_SCLK接口设置为输出高电平1
- 调用一些底层函数，以及用TM1640_write设置地址模式与显示亮度

7. 控制8个LED灯输出函数TM1640_led(u8 date);

```c
void TM1640_led(u8 date){ //固定地址模式的显示输出8个LED控制
   TM1640_start();                          //开始通信
   TM1640_write(TM1640_LEDPORT);	        //传显示数据对应的地址（LED负极接口地址）
   TM1640_write(date);	                    //传1BYTE显示数据
   TM1640_stop();                           //结束通信
}
```

8. 固定地址模式的显示输出TM1640_display(u8 address,u8 date)

```c
void TM1640_display(u8 address,u8 date){ //固定地址模式的显示输出
 	const u8 buff[21]={0x3f,0x06,0x5b,0x4f,0x66,0x6d,0x7d,0x07,0x7f,0x6f,0xbf,0x86,0xdb,0xcf,0xe6,0xed,0xfd,0x87,0xff,0xef,0x00};//数字0~9及0~9加点显示段码表
// 0    1    2    3    4    5    6    7    8    9    0.   1.   2.   3.   4.   5.   6.  7.     8.   9.   无   
   TM1640_start();
   TM1640_write(0xC0+address);	         //传显示数据对应的地址
//0xc0对应数码管第一位的地址，0xC0+address即为数码管第adress位的地址
   TM1640_write(buff[date]);				 //传1BYTE显示数据
   TM1640_stop();
}
```



