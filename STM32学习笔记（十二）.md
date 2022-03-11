# STM32学习笔记（十二）

## I2C总线

### 1. I2C原理

定义：是一种通信总线。分别有两条线：SDA数据线、SCL时钟线

- 属于板级总线，只能在同一块PCB板上进行通信，连接线一般不超过2m
- I2C的数据线上理论上需要加2K的上拉电阻
- 分主设备与从设备，主设备一般为单片机STM32，从设备的SDA和SCL与主设备并联相连
- 所有设备与单片机需要共地

如图所示：

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\I2C结构图.png" style="zoom: 33%;" />

### 2. I2C要点：

- 电路连接：由两线总线连接，总线上需要有1~10KΩ上拉电阻，与I2C复用的IO端口需要设置为复用开漏模式
- 器件地址：每个器件都有唯一的地址（7位的16进制数）



### 3. 一些工程tips

要编写添加I2C驱动文件：i2c.h和i2c.c

要编写添加温度传感器驱动文件：lm75a.h和lm75a.c

要添加stm32f10x的I2C固件库驱动程序：stm32f10x_i2c.c



### 4. I2C驱动程序分析

#### （1）工程文件组成关系

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\工程文件组成关系.jpg" style="zoom:33%;" />



#### （2）I2C初始化函数

1. 在i2c.h文件中定义如下：

```c
#define HostAddress	0xc0	//总线主机的器件地址
#define BusSpeed	200000	//总线速度（不高于400000）
```

总线速度过快容易卡死（数据出错）导致通信不稳定

2. 先编写I2C接口初始化函数（后续初始化会调用它）

主要内容：设置IO端口

```c
void I2C_GPIO_Init(void){ //I2C接口初始化
 GPIO_InitTypeDef  GPIO_InitStructure;  //结构体
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA|RCC_APB2Periph_GPIOB|RCC_APB2Periph_GPIOC,ENABLE);       
 RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1, ENABLE); //启动I2C功能 
 GPIO_InitStructure.GPIO_Pin = I2C_SCL | I2C_SDA; //选择端口号                      
 GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_OD; //选择IO接口工作方式       
 GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz; //设置IO接口速度（2/10/50MHz）    
 GPIO_Init(I2CPORT, &GPIO_InitStructure);
}
```



3. 再编写I2C初始化函数（里面调用I2C接口初始化函数）

主要内容：设置I2C功能

```c
void I2C_Configuration(void){ //I2C初始化
	I2C_InitTypeDef  I2C_InitStructure;
	I2C_GPIO_Init(); //先设置GPIO接口的状态
	I2C_InitStructure.I2C_Mode = I2C_Mode_I2C;//设置为I2C模式
	I2C_InitStructure.I2C_DutyCycle = I2C_DutyCycle_2;
	I2C_InitStructure.I2C_OwnAddress1 = HostAddress; //主机地址（从机不得用此地址）
	I2C_InitStructure.I2C_Ack = I2C_Ack_Enable;//允许应答
	I2C_InitStructure.I2C_AcknowledgedAddress = I2C_AcknowledgedAddress_7bit; //7位地址模式
	I2C_InitStructure.I2C_ClockSpeed = BusSpeed; //总线速度设置 	
	I2C_Init(I2C1,&I2C_InitStructure);
	I2C_Cmd(I2C1,ENABLE);//开启I2C					
}
```



#### （3）I2C发送数据串函数

利用指针与参数四（数据长度）依次发送数据

```c
void I2C_SAND_BUFFER(u8 SlaveAddr,u8 WriteAddr,u8* pBuffer,u16 NumByteToWrite){ //I2C发送数据串（器件地址，寄存器，内部地址，数量）
	I2C_GenerateSTART(I2C1,ENABLE);//产生起始位
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_MODE_SELECT)); //清除EV5
	I2C_Send7bitAddress(I2C1,SlaveAddr,I2C_Direction_Transmitter);//发送器件地址
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED));//清除EV6
	I2C_SendData(I2C1,WriteAddr); //内部功能地址
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED));//移位寄存器非空，数据寄存器已空，产生EV8，发送数据到DR既清除该事件
	while(NumByteToWrite--){ //循环发送数据	
		I2C_SendData(I2C1,*pBuffer); //发送数据
		pBuffer++; //数据指针移位
		while (!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED));//清除EV8
	}
	I2C_GenerateSTOP(I2C1,ENABLE);//产生停止信号
}
```

**发送数据串片段的单独分析：**

```c
while(NumByteToWrite--){ //循环发送数据，直到数据长度清0	
		I2C_SendData(I2C1,*pBuffer); //发送数据
		pBuffer++; //数据指针移位
		while (!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED));//等待数据发送完成。清除EV8
	}
```

当NumByteToWrite减为0时跳出循环

#### （4）I2C发送一个字节函数

- 参数1：器件地址

- 参数2：器件内要写入数据的具体寄存器地址

- 参数3：具体要写入的数据

数据传输格式如下图：

<img src="F:\电设赛资料\原理图设计Multisim与单片机笔记\单片机相关\数据传输格式.jpg" style="zoom:33%;" />

程序即按照上图数据传输的顺序一步步执行

