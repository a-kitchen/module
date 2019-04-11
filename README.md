# 模块

## 数据帧

* 帧长度 - 4 ~ 128 字节
* 每帧数据的格式 - 固定的起始头 ( AK 的 ASCII 码 0x41 + 0x4B ) 、引导区 ( len 本帧数据长度 + sum 校验和，
校验和是本帧所有其他数据字节和的补码 )、数据区 ( 每个数据包含键值数据对 key + val )

‘A’ | ‘K’
-- | --
len | sum
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
* 0x30 - 屏蔽错误码，时间 4 秒，如果要连续屏蔽，则重复发送
```c
  0x30, 0x80, // errcode 0x80 blocked for 4 sec.
```
* 0x68 - 锅底温度低字节，单位: 0.01 摄氏度
* 0x69 - 锅底温度高字节，单位: 0.01 摄氏度
```c
  0x68, 0xa8, 0x69, 0x16, // temperature 58 degrees centigrade (0x16a8)
```

## 上行数据键

* 0x2d - 模式
* 0x4e - 数据帧格式
* 0x4f - 设备信息
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
* 32-63 - 应用模式
* 65-73 - 手动挡位 1-9
* 128 - 升级固件
* 132 - 蓝牙配对模式
* 192-210 - 菜谱模式。其中：197 - 暂停；201 - 等待确认。重新设置模式为 193，然后 192 恢复或确认
* 254 - 关机

