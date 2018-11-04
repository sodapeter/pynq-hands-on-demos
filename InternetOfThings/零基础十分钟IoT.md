

<img src="xilinx_logo.png" style="width:400px" align=right/>

<h1 align=center>零基础十分钟开发自己的IoT</h1>

<img src="pynq_logo.png" style="width:200px" align=right/>

规则：十分钟之内根据老师的指导，在PYNQ-Z2开发板上独立完成IoT应用的开发或者算法的加速。

| 活动时间（报名制，准点开课） |
| :--------------------------: |
|         10:30-10:45          |
|         11:30-11:45          |
|         14:30-14:45          |
|         16:30-16:45          |

在本教程的带领下，你将在十分钟内学会在PYNQ-Z2板卡上使用传感器和LED开发自己的IoT应用！

## 初始化板卡


```python
from pynq.overlays.base import BaseOverlay
base = BaseOverlay("base.bit")
```

## 初始化LED Bar
Grove LED bar上有10个LED灯，8个绿色的LED，一个橘黄色的LED和一个红色的LED。每个LED的亮度都可以独立被控制。我们将会在Z2板卡上外接一块Arduino Shield扩展板，LED bar连接在扩展板的`D6`接口上。

<img src="1.png" style="width:600px"/>


```python
from pynq.lib.arduino import Grove_LEDbar
from pynq.lib.arduino import ARDUINO_GROVE_G4

# Instantiate Grove LED Bar on Arduino shield G4
ledbar = Grove_LEDbar(base.ARDUINO, ARDUINO_GROVE_G4)
ledbar.reset()
```

## 初始化超声波传感器
Pmod-grove扩展板连接在`PMODA`接口上，超声波传感器连接Pmod-Grove转接器的`G1`接口上的

<img src="2.png" style="width:600px"/>

```python
%%microblaze base.PMODA

#include "xparameters.h"
#include "xtmrctr.h"
#include "gpio.h"
#include "timer.h"
#include <pmod_grove.h>

#define TCSR0 0x00
#define TLR0 0x04
#define TCR0 0x08
#define TCSR1 0x10
#define TLR1 0x14
#define TCR1 0x18
#define MAX_COUNT 0xFFFFFFFF

void create_10us_pulse(gpio usranger){
    gpio_set_direction(usranger, GPIO_OUT);
    gpio_write(usranger, 0);
    delay_us(2);
    gpio_write(usranger, 1);
    delay_us(10);
    gpio_write(usranger, 0);
}

void configure_as_input(gpio usranger){
    gpio_set_direction(usranger, GPIO_IN);
}

unsigned int capture_duration(gpio usranger){
    unsigned int count1, count2;
    count1=0;
    count2=0;
    XTmrCtr_WriteReg(XPAR_TMRCTR_0_BASEADDR, 0, TLR0, 0x0);
    XTmrCtr_WriteReg(XPAR_TMRCTR_0_BASEADDR, 0, TCSR0, 0x190);
    while(!gpio_read(usranger));
    count1=XTmrCtr_ReadReg(XPAR_TMRCTR_0_BASEADDR, 0, TCR0);
    while(gpio_read(usranger));
    count2=XTmrCtr_ReadReg(XPAR_TMRCTR_0_BASEADDR, 0, TCR0);
    if(count2 > count1) {
        return (count2 - count1);
    } else {
        return((MAX_COUNT - count1) + count2);  
    }
}

unsigned int read_raw(){
    gpio usranger;
    usranger = gpio_open(PMOD_G1_A);
    create_10us_pulse(usranger);
    configure_as_input(usranger);
    return capture_duration(usranger);
}
```

## 使用python即可调用超声波传感器


```python
from pynq import Clocks

def read_distance_cm():
    raw_value = read_raw()
    clk_period_ns = int(1000 / Clocks.fclk0_mhz)
    num_microseconds = raw_value * clk_period_ns * 0.001
    if num_microseconds * 0.001 > 30:
        return 500
    else:
        return num_microseconds/58
```

## 案例展示：超声波测距仪

## 靠近测距仪50cm以内，LED每近5cm多亮一格

按BTN3结束


```python
import time
import math

btns = [base.buttons[index] for index in range(4)] 
ledbar.reset()

done = False
while not done:
    if (btns[3].read()==1):
        ledbar.reset()
        done = True
        
    distance = read_distance_cm()
    if distance < 50.0:
        ledbar.write_level(10-math.floor(distance/5), 3, 1)
    else:
        ledbar.write_level(0, 3, 1)
    time.sleep(0.1)
```

