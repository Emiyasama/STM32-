# STM32学习笔记（九）

## 触摸按键滑动程序

### 1、触摸按键物理结构

<img src="C:\Users\Administrator\AppData\Local\Temp\WeChat Files\7513b520fae8d88899d55e0fd8ab376.jpg" alt="7513b520fae8d88899d55e0fd8ab376" style="zoom: 33%;" />

判断原理如上：即判断是否有两个按键同时被按下的情况，具体程序实现如下

### 2、程序实现方法

1. main函数内分别编写A、B、C、D的判断程序
2. 每一个按键的程序都有对于长按、滑动、双击、单击的判断（判断顺序不能变）
3. 对于滑动的判断方法为：（以A为例）若判断A处不为长按，则判断B处是否被按下（若按下则为滑动，反之则不为滑动，往下判断其余两种）
4. 若不为滑动（a=0），则往下判断双击和单击，方法见**STM32学习笔记（八）**

#### 对A判断滑动的代码：

```c
else{ //判断完不为长按
					if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_B)){
						k++; //用于显示的计数值
						printf("A键右滑 %d \r\n",k); 
						a=1;s=1; //a是单双击判断标志，s是刚刚结束滑动标志
					}//if
```

#### 对B判断滑动的代码：

```c
else{ //单击处理
					if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_C)){
						k++;
						printf("B键右滑 %d \r\n",k); 
						a=1;s=1; //a是单双击判断标志，s是刚刚结束滑动标志
					}
					if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A)){
						k--;
						printf("B键左滑 %d \r\n",k); 
						a=1;s=1; //a是单双击判断标志，s是刚刚结束滑动标志
					}
```



#### 对于【滑动后松开的那个键会被判定为单击】问题的解决方案：

**做法：**在判断单击的程序中增加一步判断是否为滑动后的最后那个键

代码如下：

```c
if(a==0){ //判断单击
		if(s==1){ //判断是不是刚执行完滑动操作
				s=0; //如果是则本次不执行单击处理（因为是滑动的放开操作）
		}else{	 //如果不是，则正常执行单击处理
				 //单击后执行的程序放到此处
				GPIO_WriteBit(LEDPORT,LED1|LED2,(BitAction)(0));//LED控制
				printf("B键单击 \r\n");
			 }
```



