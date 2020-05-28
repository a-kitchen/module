# FAQ

**Q:** For safety issue, the device should start to operate when an action is taken by a device user after APP send operation order. Can this feature be realized using AK protocol? 

**A:** Our protocol have a solution for "user confirm by pressing/turning button then device operate" when the APP is taking control of the device. The products that we worked with other brandings already have the same feature.


**Q:** Is AK protocol a one-way protocol(AK module => other MCU)?

**A:** AK protocol is a two-way protocol(AK module <=> other MCU), in the document, upstream data means other MCU => AK module, downstream data means AK module => other MCU. Both upstream and downstream data share the same frame format but with different data keys.


# Module

## Example

Integrated control

![schematic](https://raw.githubusercontent.com/a-kitchen/module/master/integrated.png)

Passthrough

![schematic](https://raw.githubusercontent.com/a-kitchen/module/master/passthrough-new.png)

## Data port

* UART - baud rate 9600

## Data frame

* Frame length - 4 ~ 128 byte
* Frame speed - 8 ~ 24 frame/sec
* Frame format - header ( ASCII of 'AK' 0x41 + 0x4B ) 、leading section ( len: length of this frame + sum: checksum, checksum is complement code of all bytes)、data section ( key + val )

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
* 17-31 - reserve
* 33-47 - timing
* 49-63 - ready
* 128 - OTA
* 132 - bluetooth pairing
* 192-210 - program running. 197 - pause; 201 - wait next. Resume - 193, then 192.
* 254 - power off

![mode](https://raw.githubusercontent.com/a-kitchen/module/master/modes-EN.png)

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

Device information is set by sending a data frame which includes a special data section. The frame starts with header 'AK' and leading section len + sum, data section starts with 0x4f, 0x01, then follow 2 byte of hardware number, then one leading byte ( length ), followed by length bytes of character data.

For example, if you want device information as below：

info | value
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

In the example if initialized successfully，"hardware number (key = 0x88, 0x89)" should turn to 0xaa55.

## Error code (Key = 0x63)

* 0-63 - no action
* 64-127 - display only
* 112-144 - maskable
* 128-191 - pause
* 192-255 - stop

## Flags (Key = 0x8a, 0x8b)

* 0x8000 - bluetooth control


## Pin

1. CAP1 - touch 1
1. CAP2 - touch 2
1. CAP3 - touch 3
1. CAP4 - touch 4
1. CAP5 - touch 5
1. CAP6 - touch 6
1. CAP7 - touch 7
1. CAP8 - touch 8
1. RES1
1. SWA - switch A
1. SWB - switch B
1. SW2 - switch 2
1. RES2
1. RES3
1. RES4
1. RES5
1. \__VAA - analog power
1. \__GNDA - analog ground
1. \__NTC - NTC
1. \__GNDD - digital ground
1. \__VDD - digital power
1. \__RX - UART RX
1. \__TX - UART TX
1. INT
1. SCL
1. SDA
1. COM - display background
1. LED1 - display 1
1. LED2 - display 2
1. LED3 - display 3
1. LED4 - display 4
1. LED5 - display 5
1. LED6 - display 6
1. LED7 - display 7
1. LED8 - display 8
1. LED9 - display 9
1. LED10 - display 10
1. LED11 - display 11
1. CAPC - touch common

![views](https://raw.githubusercontent.com/a-kitchen/module/master/layout.png)

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

static U08 cled = LED_OFF;                          // local display
static U08 cmod = AkMODE_OFF;                       // local mode
static U08 dkey;                                    // downstream data key
static U08 dlen;                                    // downstream data length
static U08 rmod = AkMODE_OFF;                       // remote mode
static U16 hdwr;                                    // device information

U08 Ak_GetParam08(U08 key) {                        // hook for compiling
  return key == KEY_LED ? cled : 0;
}

void Ak_SetParam08(U08 key, U08 value) {            // hook for compiling
  if(key == KEY_LED)
    cled = value;
}

static void send(U08*, U08);

static void beat(void) {
  if(dkey != KEY_HDWARE_HI || dlen != 12) {         // check format of downstream data
    static U08 ini[] = {
      'A', 'K', LEN, SUM, KEY_DATFRAME, 1,              // downstream data initialization
      KEY_MODE, KEY_ERRCODE,                        // downstream data format
      KEY_HDWARE_HI, KEY_HDWARE_LO,
    };
    send(ini, sizeof ini);                          // initialization data format
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
    send(inf, sizeof inf);                          // initialize device information
  } else {
    static U08 dat[] = {                            // upstream data frame
      'A', 'K', LEN, SUM,                   // header
      KEY_MODE,    0,                       // 4,  5
      KEY_ERRCODE, 0,                       // 6,  7
      KEY_TEMP_HI, 0,                       // 8,  9
      KEY_TEMP_LO, 0,                       // 10, 11
    };                                      // 12
    U16 tmp = (Clock_millisecond / 1000) & 0x3fff;  // simulate temperature，0.01/s
    dat[5] = cmod;                                  // set value
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

static void stream(U08 value) {                     // parse downstream data
  static U08 a, k, n;

  a += value;
  n--;
  switch (k) {
  case 0:                                           // check header
    if (value == 'A')
      k = 1;
    return;
  case 1:                                           // check header
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
      n = value - 2;                                // frame length
      dlen = value;
    }
    return;
  case 3:                                          
    k = 4;
    return;
  case 4:
    if(n){
      k = value;                                    // key
      return;
    }
    break;
  default:
    if(n){                                          // not end of frame
      if(n == 3)                                    
        dkey = k;                                   //   flags of downstream data frame
      switch(k) {
      case KEY_MODE:                                // remote mode
        rmod = value;
        //if (cmod == AkMODE_PREP && rmod == AkMODE_PREP)
        if (rmod == AkMODE_PREP)
          cmod = AkMODE_RUN;
        else if (cmod && (cmod == rmod || (cmod == AkMODE_RETURN && rmod >= AkMODE_APPMIN) ||
            (rmod > AkMODE_RUN && rmod < AkMODE_BOOTING)))
          cmod = 0;                                 // sync to local data
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
  if (!(a - value))                                 // if: checksum passed
    Clock_Light(3);                                 // LED on indicates success
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
    case EVENT_BEAT:                                // beat, 16 times/sec
      Clock_OnBeat();
      Light_OnBeat();
      beat();
      break;
    case EVENT_IDLE:                                // idle
      Knob_OnIdle();
      break;
    case EVENT_LPRESS:                              // long press
      if(Knob_key == KNOB_POWER)                    // power button
        cmod = mod == AkMODE_OFF ? AkMODE_L_0 : AkMODE_OFF;
      break;
    case EVENT_RELEASE:                             // release button
      if(mod == AkMODE_OFF)
        break;
	  switch(Knob_key) {
      case KNOB_MINUS:                              // turn left
        if (mod > AkMODE_L_1)
          cmod = mod - 1;
        else if (mod == AkMODE_L_1 || mod == AkMODE_PAIRING)
          cmod = AkMODE_L_0;
        break;
      case KNOB_PLUS:                               // turn right
        if (mod == AkMODE_L_0 || mod == AkMODE_PAIRING)
          cmod = AkMODE_L_1;
        else if (mod < AkMODE_L_9)
          cmod = mod + 1;
        break;
      case KNOB_POWER:                              // power button
        if (mod <= AkMODE_READYMAX || mod == AkMODE_PAIRING)
          cmod = AkMODE_L_5;
        else if (mod < AkMODE_APPMIN)
          cmod = AkMODE_RETURN;                     // resume
        else if (mod <= AkMODE_APPMAX)
          cmod = AkMODE_READYMAX + 1;               // pause
        else if (mod >= AkMODE_L_1 || mod <= AkMODE_L_9)
          cmod = AkMODE_L_0;
        else if (mod == AkMODE_WAITNEXT || mod == AkMODE_PAUSE)
          cmod = AkMODE_PREP;                       // next step
        else if (mod > AkMODE_PREP && mod != AkMODE_OFF)
          cmod = AkMODE_PAUSE;                      // pause
        break;
      }
      break;
    case EVENT_XPRESS:                              // extra long press
      if(Knob_key == KNOB_POWER)                    //  pairing
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
      Light_OnParam(KEY_LED);                       // set led
    }
    if(SCB_SpiUartGetRxBufferSize())                // get downstream data
      stream(SCB_SpiUartReadRxData());              // parse downstream data
  }
}
```
