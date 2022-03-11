# STM32学习笔记（十三）

## OLED屏原理与驱动程序

### 1. 原理与相关知识

#### （1）OLED与LCD区别

OLED省电，靠PWM控制具体点块的led开关

LCD耗电，但相对护眼，成本低

#### （2）OLED尺寸与字体尺寸

OLED尺寸为128*64

而英文字母与数字尺寸为8*16

中文字体为16*16

### 2. main函数

```c
	delay_ms(100); //上电时等待其他器件就绪
	OLED0561_Init(); //OLED初始化

	OLED_DISPLAY_8x16_BUFFER(0,"   EMIYAsama "); //显示字符串
	OLED_DISPLAY_8x16_BUFFER(6,"  Temp:"); //显示字符串

	while(1){
		LM75A_GetTemp(buffer); //读取LM75A的温度数据
			
		if(buffer[0])OLED_DISPLAY_8x16(6,7*8,'-'); //如果第1组为1即是负温度
		OLED_DISPLAY_8x16(6,8*8,buffer[1]/10+0x30);//显示温度值
		OLED_DISPLAY_8x16(6,9*8,buffer[1]%10+0x30);//
		OLED_DISPLAY_8x16(6,10*8,'.');//
		OLED_DISPLAY_8x16(6,11*8,buffer[2]/10+0x30);//
		OLED_DISPLAY_8x16(6,12*8,buffer[2]%10+0x30);//
		OLED_DISPLAY_8x16(6,13*8,'C');//

		delay_ms(200); //延时
```

知识点：

1. OLED_DISPLAY_8x16_BUFFER是显示字符串函数；OLED_DISPLAY_8x16是显示单个字符函数
2. 两函数参数1都为行数，BUFFER的参数2为要显示的字符串
3. OLED_DISPLAY_8x16参数2为列数
4. OLED_DISPLAY_8x16参数3是要输入的字符
5. 某数字对应字符的ascii码 = 此数字 + 0x30
6. 最开始的延时函数 delay_ms(100); 是为了等待温度传感器和显示屏启动，否则容易卡死



### 3. OLED驱动程序

所需程序文件有：

- ASCII_8X16.h
- oled0561.h
- oled0561.c

所需要的函数有：

- OLED0561_Init();         //OLED初始化
- OLED_DISPLAY_8x16_BUFFER
- OLED_DISPLAY_8x16

#### （1）OLED初始化函数

```c
void OLED0561_Init (void){//OLED屏开显示初始化
	OLED_DISPLAY_OFF(); //OLED关显示
	OLED_DISPLAY_CLEAR(); //清空屏幕内容
	OLED_DISPLAY_ON(); //OLED屏初始值设置并开显示

}
```

知识点：

1. 在刚上电时显示屏里的数据都是乱码，所以此时若是打开显示则图案为乱码
2. 因此初始化时需要关闭显示，然后清空屏幕内容，即把显示屏数据清零



拓展：OLED_DISPLAY_ON 和OFF 函数的大致实现方法：

1. 本质是要把一串厂家定好的指令发送给OLED屏
2. 发送指令的函数为 I2C_SAND_BYTE
3. 参数1为OLED0561_ADD，参数2 COM 指的是输入的是指令位，
4. 参数3是指令数组，参数4是指数组长度



OLED_DISPLAY_CLEAR(); 函数如下

```c
void OLED_DISPLAY_CLEAR(void){//清屏操作
	u8 j,t;
	for(t=0xB0;t<0xB8;t++){	//设置起始页地址为0xB0
		I2C_SAND_BYTE(OLED0561_ADD,COM,t); 	//页地址（从0xB0到0xB7）
		I2C_SAND_BYTE(OLED0561_ADD,COM,0x10); //起始列地址的高4位
		I2C_SAND_BYTE(OLED0561_ADD,COM,0x00);	//起始列地址的低4位
		for(j=0;j<132;j++){	//整页内容填充
 			I2C_SAND_BYTE(OLED0561_ADD,DAT,0x00);
 		}
	}
}
```

知识点：

1. 本质是把每行每列清零
2. OLED屏把64行像素点分为8大行，分别是0xB0-0xB7
3. for循环里第一句是输入指令定位行（t）
4. for循环里第二第三句是输入指令定位列（总共有128个像素点列）
5. 像素点列有128个但驱动芯片中控制行数的数组有132行，因此要循坏清空132次



#### （2）显示亮度函数

```c
void OLED_DISPLAY_LIT (u8 x){//OLED屏亮度设置（0~255）
	I2C_SAND_BYTE(OLED0561_ADD,COM,0x81);
	I2C_SAND_BYTE(OLED0561_ADD,COM,x);//亮度值
}
```



#### （3）写入字符函数

```c
void OLED_DISPLAY_8x16(u8 x, //显示汉字的页坐标（从0到7）（此处不可修改）
						u8 y, //显示汉字的列坐标（从0到63）
						u16 w){ //要显示汉字的编号
	u8 j,t,c=0;
	y=y+2; //因OLED屏的内置驱动芯片是从0x02列作为屏上最左一列，所以要加上偏移量
	for(t=0;t<2;t++){
		I2C_SAND_BYTE(OLED0561_ADD,COM,0xb0+x); //页地址（从0xB0到0xB7）
		I2C_SAND_BYTE(OLED0561_ADD,COM,y/16+0x10); //起始列地址的高4位
		I2C_SAND_BYTE(OLED0561_ADD,COM,y%16);	//起始列地址的低4位
		for(j=0;j<8;j++){ //整页内容填充
 			I2C_SAND_BYTE(OLED0561_ADD,DAT,ASCII_8x16[(w*16)+c-512]);//为了和ASII表对应要减512
			c++;
        }
        x++; //页地址加1
	}
}
```

