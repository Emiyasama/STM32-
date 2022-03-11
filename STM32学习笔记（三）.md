# STM32学习笔记（三）

## 1. 按键控制LED程序

### 第一步：初始化LED和按键

本质都是设置GPIO端口，初始化画LED在之前已经有讲过了

下面是按键初始化函数KEY_Init()：

```c
void KEY_Init(void){ //微动开关的接口初始化
	GPIO_InitTypeDef  GPIO_InitStructure; //定义GPIO的初始化枚举结构，GPIO_InitTypeDef是已经定义好的枚举结构，GPIO_InitStructure是枚举变量
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA,ENABLE);       
    GPIO_InitStructure.GPIO_Pin = KEY1 | KEY2; //选择端口号，这里叫KEY1和KEY2是因为key.h里define过它们是什么含义                    
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU; //选择IO接口工作模式：上拉电阻(即常态为高电平，按下开关为低电平)    
//这里不用写GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz; // 设置IO接口速度，一般是输出端口要写
	GPIO_Init(KEYPORT,&GPIO_InitStructure);//初始化函数			
}
```

因此，在main.c中注意要导入key.h文件

```c
#include "key.h"
```



### 第二步：读取按键输入

利用库函数读取按键输入

![image-20220120204247530](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220120204247530.png)

其中：

- GPIO_ReadInputDatabit：比如读取PA1端口的输入
- GPIO_ReadInputData：比如读取PA这16个端口上的输入

 

### 第三步：各种实现方式

#### 方法1：无锁存（复杂版）

即按下时灯亮，松手时灯灭

```c
if(GPIO_ReadInputDataBit(KEYPORT,KEY1)){ //这里利用库函数读取KEY1上的电平，当电平为1，即if为真，执行如下
	GPIO_ResetBits(LEDPORT,LED1); //让LED1输出0（因为按键不按时电平为1，此时LED即0）
}else{	//当电平为0时，即按键按下时，执行如下
    	GPIO_SetBits(LEDPORT,LED1); //让LED1输出为1
}
```

#### 方法2：无锁存（简化版）

即把方法1的代码合成成一句

```c
GPIO_WriteBit(LEDPORT,LED1,(BitAction)(!GPIO_ReadInputDataBit(KEYPORT,KEY1)));
```

#### 方法3：有锁存

首先需要考虑的是**按键抖动**，即按下去之后的10-20ms之内电平会上下抖动，而非稳定电平，此时读取的电平值是不准确的，解决方法就是用delay_ms等待20ms

```c

if(!GPIO_ReadInputDataBit(KEYPORT,KEY1)){ //当输入电平为0，即按下时
	//延时20ms再判断一次
    delay_ms(20); 
	if(!GPIO_ReadInputDataBit(KEYPORT,KEY1)){ //第二次判断，如果电平还是0则证明的确是按下了
        //取反操作：原来LED亮则写入0，原来LED暗则写入1
		GPIO_WriteBit(LEDPORT,LED1,(BitAction)(1-GPIO_ReadOutputDataBit(LEDPORT,LED1))); 
        //等待按键松开：只用松开（电平恢复1）时才能跳出这个死循环
		while(!GPIO_ReadInputDataBit(KEYPORT,KEY1)); 
	}
}
```

#### 方法4：有锁存（在2个LED上显示二进制加法）

本质是利用了GPIO_Write函数的功能：可把数据写入端口数据寄存器

1. 函数里设置的需写入的值可以是任意进制的（只要能看得出来）
2. 写入时会自动转成二进制，写入端口（如PA或者PB的所有IO口），通过1和0来保存
3. 只需要把写入的端口设置成LED的端口即可实现控制多个LED的亮灭了

```c
//当按键按下时
if(!GPIO_ReadInputDataBit(KEYPORT,KEY1)){ 
	//延时
    delay_ms(20); 
    //再次判断按下
	if(!GPIO_ReadInputDataBit(KEYPORT,KEY1)){ 
		//默认a初值为0，此时a++即保证了 a = 按键次数
		a++; 
        //只有两个灯，因此最大计数为3，所以次数到3时清零
		if(a>3){ 
					a=0; 
				}
        //把按键次数写入LED所在的端口
		GPIO_Write(LEDPORT,a); 
        //等待按键松开
		while(!GPIO_ReadInputDataBit(KEYPORT,KEY1)); 
		}
}
```



## 2.Flash读写程序

在按键控制LED灯的基础上，增加了“记忆断电前LED灯状态”的功能

### 第一步：导入flash.c和flash.h

flash.h就是给flash.c中的函数做了定义

而flash.c包含了**flash写入函数**和**flash读取函数**

这两个函数利用了FLASH库函数，集体说明如下：

![image-20220120223538499](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220120223538499.png)

#### flash写入函数：

1. flash操作必须要在在高速时钟开启时使用，一般都已经开了
2. 要先解锁才能写
3. 每次写入之前都要先擦除，而且是以整页的方式擦除
4. 每一页有1024个地址
5. 指定要擦除的地址，函数即可帮你擦除地址所在页的一整页
6. 写入时需要输出地址参数与数据参数
7. 清除标志位
8. 最后记得要上锁

程序如下：