```c
void I2C_SAND_BYTE(u8 SlaveAddr,u8 writeAddr,u8 pBuffer){ //I2C发送一个字节（从地址，内部地址，内容）
	I2C_GenerateSTART(I2C1,ENABLE); //发送开始信号
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_MODE_SELECT)); //等待完成	
	I2C_Send7bitAddress(I2C1,SlaveAddr, I2C_Direction_Transmitter); //发送从器件地址及状态（写入）
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED)); //等待完成	
	I2C_SendData(I2C1,writeAddr); //发送从器件内部寄存器地址
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED)); //等待完成	
	I2C_SendData(I2C1,pBuffer); //发送要写入的内容
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED)); //等待完成	
	I2C_GenerateSTOP(I2C1,ENABLE); //发送结束信号
}
```



#### （5）I2C读取数据串函数

**单独分析读取数据串的程序片段**：

```c
while(NumByteToRead){ //循环次数为数据长度（递减步骤在循最后）
		if(NumByteToRead == 1){ //只剩下最后一个数据时进入 if 语句
			I2C_AcknowledgeConfig(I2C1,DISABLE); //最后有一个数据时关闭应答位
			I2C_GenerateSTOP(I2C1,ENABLE);	//最后一个数据时使能停止位
		}
		if(I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_RECEIVED)){ //读取数据
			*pBuffer = I2C_ReceiveData(I2C1);//调用库函数将数据取出到 pBuffer
			pBuffer++; //指针移位
			NumByteToRead--; //字节数减 1 
		}
	}
```



#### （6）I2C读取一个字节函数

```c
u8 I2C_READ_BYTE(u8 SlaveAddr,u8 readAddr){ //I2C读取一个字节
	u8 a;
	while(I2C_GetFlagStatus(I2C1,I2C_FLAG_BUSY));//等待总线空闲
	I2C_GenerateSTART(I2C1,ENABLE);//发送起始信号
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_MODE_SELECT));//等待
	I2C_Send7bitAddress(I2C1,SlaveAddr, I2C_Direction_Transmitter);//发送器件地址（参数1） 
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED));//等待
	I2C_Cmd(I2C1,ENABLE);//开启I2C1的功能
	I2C_SendData(I2C1,readAddr);//发送器件子地址（参数2）
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_BYTE_TRANSMITTED));
	I2C_GenerateSTART(I2C1,ENABLE);//允许其他器件朝I2C1发送起始命令的数据
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_MODE_SELECT));
	I2C_Send7bitAddress(I2C1,SlaveAddr, I2C_Direction_Receiver);//发送器件地址（参数1）[同第六行]
	while(!I2C_CheckEvent(I2C1,I2C_EVENT_MASTER_RECEIVER_MODE_SELECTED));//等待接收到数据
	I2C_AcknowledgeConfig(I2C1,DISABLE); //最后有一个数据时关闭应答位
	I2C_GenerateSTOP(I2C1,ENABLE);	//最后一个数据时使能停止位
	a = I2C_ReceiveData(I2C1);
	return a;
}
```



#### （7）读取温度值函数

查找温度寄存器Temp（指针位0x0H）可知:

温度寄存器有两个字节的存储空间（D0-D15位）

- D15是正负号位：0为正，1为负
- D14-D8是温度整数部分：分别代指的温度是64、32、16、8、4、2、1度
- D7-D5是温度小数部分：分别指代的温度是0.5、0.25、0.12度

PS：若D15为1，即温度是负值时需要把后面的温度数据转换为补码

```c
t = ~t;t++
```

总体函数如下：

```c
//读出LM75A的温度值（-55~125摄氏度）
//温度正负号（0正1负），温度整数，温度小数（点后2位）依次放入*Tempbuffer（十进制）
void LM75A_GetTemp(u8 *Tempbuffer){   
    u8 buf[2]; //温度值储存   
    u8 t=0,a=0;   
    I2C_READ_BUFFER(LM75A_ADD,0x00,buf,2); //读出温度值（器件地址，子地址，数据储存器，字节数）
	t = buf[0]; //处理温度整数部分，0~125度
	*Tempbuffer = 0; //温度值为正值
	if(t & 0x80){ //判断温度是否是负（MSB表示温度符号）
		*Tempbuffer = 1; //温度值为负值
		t = ~t; t++; //计算补码（原码取反后加1）
	}
	if(t & 0x01){ a=a+1; } //从高到低按位加入温度积加值（0~125）
	if(t & 0x02){ a=a+2; }
	if(t & 0x04){ a=a+4; }
	if(t & 0x08){ a=a+8; }
	if(t & 0x10){ a=a+16; }
	if(t & 0x20){ a=a+32; }
	if(t & 0x40){ a=a+64; }
	Tempbuffer++;           //指针后移一个字节
	*Tempbuffer = a;        //把整数部分存入此指针的位置
	a = 0;           
	t = buf[1]; //处理小数部分，取0.125精度的前2位（12、25、37、50、62、75、87）
	if(t & 0x20){ a=a+12; }
	if(t & 0x40){ a=a+25; }
	if(t & 0x80){ a=a+50; }
	Tempbuffer++;           //指针后移一个字节
	*Tempbuffer = a;        //把小数部分存入此指针的位置
}
```



#### （8）掉电模式函数

查找配置寄存器Conf表格（指针位0x1H）可知：第B0位是选择器件工作模式的寄存器地址

当此位为1时关断（即进入掉电模式）

当此位为0时为正常工作模式

因此掉电模式函数如下：

```
void LM75A_POWERDOWN(void){
	I2C_SAND_BYTE(LM75A_ADD,0x01,1);
}
```