![模式](https://raw.githubusercontent.com/a-kitchen/module/master/modes.png)

正常情况下，上电会将模式初始化成 254 关机。但全新模块没有经过温度校准，模式将会初始化成 132 蓝牙配对。

## 数据帧格式 (Key = 0x4e)

数据帧格式是通过发送一个包含特殊数据区的数据帧实现，其中起始头 'AK' 和引导区 len + sum 与常规数据帧相同，
数据区由数据键值对 0x4e, 0x01 打头，按顺序紧跟需要的常规数据帧键。

例如，假设需要模块发出数据帧：

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

需要进行如下初始化：

```c
  'A', 'K', <len>, <sum>, 0x4e, 0x01, /**/0x24, 0x26, 0x22, 0x23, 0x68, 0x69, // init data frame
```

## 设备信息 (Key = 0x4f)

设备信息是通过发送一个包含特殊数据区的数据帧实现，其中起始头 'AK' 和引导区 len + sum 与常规数据帧相同，
数据区由数据键值对 0x4f, 0x01 打头，首先是 2 字节的 hardware revision 值，之后按顺序紧跟前置长度字节的
字符串段，前置长度字节为 0 时，本信息段维持之前的值。

例如，假设需要如下设备信息：

信息段 | 内容
-- | --
hardware string | 0xaa55
serial number | '1234567890'
model string | 'model'
manufacturer name | 'vander'
firmware string | 'firmware'
device name | 'name'

相应的初始化数据帧为：

```c
  0x4f, 0x01,
  'A', 'K', 
  <len>, <sum>,
  0x55, 0xaa,											// hardware string
  10, '1', '2', '3', '4', '5', '6', '7', '8', '9', '0',	// serial number
  5, 'm', 'o', 'd', 'e', 'l',							// model string
  6, 'v', 'a', 'n', 'd', 'e', 'r',						// manufacturer name
  8, 'f', 'i', 'r', 'm', 'w', 'a', 'r', 'e',			// firmware string
  4, 'n', 'a', 'm', 'e',								// name
```

## 错误码 (Key = 0x63)

* 0-63 - 无动作
* 64-127 - 仅显示
* 112-144 - 可屏蔽
* 128-191 - 停功率
* 192-255 - 停机

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

1. CAP1 - 触摸1
1. CAP2 - 触摸2
1. CAP3 - 触摸3
1. CAP4 - 触摸4
1. CAP5 - 触摸5
1. CAP6 - 触摸6
1. CAP7 - 触摸7
1. CAP8 - 触摸8
1. RES1
1. SWA - 开关A
1. SWB - 开关B
1. SW2 - 开关2
1. RES2
1. RES3
1. RES4
1. RES5
1. \__VAA - 模拟电源
1. \__GNDA - 模拟地
1. \__NTC - 测温热敏电阻
1. \__GNDD - 数字地
1. \__VDD - 数字电源
1. \__RX - 串口接收
1. \__TX - 串口发送
1. INT
1. SCL
1. SDA
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
1. CAPC - 触摸公共端

![针脚](https://raw.githubusercontent.com/a-kitchen/module/master/layout.png)

## 代码例子

```c
#include <clock.h>
#include <event.h>
#include <knob.h>
#include <light.h>
#include <log.h>
#include <project.h>

#define KEY_LED         0x27
#define KEY_MODE        0x2d
#define KEY_DATAFRAME   0x4e
#define KEY_ERRMASK     0x62
#define KEY_ERRCODE     0x63
#define KEY_TEMP_LO     0x68
#define KEY_TEMP_HI     0x69

#define LED_BOOTING     48
#define LED_OFF         64
#define LED_ON          128
#define LED_PAIRING     32
#define LED_RUNING      226
#define LED_WAITING     217

static U08 cled = LED_OFF;                          // 本地显示
static U08 cmod = AKMODE_OFF;                       // 本地模式
static U08 dkey;                                    // 下行数据帧键
static U08 dlen;                                    // 下行数据帧长度
static U08 rmod = AKMODE_OFF;                       // 远程模式
static U08 frame[] = {                              // 上行数据帧
  'A', 'K', 0, 0,   // header
  KEY_MODE,    0,   // 4,  5
  KEY_ERRCODE, 0,   // 6,  7
  KEY_TEMP_HI, 0,   // 8,  9
  KEY_TEMP_LO, 0,   // 10, 11
};                  // 12

U08 AkFw_GetParam(U08 key) {                        // 系统编译钩子
  return key == KEY_LED ? cled : 0;
}

void AkFw_SetParam(U08 key, U08 value) {            // 系统编译钩子
  if(key == KEY_LED)
    cled = value;
}

static void init(void) {
  static U08 ini_frame[] = {
    'A', 'K', 0, 0, KEY_DATAFRAME, 1,               // 下行数据帧初始化命令
    KEY_MODE, KEY_ERRCODE,                          // 下行数据帧格式
  };
  if(dkey != KEY_MODE || dlen != 8) {               // 检查下行数据帧格式
    // 初始化数据帧格式
    U08 sum;
    sum = 0;
    ini_frame[2] = sizeof ini_frame;
    ini_frame[3] = sum;
    for(U08 i = 0; i < sizeof ini_frame; i++)
      sum -= ini_frame[i];
    ini_frame[3] = sum;
    SCB_SpiUartPutArray(ini_frame, sizeof ini_frame);
  }
}

static void stream(U08 value) {                     // 解析下行数据
  static U08 a, k, n;

  a += value;
  n--;
  switch (k) {
  case 0:                                           // 检查帧头
    if (value == 'A')
      k = 1;
    return;
  case 1:                                           // 检查帧头
    if(value == 'K')
      k = 2;
    else k = value == 'A'? 1 : 0;
    return;
  case 2:
    if((value & 1) || (value < 4))
      k = value == 'A'? 1 : 0;
    else {
      a = 'A' + 'K' + value;
      k = 3;
      n = value - 2;                                // 提取帧长度
      dlen = value;
    }
    return;
  case 3:                                           // 越过校验和
    k = 4;
    return;
  case 4:
    if(n){
      k = value;                                    // 提取数据键
      return;
    }
    break;
  default:
    if(n){                                          // 未到达帧尾
      if(n == 3)                                    // 在确定位置
        dkey = k;                                   //   提取下行数据帧标志
      if(k == KEY_MODE) {                           // 提取数据值
        rmod = value;
        if (rmod == AKMODE_PREP)
          cmod = AKMODE_RUN;
        else if (cmod && cmod == rmod)
          cmod = 0;                                 // 本地数据一致化
      }
      k = 4;
      return;
    }
  }
  if (!(a - value))                                 // 校验和正确
    Clock_Light(3);
  k = value == 'A' ? 1 : 0;
}

int main(void) {
  U08 mod, sum, vled = 0, vmod = 0;
  U16 tmpr;

  CyGlobalIntEnable;
  Log_Start("\n\n\rPanel\n\r(c) a.kitchen\n\r");
  Clock_Start();
  SCB_Start();
  Knob_Start();
  Light_Start();

  Clock_OnBootend();
  init();

  for(;;) {
    mod = cmod ? cmod : rmod;
    switch(Event_Get()){
    case EVENT_BEAT:                                // 心跳，16 次/秒
      Clock_OnBeat();
      Light_OnBeat();
      tmpr = (Clock_millisecond / 1000) & 0x3fff;   // 模拟温度变化，0.01/s

      frame[2] = sizeof frame;                      // 构建上行数据帧
      frame[5] = cmod;
      frame[7] = 0;
      frame[9] = tmpr >> 8;
      frame[11] = tmpr;
      sum = 0;
      frame[3] = sum;
      for(U08 i = 0; i < sizeof frame; i++)
        sum -= frame[i];
      frame[3] = sum;                               // 校验和
      SCB_SpiUartPutArray(frame, sizeof frame);     // 发送上行数据帧

      break;
    case EVENT_CLOCK:                               // 时钟，1 次/秒
      init();                                       // 初始化数据帧格式
      break;
    case EVENT_IDLE:                                // 空闲
      Knob_OnIdle();
      break;
    case EVENT_LPRESS:                              // 长按
      if(Knob_key == KNOB_POWER)                    // 电源键开关机
        cmod = mod == AKMODE_OFF ? AKMODE_L_0 : AKMODE_OFF;
      break;
    case EVENT_RELEASE:                             // 释放开关
      if(mod == AKMODE_OFF)
        break;
	  switch(Knob_key) {
      case KNOB_MINUS:                              // 左转按钮
        if (mod > AKMODE_L_1)
          cmod = mod - 1;
        else if (mod == AKMODE_L_1 || mod == AKMODE_PAIRING)
          cmod = AKMODE_L_0;
        break;
      case KNOB_PLUS:                               // 右转按钮
        if (mod == AKMODE_L_0 || mod == AKMODE_PAIRING)
          cmod = AKMODE_L_1;
        else if (mod < AKMODE_L_9)
          cmod = mod + 1;
        break;
      case KNOB_POWER:                              // 电源键
        if (mod == AKMODE_L_0 || mod == AKMODE_PAIRING)
          cmod = AKMODE_L_5;
        else if (mod >= AKMODE_L_1 || mod <= AKMODE_L_9)
          cmod = AKMODE_L_0;
        else if (mod == AKMODE_WAITNEXT || mod == AKMODE_PAUSE)
          cmod = AKMODE_PREP;                       // 下一步
        else if (mod > AKMODE_PREP && mod != AKMODE_OFF)
          cmod = AKMODE_PAUSE;                      // 暂停
        break;
      }
      break;
    case EVENT_XPRESS:                              // 超长按
      if(Knob_key == KNOB_POWER)                    //   电源键配对
        cmod = AKMODE_PAIRING;
      break;
    }
	if(cmod)
      mod = cmod;
    if(vmod != mod) {
      vmod = mod;
      // 设置指示灯
      switch (mod) {
      case AKMODE_BOOTING:
        cled = LED_BOOTING;
        break;
      case AKMODE_L_0:
        cled = LED_ON;
        break;
      case AKMODE_OFF:
      case AKMODE_PREP:
        cled = LED_OFF;
        break;
      case AKMODE_PAIRING:
        cled = LED_PAIRING;
        break;
      case AKMODE_PAUSE:
      case AKMODE_WAITNEXT:
        cled = LED_WAITING;
        break;
      default:
        cled = vmod > AKMODE_L_9 ? LED_RUNING : LED_ON - AKMODE_L_1 + 1 + mod;
      }
    }
    if(vled != cled) {
      vled = cled;
      Light_OnParam(KEY_LED);                       // 设置指示灯
    }
    if(SCB_SpiUartGetRxBufferSize())                // 获取下行数据
      stream(SCB_SpiUartReadRxData());              // 解析下行数据
  }
}
```