```c
//flash写入函数
void FLASH_W(u32 add,u16 dat){ 
     //RCC_HSICmd(ENABLE); //用于开启高速时钟
     //解锁FLASH
	 FLASH_Unlock();  
     //清除标志位
     FLASH_ClearFlag(FLASH_FLAG_BSY|FLASH_FLAG_EOP|FLASH_FLAG_PGERR|FLASH_FLAG_WRPRTERR);
     //擦除地址所在页
     FLASH_ErasePage(add); 
     //写入数据进指定地址
     FLASH_ProgramHalfWord(add,dat); 
     //清除标志位
     FLASH_ClearFlag(FLASH_FLAG_BSY|FLASH_FLAG_EOP|FLASH_FLAG_PGERR|FLASH_FLAG_WRPRTERR);
     //上锁
     FLASH_Lock();    
}
```

#### flash读取函数：

```c
u16 FLASH_R(u32 add){ 
	u16 a;
    //这个操作可使从指定页的addr地址开始读数据，然后存入a中
    a = *(u16*)(add);
return a;
}
```



### 第二步：在主循环前读取FLASH

1. 用FLASH_R读取出来
2. 然后用GPIO_Write写入到LED端口中

```c
    a = FLASH_R(FLASH_START_ADDR);
	GPIO_Write(LEDPORT,a);
```



### 第三步：把按键次数a写入FLASH

在每次按键次数a写入端口后，也写入FLASH

```c
FLASH_W(FLASH_START_ADDR,a);
```

其中FLASH_START_ADDR即为宏定义的写入地址

其实地址是0x08000000 

**注意**：FLASH写入的数据一定要是16位的，因此a一开始要设定成u16



## 3.蜂鸣器驱动程序 

**有源蜂鸣器与无缘蜂鸣器**：

- 有源：内置频率发生电路，通电既能发声，声音频率固定

- 无源：内部没有发生电路，由外部控制发声频率

### 第一步：添加buzzer.h和buzzer.c

在工程树内合适位置添加文件

buzzer.h即定义文件

buzzer.c即包括**蜂鸣器初始化函数**和**蜂鸣器响声函数**

#### 蜂鸣器响声函数

1. 用for循环使现象保持一段时间
2. 蜂鸣器接口写入0
3. 延时（延时的时间决定了蜂鸣器的频率）
4. 蜂鸣器接口写入1
5. 延时



### 第二步：蜂鸣器初始化

要注意这里初始化了PB5，因此如果数据写入PB端口，可能会修改到PB5的电平

因此在把FLASH内的LED状态重新写入时需要做如下运算：

```c
GPIO_Write(LEDPORT,a|0xfffc&GPIO_ReadOutputData(LEDPORT));
```



### 第三步：把BUZZER_BEEP1()函数放在想要的位置

把BUZZER_BEEP1()函数放在每次按键后，能在每次按下后听到提示音



## 4.MIDI音乐播放程序

 ![image-20220121151431789](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220121151431789.png)

### 第一步：编写MIDI_PLAY()函数

1. 利用for循环来切换不同的音符（递增变量为i）
2. 在for循环内，设置此音符的频率与时间长度
3. 设置频率：即控制开蜂鸣器与关蜂鸣器的延时时间
4. 设置音符时间长度：需要再在里面套一个for循环控制时间（递增变量为e）

#### 1.**那么如何编写音乐？**

1. 声明一个只读数组music1，写入数据：奇数位为音调，偶数位为时间长度（ms）

#### 2.**延时函数怎么控制音调？**

1. 我们知道高低电平频率和音调的对应关系
2. 因此只要通过频率换算出低电平和高电平的延时时间即可
3. 比如频率 f 为330Hz，则证明1s有330个低电平和330个高电平
4. 我们假设高电平和低电平的总时间相同，那么他们分别占0.5s
5. 因此每一次高电平延时应为（500000us/330Hz），低电平同理
6. 因此延时函数与音乐数组的关系为：delay_us(500000/music1[i*2])
7. PS：这里取了数组的第i*2位是因为数组是从第0位开始的，而for循环中 i 即是从0开始的，相当于取了数组的第0、2、4....位（奇数位）

#### 3.**第二个for循环怎么控制音调持续时间？**

1. 首先我们希望音调持续的时间是music1[i*2+1]，单位是毫秒
2. 因此我们需要知道执行一次for循环需要多长时间
3. 我们知道每一次循环里延时的时间是（1s/music1[i*2]）
4. 我们希望的延时music1[i*2+1]毫秒，因此有等式如下

```c
//最终延时总时间 = 一次循环的延时时间 * 循环次数
t_all = t_once * cishu
```

5. 因此最终设置e循环的次数是

```c
for(e=0;e<music1[i*2]*music1[i*2+1]/1000;e++){}
```

#### 4.最终MIDI_PLAY函数

```c
void MIDI_PLAY(void){ 
	u16 i,e;
	for(i=0;i<39;i++){
		for(e=0;e<music1[i*2]*music1[i*2+1]/1000;e++){
			GPIO_WriteBit(BUZZERPORT,BUZZER,(BitAction)(0)); 
			delay_us(500000/music1[i*2]);	
			GPIO_WriteBit(BUZZERPORT,BUZZER,(BitAction)(1)); 
			delay_us(500000/music1[i*2]); 
		}	
	}
}
```







