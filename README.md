# 模块

## 数据帧

* 帧长度 - 4 ~ 256 字节
* 每帧数据的格式 - 固定的起始头 ( AK 的 ASCII 码 0x41 + 0x4B ) 、引导区 ( Len 本帧数据长度 + Sum 校验和，
校验和是本帧所有其他数据字节和的补码 )、数据区 ( 每个数据包含键值数据对 Key + Val )

‘A’ | ‘K’
------------ | -------------
Len | Sum
Key1 | Val1
Key2 | Val2
... | ...

上行和下行数据帧格式相同

## 下行数据键

* 0x22 - 功率低字节，单位: 0.1 瓦
* 0x23 - 功率高字节，单位: 0.1 瓦
```c
  0x22, 0x10, 0x23, 0x27, // power 1000w (0x2710)
```
* 0x24 - 外设控制低字节
* 0x25 - 外设控制高字节
```c
  0x24, 3, 0x25, 0,       // fan speed 3
```
* 0x28 - 定时器低字节，单位: 秒
* 0x29 - 定时器高字节，单位: 秒
```c
  0x28, 0xa8, 0x29, 0x16, // timer 5800 sec (0x16a8)
```
* 0x2d - 模式
* 0x68 - 锅底温度低字节，单位: 0.01 摄氏度
* 0x69 - 锅底温度高字节，单位: 0.01 摄氏度
```c
  0x68, 0xa8, 0x69, 0x16, // temperature 58 degrees centigrade (0x16a8)
```

数据帧例子

```c
uint8 dataframe [] = {
  'A', 'K', 20, 0,        // header, len = 20
  0x2d, 0,                // mode
  0x26, 0,
  0x22, 0, 0x23, 0,       // power
  0x24, 0, 0x25, 0,       // peripheral
  0x68, 0, 0x69, 0,       // temperature
};
```

## 上行数据键

* 0x2d - 模式
* 0x62 - 错误掩码
* 0x63 - 错误代码
* 0x64 - 电源电压低字节，单位: 0.01 伏
* 0x65 - 电源电压高字节，单位: 0.01 伏
 ```c
  0x64, 0xf0, 0x65, 0x55, // voltage 220v (0x55f0)
```
* 0x66 - 电源电流低字节，单位: 毫安
* 0x67 - 电源电流高字节，单位: 毫安
```c
0x66, 0x10, 0x67, 0x27, // current 10a (0x2710)
```
* 0x68 - 锅底温度低字节，单位: 0.01 摄氏度
* 0x69 - 锅底温度高字节，单位: 0.01 摄氏度
* 0x6a - 负载测试/负载类型
* 0x6c - 散热器温度低字节，单位: 0.01 摄氏度
* 0x6d - 散热器温度高字节，单位: 0.01 摄氏度
* 0x6e - 时钟低字节
* 0x6f - 时钟高字节

## 模式 (Key = 0x2d)

* 0 - 无模式
* 2 - 待机，手动模式
* 64-95 - 应用模式
* 128 - 升级固件
* 132 - 蓝牙配对模式
* 192-210 - 菜谱模式。其中：197 - 暂停；201 - 等待确认。重新设置模式为 193，然后 192 恢复或确认
* 254 - 关机

