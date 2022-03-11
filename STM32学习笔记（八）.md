# STM32学习笔记（八）

## 触摸按键的双击和长按

如何判断单击双击与长按：

```c
u8 a=0,b,c=0;	
while(1){

		if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A)){ //检测按键是否按下
			delay_ms(20); //延时去抖动
			if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A)){//判断长短键
				while((!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A))&&c<KEYA_SPEED1){ //循环判断长按，到时跳转
					c++;delay_ms(10); //长按判断的计时
				}
				if(c>=KEYA_SPEED1){ //长键处理
					//长按后执行的程序放到此处
					GPIO_WriteBit(LEDPORT,LED1,(BitAction)(1));//LED控制
					while(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A));
				}else{ //单击或双击处理
					for(b=0;b<KEYA_SPEED2;b++){//检测双击
						delay_ms(20);
						if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A)){
							a=1;
							//双击后执行的程序放到此处
							GPIO_WriteBit(LEDPORT,LED2,(BitAction)(1));//LED控制

							while(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A));
						}//if
					}//for
					if(a==0){ //判断单击
						//单击后执行的程序放到此处
						GPIO_WriteBit(LEDPORT,LED1|LED2,(BitAction)(0));//LED控制
						
					}//if
				}//else
				a=0;c=0; //参数清0
			}//判断到的确按下了按键（延时后）
		} //按键判断在此结束（延时前）

	}//while(1)
```

**具体分析：**

1. TOUCH_KEY_A在未按下时为1，因此当其为0时进入if分支，相关语句如下：

```c
if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A))
```

2. 然后延时20ms再判断是否按下，相关语句如下：

```c
	delay_ms(20); //延时去抖动
	if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A))
```

3. 先判断是否为长按，KEYA_SPEED1是设置好的长按的时间长度，相关语句如下：

```c
//循环判断长按，到时跳转
while((!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A))&&c<KEYA_SPEED1){ 
//这里要维持在循环里需要两个条件中任意一个：1.保持持续按下 2.c递增到超过KEYA_SPEED1
				c++;delay_ms(10); //长按判断的计时
				}
```

- 这里KEYA_SPEED1设置的是100，而每一次循环递增的时间单位是10ms，因此退出循环的时间是1s，即判断按键触摸超过1s的为长按

4. 若是长按，则退出循环的条件是c>=KEYA_SPEED1，即执行以下语句：

```c
//长键处理
if(c>=KEYA_SPEED1){ 
		//长按后执行的程序放到此处
		GPIO_WriteBit(LEDPORT,LED1,(BitAction)(1));//控制LED1亮
		while(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A));//等待按键放开
		}
```

5. 若跳出循环的条件不是c而是中途放开了，证明不是长按，执行下列语句：

```c
for(b=0;b<KEYA_SPEED2;b++){//检测双击
						delay_ms(20);
```

利用for循环**从第一次放开按键开始**计时，KEYA_SPEED2是设置好的双击判断时间，在此时间内若再有按下则为双击

- KEYA_SPEED2为10，单位是20ms，即双击的判断时间为0.2s

6. for循环计时，在KEYA_SPEED2时间内若再有按下则为双击，执行如下：

```c
if(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A)){
							a=1;//标志着已经判断为
							//双击后执行的程序放到此处
							GPIO_WriteBit(LEDPORT,LED2,(BitAction)(1));//LED控制

							while(!GPIO_ReadInputDataBit(TOUCH_KEYPORT,TOUCH_KEY_A));
						}
```

7. for循环结束后，若a仍为0而不是1，证明在for循环时间内没有再次检测到按下按键，即判断为单击，执行以下语句：

```c
if(a==0){ //判断单击
						//单击后执行的程序放到此处
						GPIO_WriteBit(LEDPORT,LED1|LED2,(BitAction)(0));//LED控制
						
					}
```