知识点：

1. 总共用两个for循环定位写入对应行和列
2. 一次写入一个字节（写入某一列的8行像素点）
3. 第一个for（外围）控制依次写入两次大行（一大行是8个像素点行）
4. 第二个for（内围）控制依次写入8列数据
5. 利用ASCII_8x16写入的数据是固定算法



#### （4）写入字符串函数

```c
void OLED_DISPLAY_8x16_BUFFER(u8 row,u8 *str){
	u8 r=0;
	while(*str != '\0'){
		OLED_DISPLAY_8x16(row,r*8,*str++);
		r++;
    }	
}
```

知识点：

1. 这里的*str++的意思是把 *str的内容输入进函数，然后str++

### 4. 拓：OLED显示汉字与图片

所需文件：

- ASCII_8x6.h（字母与数字库）
- CHS_16x16.h（中文库（只加了要用的汉字））
- oled0561.h
- oled0561.c（加了驱动中文与图片的函数）
- PIC1.h（图片库）

#### （1）main函数

```c
OLED0561_Init(); //OLED初始化
	OLED_DISPLAY_LIT(100);//亮度设置

	OLED_DISPLAY_PIC1();//显示全屏图片
	delay_ms(1000); //延时
	OLED_DISPLAY_CLEAR();
	OLED_DISPLAY_8x16_BUFFER(0,"   YoungTalk "); //显示字符串
	OLED_DISPLAY_8x16_BUFFER(6,"  Temp:"); //显示字符串

	OLED_DISPLAY_16x16(2,2*16,0);//汉字显示	 
	OLED_DISPLAY_16x16(2,3*16,1);          //0  1  2  3
	OLED_DISPLAY_16x16(2,4*16,2);         //洋  桃 电  子
	OLED_DISPLAY_16x16(2,5*16,3);

	while(1){
		LM75A_GetTemp(buffer); //读取LM75A的温度数据
			
		if(buffer[0])OLED_DISPLAY_8x16(6,7*8,'-'); //如果第1组为1即是负温度
		OLED_DISPLAY_8x16(6,8*8,buffer[1]/10+0x30);//显示温度值
		OLED_DISPLAY_8x16(6,9*8,buffer[1]%10+0x30);//
		OLED_DISPLAY_8x16(6,10*8,'.');//
		OLED_DISPLAY_8x16(6,11*8,buffer[2]/10+0x30);//
		OLED_DISPLAY_8x16(6,12*8,buffer[2]%10+0x30);//
		OLED_DISPLAY_8x16(6,13*8,'C');//

		delay_ms(200); //延时
	}
```

其中利用到的函数有：

- OLED_DISPLAY_PIC1();        //显示全屏图片
- OLED_DISPLAY_16x16        //汉字显示

这两个函数存在oled0561.c中



#### （2）显示汉字函数

```c
void OLED_DISPLAY_16x16(u8 x, //显示汉字的页坐标（从0xB0到0xB7）
			u8 y, //显示汉字的列坐标（从0到63）
			u16 w){ //要显示汉字的编号
	u8 j,t,c=0;
	for(t=0;t<2;t++){
		I2C_SAND_BYTE(OLED0561_ADD,COM,0xb0+x); //页地址（从0xB0到0xB7）
		I2C_SAND_BYTE(OLED0561_ADD,COM,y/16+0x10); //起始列地址的高4位
		I2C_SAND_BYTE(OLED0561_ADD,COM,y%16);	//起始列地址的低4位
		for(j=0;j<16;j++){ //整页内容填充
 			I2C_SAND_BYTE(OLED0561_ADD,DAT,GB_16[(w*32)+c]);
			c++;}x++; //页地址加1
	}
	I2C_SAND_BYTE(OLED0561_ADD,COM,0xAF); //开显示 
}
```

- 道理与显示字母函数相同

- 需要#include “CHS_16x16.h”
- 第二个for（内部for）退出条件是 j<16（因为汉字宽度为16个像素点）



#### （3）显示全屏图片

```c
void OLED_DISPLAY_PIC1(void){ //显示全屏图片
	u8 m,i;
	for(m=0;m<8;m++){//
		I2C_SAND_BYTE(OLED0561_ADD,COM,0xb0+m);
		I2C_SAND_BYTE(OLED0561_ADD,COM,0x10); //起始列地址的高4位
		I2C_SAND_BYTE(OLED0561_ADD,COM,0x02);	//起始列地址的低4位
		for(i=0;i<128;i++){//送入128次图片显示内容
			I2C_SAND_BYTE(OLED0561_ADD,DAT,PIC1[i+m*128]);}
	}
}
```

- 同理把图片对应的ASCII码存入PIC1.h里的PIC1数组里，此处用于调用
- 因图片为全屏显示，因此行为0-7，列为0-127



### 5. 关于取模软件

取模软件用于获得字母、数字、符号、汉字、图片对应的ASCII码表

一些参数设定：

1. 数据排列顺序：从左到右从上到下

2. 取模方式：纵向8点下高位

3. ASC/汉字选择：按要求自选

4. 字库选择：按要求自选

5. 图片截取范围：按要求设置X、Y

6. 载入图片：必须是bmp格式，必须是无灰度的单色位图，尺寸要和设置的X、Y相同

7. 数据保存：会存为.h文件，可打开复制数组数据

   最后点击参数确认即可

PS：当需要在原有的汉字数组内加入新汉字时，需要记得在原先最后一个数据后面加个逗号“，”再在下面行内添加新汉字数组