![模式](https://raw.githubusercontent.com/a-kitchen/module/master/modes.png)

正常情况下，上电会将模式初始化成 254 关机。但全新模块没有经过温度校准，模式将会初始化成 132 蓝牙配对。

## 错误码 (Key = 0x63)

* 4 - 传感器故障
* 5 - 传感器开路
* 6 - 传感器短路
* 7 - 传感器引脚故障
* 8 - IGBT 过热
* 11 - 有线通信故障
* 12 - 无线通信故障
* 16 - 电源故障
* 17 - 过流
* 18 - 过压
* 20 - 无负债
* 22 - 干锅
* 26 - 无锅盖
* 27 - 无锅铲
* 28 - 无真空
* 65 - 命令错误

## 外设控制掩码 (Key = 0x24, 0x25)

* 0x0000 - 风扇停，翻炒/搅拌停
* 0x0001 - 风扇 1
* 0x0002 - 风扇 2
* 0x0003 - 风扇 3
* 0x0100 - 翻炒/搅拌 1
* 0x0200 - 翻炒/搅拌 2
* 0x0300 - 翻炒/搅拌 3
* 0x0400 - 翻炒/搅拌 4
* 0x0500 - 翻炒/搅拌 5
* 0x0600 - 翻炒/搅拌 6
* 0x0700 - 翻炒/搅拌 7

## 端子

1. CAPC - 触摸公共端
1. CAP1 - 触摸1
1. CAP2 - 触摸2
1. CAP3 - 触摸3
1. CAP4 - 触摸4
1. CAP5 - 触摸5
1. CAP6 - 触摸6
1. CAP7 - 触摸7
1. CAP8 - 触摸8
1. SWA - 编码器A
1. SWB - 编码器B
1. SW2 - 开关
1. RSV1
1. RSV2
1. RSV3
1. RSV4
1. \__VAA - 模拟电源
1. \__GNDA - 模拟地
1. \__NTC - 测温热敏电阻
1. \__GNDD - 数字地
1. \__VDD - 数字电源
1. \__RX - 串口接收
1. \__TX - 串口发送
1. SCL
1. SDA
1. INT
1. COM - 显示背景亮度
1. LED1 - 显示1
1. LED2 - 显示2
1. LED3 - 显示3
1. LED4 - 显示4
1. LED5 - 显示5
1. LED6 - 显示6
1. LED7 - 显示7
1. LED8 - 显示8
1. LED9 - 显示9
1. LED10 - 显示10
1. LED11 - 显示11
1. LED12 - 显示12

![针脚](https://raw.githubusercontent.com/a-kitchen/module/master/layout.png)

## 代码例子

```c
#include "clock.h"
#include "event.h"
#include "knob.h"
#include "light.h"

#define KEY_LED         0x27
#define KEY_MODE        0x2d
#define KEY_RESERVED    0x62
#define KEY_ERROR       0x63

#define LED_PAIRING     32
#define LED_READY       128
#define LED_RUNING      226
#define LED_STANDBY     0
#define LED_WAITING     217

#define MODE_CONT       193
#define MODE_DUMMY      0
#define MODE_PAIRING    132
#define MODE_READY      2
#define MODE_RUN        192
#define MODE_STANDBY    254
#define MODE_UPGRADE    128
#define MODE_WAITNEXT   201

static uint8 led = LED_STANDBY;
static uint8 cmode = MODE_STANDBY;  // 本地模式
static uint8 rmode;                 // 远程模式
static uint8 dataframe[] = {        // 上行数据帧
  'A', 'K', 0, 0,
  KEY_MODE, 0,      // 4
  KEY_LED, 0,       // 6
  KEY_RESERVED, 0,  // 8
  KEY_ERROR, 0,     // 10
};

int8 stream(uint8 value) {          // 解析下行数据
  static uint8 a, k, n;

  a += value;
  n--;
  switch (k) {
  case 0:
    if (value == 'A')
      k = 1;
    return 0;
  case 1:
    if(value == 'K')
      k = 2;
    else k = value == 'A'? 1 : 0;
    return k == 1 ? 1 : 0;
  case 2:
    if((value & 1) || (value < 4))
      k = value == 'A'? 1 : 0;
    else {
      a = 'A' + 'K' + value;
      k = 3;
      n = value - 2;
    }
    return k == 1 ? 1: 0;
  case 3:
    k = 4;
    Light_off();
    return 0;
  case 4:
    if(n){
      k = value;
      return 0;
    }
    break;
  default:
    if(n){
      switch (k){
      case KEY_MODE:
        rmode = value;
        if (cmode && cmode == rmode)
          cmode = MODE_DUMMY;
        else if (rmode == MODE_CONT)
          cmode = MODE_RUN;				
        break;
      }
      k = 4;
      return 0;
    }
  }
  a -= value;
  if (!a)
    Light_on();
  k = value == 'A' ? 1 : 0;
  return !a;
}

int main(void) {
  uint8 e = 0, v = 5;
  int m = MODE_STANDBY;

  CyGlobalIntEnable;

  Clock_start();
  SCB_Start();
  Light_start();
  Knob_start();

  for(;;) {
    switch (Event_get()){
    case EVENT_LPRESS:      // 长按开关机
      cmode = rmode == MODE_STANDBY ? MODE_READY : MODE_STANDBY;
      e = 0;
      break;
    case EVENT_XPRESS:      // 超长按配对
      led = LED_PAIRING;
      cmode = MODE_PAIRING;
      break;
    case EVENT_RELEASE:     // 释放开关
      if (rmode == MODE_PAIRING)
        cmode = MODE_READY;
      if (rmode == MODE_READY || rmode == MODE_PAIRING) {
        e = e ? 0 : v;
        led = LED_READY | e;
      } else if (rmode > MODE_CONT)	// 下一步
        cmode = MODE_CONT;
      break;
    case EVENT_TICK:        // 滴答时钟， 20 次/秒
      Clock_tick();
      Light_tick();
      dataframe[2] = sizeof dataframe;
      dataframe[5] = cmode;
      dataframe[7] = led;
      dataframe[9] = 0;
      dataframe[11] = 0;
      dataframe[3] = - dataframe[5] - dataframe[7] - dataframe[9] - dataframe[11] -
      	('A' + 'K' + sizeof dataframe + KEY_MODE + KEY_LED + KEY_RESERVED + KEY_ERROR);
      SCB_SpiUartPutArray(dataframe, sizeof dataframe);
      break;
    case EVENT_TURN:        // 旋转按钮
      if (rmode == MODE_PAIRING)
        cmode = MODE_READY;
      if (rmode == MODE_READY || rmode == MODE_PAIRING) {
        int8 d = Knob_delta();
        e += d;
        if (e > 127)
          e = 0;
        else if (e > 9)
          e = 9;
        if (e)
          v = e;
        led = LED_READY | e;
      }
      break;
    }
    if (m != rmode) {
      m = rmode;
      // 设置指示灯
      switch (m) {
      case MODE_DUMMY:
        break;
      case MODE_PAIRING:
        led = LED_PAIRING;
        break;
      case MODE_STANDBY:
        led = LED_STANDBY;
        break;
      case MODE_WAITNEXT:
        led = LED_WAITING;
        break;
      default:
        led = m < MODE_UPGRADE ? LED_READY | e : LED_RUNING;
      }
    }
    static int d;
    if (d != led) {
      Light_indicate(led);
      d = led;
    }
    Knob_none();
    while(SCB_SpiUartGetRxBufferSize())
      stream(SCB_SpiUartReadRxData());  // 读取下行数据
  }
}
```
