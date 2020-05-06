# Module

## Example

Integrated control

![schematic](https://raw.githubusercontent.com/a-kitchen/module/master/integrated.png)

Pass through

![schematic](https://raw.githubusercontent.com/a-kitchen/module/master/passthrough.png)

## Data frame

* Frame length - 4 ~ 128 byte
* Frame format - header ( ASCII of 'AK' 0x41 + 0x4B ) 、leading section ( len: length of this frame + sum: checksum, 校验和是本帧所有其他数据字节和的补码checksum is complement code of  all bytes'   )、data section ( key + val )

‘A’ | ‘K’
-- | --
len | sum
Key1 | Val1
Key2 | Val2
... | ...

```c
  // default downstream data frame 
  0x41, 0x4b,   // data header 'AK'
  0x10, 0x1f,   // length = 16, checksum = -(41+4b+10+23+27+22+10+24+26+09+89+88+65)
  0x23, 0x27,
  0x22, 0x10,   // power = 10000 (1000w)
  0x24, 0,      // fan speed: 0
  0x26, 9,      // loadtest = 9
  0x89, 0,
  0x88, 0x65,   // hardware_id = 101
```

Upstream data has the same format as downstream data.

## Key of downstream data

* 0x22 - lower byte of power, unit: 0.1 W
* 0x23 - higher byte of power, unit: 0.1 W
```c
  0x22, 0x10, 0x23, 0x27, // power 1000w (0x2710)
```
* 0x24 - peripheral control
* 0x25 - motor control
```c
  0x24, 3, 0x25, 0,       // fan speed 3
```
* 0x28 - lower byte of timer, unit: second
* 0x29 - higher byte of timer, unit: second
```c
  0x28, 0xa8, 0x29, 0x16, // timer 5800 sec (0x16a8)
```
* 0x2d - mode
* 0x30 - error code mask, last for 4 second, repeat if need longer than 4 second
```c
  0x30, 0x80, // errcode 0x80 blocked for 4 sec.
```
* 0x68 - lower byte of container's temperature, unit: 0.01 ℃
* 0x69 - higher byte of container's temperature, unit: 0.01 ℃
```c
  0x68, 0xa8, 0x69, 0x16, // temperature 58 degrees centigrade (0x16a8)
```
* 0x88 - lower byte of hardware number
* 0x89 - higher byte of hardware number
* 0x8a - lower byte of flags
* 0x8b - higher byte of flags

## Key of upstream data

* 0x2d - mode
* 0x4e - format of data frame
* 0x4f - device information
* 0x54 - level information
* 0x63 - error code
* 0x64 - lower byte of power voltage, unit: 0.01 V
* 0x65 - higher byte of power voltage, unit: 0.01 V
 ```c
  0x64, 0xf0, 0x65, 0x55, // voltage 220v (0x55f0)
```
* 0x66 - lower byte of current, unit: mA
* 0x67 - higher byte of current, unit: mA
```c
  0x66, 0x10, 0x67, 0x27, // current 10a (0x2710)
```
* 0x68 - lower byte of container's temperature, unit: 0.01 ℃
* 0x69 - higher byte of container's temperature, unit: 0.01 ℃
* 0x6a - load test/load type
* 0x6c - lower byte of sink's temperature, unit: 0.01 ℃
* 0x6d - higher byte of sink's temperature, unit: 0.01 ℃
* 0x6e - lower byte of clock
* 0x6f - higher byte of clock

## Peripheral control (Key = 0x24)

* 0x0000 - fan stops
* 0x0001 - fan level 1
* 0x0002 - fan level 2
* 0x0003 - fan level 3

## Motor control (Key = 0x25)

* 0x00 - motor stops
* 0x01 - motor level 1
* 0x02 - motor level 2
* 0x03 - motor level 3
* 0x04 - motor level 4
* 0x05 - motor level 5
* 0x06 - motor level 6
* 0x07 - motor level 7
* 0x08 - motor level 8
* 0x09 - motor level 9
* 0x0a - motor level 10

## Mode (Key = 0x2d)

* 0 - none
* 1-15 - preset program
* 65-73 - manual  1-9
* 128 - OTA
* 132 - bluetooth pairing
* 192-210 - program running. 197 - pause; 201 - wait next. Resume - 193, then 192.
* 254 - power off

![mode](https://raw.githubusercontent.com/a-kitchen/module/master/modes.png)

## Format of data frame (Key = 0x4e)

The format of data initially is:

```c
  'A', 'K', 16, sum,
  0x23, power_hi,
  0x22, power_lo,
  0x24, fan_speed,
  0x26, loadtest,
  0x89, hardware_id_hi,
  0x88, hardware_id_lo,
```
You can change the format by sending a data frame which includes a special data section. The frame starts with header 'AK' and leading section len + sum, data section starts with 0x4e, 0x01, followed by data keys.

For example:

```c
  'A', 'K', len, sum,     // header
  0x2d, 0,                // mode
  0x26, 0,                // load
  0x22, 0, 0x23, 0,       // power
  0x24, 0, 0x25, 0,       // peripheral
  0x68, 0, 0x69, 0,       // temperature
  0x88, 0, 0x89, 0,       // hardware
  0x30, 0,                // block error
```
It needs to be initiated by :

```c
  'A', 'K', len, sum, 
  0x4e, 0x01,                                                       // key: frame format
  0x2d, 0x26, 0x22, 0x23, 0x24, 0x25, 0x68, 0x69, 0x88, 0x89, 0x30, // init data frame
```

## Device information (Key = 0x4f)

Device information is set by sending a data frame which includes a special data section. The frame starts with header 'AK' and leading section len + sum, data section starts with 0x4f, 0x01，then will have 2 byte of hardware number，之后按顺序紧跟前置长度字节的字符串段，前置长度字节为 0 时，本信息段维持之前的值。

For example, if you want device information as below：

信息段 | 内容
-- | --
hardware number | 0xaa55
serial number | '1234567890'
model string | 'model'
manufacturer name | 'vander'
firmware string | 'firmware'
device name | 'name'

Initialization data frame should be:

```c
  'A', 'K', <len>, <sum>,
  0x4f, 0x01,                                           // key: device information
  0x55, 0xaa,                                           // hardware number
  0,
  6, 'v', 'a', 'n', 'd', 'e', 'r',                      // manufacturer name
  5, 'm', 'o', 'd', 'e', 'l',                           // model string
  10, '1', '2', '3', '4', '5', '6', '7', '8', '9', '0',	// serial number
  0,
  5, 'f', 'w', 'a', 'r', 'e',                           // firmware string
  0,
  4, 'n', 'a', 'm', 'e',                                // name
  0,                                                    // levels
  0,                                                    // service data
```

If initialized successfully，"hardware number (key = 0x88, 0x89)" should turn to 0xaa55 which is the value in the example.

## Error code (Key = 0x63)

* 0-63 - 无动作
* 64-127 - 仅显示
* 112-144 - 可屏蔽
* 128-191 - 停功率
* 192-255 - 停机

## Flags (Key = 0x8a, 0x8b)

* 0x8000 - 蓝牙连接


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

## Code example

```c
#include <clock.h>
#include <event.h>
#include <knob.h>
#include <light.h>
#include <loud.h>
#include <project.h>

#define	LEN	0
#define	SUM	0

#define HARDWARE		201

#define KEY_LED         0x27
#define KEY_MODE        0x2d
#define KEY_DATFRAME    0x4e
#define KEY_DEVINFO     0x4f
#define KEY_ERRCODE     0x63
#define KEY_TEMP_LO     0x68
#define KEY_TEMP_HI     0x69
#define KEY_HDWARE_LO   0x88
#define KEY_HDWARE_HI   0x89

#define LED_BOOTING     48
#define LED_OFF         64
#define LED_ON          128
#define LED_PAIRING     32
#define LED_RUNING      226
#define LED_WAITING     217

static U08 cled = LED_OFF;                          // 本地显示
static U08 cmod = AkMODE_OFF;                       // 本地模式
static U08 dkey;                                    // 下行数据帧键
static U08 dlen;                                    // 下行数据帧长度
static U08 rmod = AkMODE_OFF;                       // 远程模式
static U16 hdwr;                                    // 设备信息

U08 Ak_GetParam08(U08 key) {                        // 系统编译钩子
  return key == KEY_LED ? cled : 0;
}

void Ak_SetParam08(U08 key, U08 value) {            // 系统编译钩子
  if(key == KEY_LED)
    cled = value;
}

static void send(U08*, U08);

static void beat(void) {
  if(dkey != KEY_HDWARE_HI || dlen != 12) {         // 检查下行数据帧格式
    static U08 ini[] = {
      'A', 'K', LEN, SUM, KEY_DATFRAME, 1,              // 下行数据帧初始化命令
      KEY_MODE, KEY_ERRCODE,                        // 下行数据帧格式
      KEY_HDWARE_HI, KEY_HDWARE_LO,
    };
    send(ini, sizeof ini);                          // 初始化数据帧格式
  } else if(hdwr != HARDWARE) {
/*    static U08 inf[] = {
      'A', 'K', LEN, SUM, 0x4f, 1,
      0x0a, 0x00,                           // hardware = 0x000a
      0,
      6, 'v', 'a', 'n', 'd', 'e', 'r',      // vander
      5, 'm', 'o', 'd', 'e', 'l',           // model
      6, 's', 'e', 'r', 'i', 'a', 'l',      // serial
      0,
      4, 'f', 'i', 'r', 'm',                // firmware
      0,
      5, 'p', 'a', 'n', 'e', 'l',           // name
      0, 0,                                 // levels, heating
      18,                                   // scan response
      0x10, 0x11,                           //   sign
      0x00, 0x54,                           //   product id
      0x00,                                 //   counter
      '%', '%', '%', '%', '%', '%',         //   mac address
      0x00,                                 //   capability
      0x00, 0x00,                           //   wifi
      0x00, 0x00,                           //   io capability
      0x00, 0x00,                           //   reserved
    };// */
    static U08 inf[] = {
      'A', 'K', LEN, SUM, 0x4f, 1,
      (U08)HARDWARE, (U08)(HARDWARE >> 8),  // hardware
      0,
      7, 'J', 'o', 'y', 'o', 'u', 'n', 'g', // vander
      2, 'A', '8',                          // model
      3, 's', '/', 'n',                     // serial
      0,
      2, '2', '1',                          // firmware
      0,
      7, 'J', 'Y', '_', '4', '0', '0', '0', // name
      0,                                    // levels
      8, 0x80, 0x3e, 0x70, 0x17, 0x14, 0x50, 0x00, 0x00,
                                            // heating: 16000, 6000, 20500
      18,                                   // scan response
      0x10, 0x11,                           //   sign
      0x00, 0x54,                           //   product id
      0x00,                                 //   counter
      '%', '%', '%', '%', '%', '%',         //   mac address
      0x00,                                 //   capability
      0x00, 0x00,                           //   wifi
      0x00, 0x00,                           //   io capability
      0x00, 0x00,                           //   reserved
    };// */
    send(inf, sizeof inf);                          // 初始化设备信息
  } else {
    static U08 dat[] = {                            // 上行数据帧
      'A', 'K', LEN, SUM,                   // header
      KEY_MODE,    0,                       // 4,  5
      KEY_ERRCODE, 0,                       // 6,  7
      KEY_TEMP_HI, 0,                       // 8,  9
      KEY_TEMP_LO, 0,                       // 10, 11
    };                                      // 12
    U16 tmp = (Clock_millisecond / 1000) & 0x3fff;  // 模拟温度变化，0.01/s
    dat[5] = cmod;                                  // 构建上行数据帧
    dat[7] = 0;
    dat[9] = tmp >> 8;
    dat[11] = tmp;
    send(dat, sizeof dat);
  }
}

static void send(U08 *bits, U08 len) {
  U08 sum;
  sum = 0;
  bits[2] = len;
  bits[3] = sum;
  for(U08 i = 0; i < len; i++)
    sum -= bits[i];
  bits[3] = sum;
  SCB_SpiUartPutArray(bits, len);
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
    if(value < 4)
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
      switch(k) {
      case KEY_MODE:                                // 提取远程模式
        rmod = value;
        //if (cmod == AkMODE_PREP && rmod == AkMODE_PREP)
        if (rmod == AkMODE_PREP)
          cmod = AkMODE_RUN;
        else if (cmod && (cmod == rmod || (cmod == AkMODE_RETURN && rmod >= AkMODE_APPMIN) ||
            (rmod > AkMODE_RUN && rmod < AkMODE_BOOTING)))
          cmod = 0;                                 // 本地数据一致化
        break;
      case KEY_HDWARE_LO:
        hdwr = value | (hdwr & 0xff00);
        break;
      case KEY_HDWARE_HI:
        hdwr = (value << 8) | (hdwr & 0x00ff);
        break;
      }
      k = 4;
      return;
    }
  }
  if (!(a - value))                                 // 如果：校验和正确
    Clock_Light(3);                                 // 亮灯，表示接收成功
  k = value == 'A' ? 1 : 0;
}

int main(void) {
  U08 mod, vled = 0, vmod = 0;

  CyGlobalIntEnable;
  Log_Start("\n\n\rPanel\n\r(c) a.kitchen\n\r");
  Clock_Start();
  SCB_Start();
  Knob_Start();
  Light_Start();

  Clock_OnBootend();

  for(;;) {
    mod = cmod ? cmod : rmod;
    switch(Event_Get()){
    case EVENT_BEAT:                                // 心跳，16 次/秒
      Clock_OnBeat();
      Light_OnBeat();
      beat();
      break;
    case EVENT_IDLE:                                // 空闲
      Knob_OnIdle();
      break;
    case EVENT_LPRESS:                              // 长按
      if(Knob_key == KNOB_POWER)                    // 电源键开关机
        cmod = mod == AkMODE_OFF ? AkMODE_L_0 : AkMODE_OFF;
      break;
    case EVENT_RELEASE:                             // 释放开关
      if(mod == AkMODE_OFF)
        break;
	  switch(Knob_key) {
      case KNOB_MINUS:                              // 左转按钮
        if (mod > AkMODE_L_1)
          cmod = mod - 1;
        else if (mod == AkMODE_L_1 || mod == AkMODE_PAIRING)
          cmod = AkMODE_L_0;
        break;
      case KNOB_PLUS:                               // 右转按钮
        if (mod == AkMODE_L_0 || mod == AkMODE_PAIRING)
          cmod = AkMODE_L_1;
        else if (mod < AkMODE_L_9)
          cmod = mod + 1;
        break;
      case KNOB_POWER:                              // 电源键
        if (mod <= AkMODE_READYMAX || mod == AkMODE_PAIRING)
          cmod = AkMODE_L_5;
        else if (mod < AkMODE_APPMIN)
          cmod = AkMODE_RETURN;                     // 恢复
        else if (mod <= AkMODE_APPMAX)
          cmod = AkMODE_READYMAX + 1;               // 暂停
        else if (mod >= AkMODE_L_1 || mod <= AkMODE_L_9)
          cmod = AkMODE_L_0;
        else if (mod == AkMODE_WAITNEXT || mod == AkMODE_PAUSE)
          cmod = AkMODE_PREP;                       // 下一步
        else if (mod > AkMODE_PREP && mod != AkMODE_OFF)
          cmod = AkMODE_PAUSE;                      // 暂停
        break;
      }
      break;
    case EVENT_XPRESS:                              // 超长按
      if(Knob_key == KNOB_POWER)                    //   电源键配对
        cmod = AkMODE_PAIRING;
      break;
    }
	if(cmod)
      mod = cmod;
    if(vmod != mod) {
      vmod = mod;
      // 设置指示灯
      switch (mod) {
      case AkMODE_BOOTING:
        cled = LED_BOOTING;
        break;
      case AkMODE_OFF:
      case AkMODE_PREP:
        cled = LED_OFF;
        break;
      case AkMODE_PAIRING:
        cled = LED_PAIRING;
        break;
      case AkMODE_PAUSE:
      case AkMODE_WAITNEXT:
        cled = LED_WAITING;
        break;
      default:
        if (vmod >= AkMODE_L_1 && vmod <= AkMODE_L_9)
          cled = LED_ON - AkMODE_L_1 + 1 + mod;
        else if (vmod <= AkMODE_READYMAX)
          cled = LED_ON;
        else if (vmod < AkMODE_APPMIN)
          cled = LED_WAITING;
        else cled = LED_RUNING;
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
