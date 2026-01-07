# Linux Kernel HWMON 与 PMBus 子系统详细总结

## 目录
1. [概述](#概述)
2. [架构层次图](#架构层次图)
3. [模块关系图](#模块关系图)
4. [HWMON核心子系统](#hwmon核心子系统)
5. [PMBus子系统](#pmbus子系统)
   - [PMBus数据格式详解](#pmbus数据格式详解)
     - Linear格式 (LINEAR11/LINEAR16)
     - Direct格式 (m/b/R系数)
     - VID格式 (电压标识码)
     - IEEE754半精度浮点格式
   - [PMBus标准寄存器完整列表](#pmbus标准寄存器完整列表)
   - [状态位详解](#状态位详解)
   - [虚拟寄存器机制](#虚拟寄存器机制)
   - [页面与相位管理](#页面与相位管理)
   - [访问延迟机制](#访问延迟机制)
   - [故障处理机制](#故障处理机制)
   - [数据缓存机制](#数据缓存机制)
   - [调节器框架集成](#调节器框架集成)
6. [泳道图：设备初始化流程](#泳道图设备初始化流程)
7. [泳道图：数据读取流程](#泳道图数据读取流程)
8. [关键数据结构](#关键数据结构)
9. [Sysfs接口](#sysfs接口)
10. [PMBus设备驱动开发指南](#pmbus设备驱动开发指南)
    - [驱动开发流程概述](#驱动开发流程概述)
    - [最小驱动示例](#最小驱动示例)
    - [回调函数详解](#回调函数详解)
    - [Direct模式系数配置详解](#direct模式系数配置详解)
    - [多设备变体支持模式](#多设备变体支持模式)
    - [添加调节器支持](#添加调节器支持)
    - [设备轮询与等待模式](#设备轮询与等待模式)
    - [添加自定义sysfs属性](#添加自定义sysfs属性)
    - [使用pmbus_lock保护访问](#使用pmbus_lock保护访问)
11. [调试与诊断](#调试与诊断)
12. [参考资源](#参考资源)
13. [总结](#总结)

---

## 概述

### HWMON (Hardware Monitoring)
HWMON是Linux内核中用于硬件监控的统一框架，提供了标准化的sysfs接口，用于监控：
- **温度** (Temperature)
- **电压** (Voltage) 
- **电流** (Current)
- **功率** (Power)
- **风扇转速** (Fan Speed)
- **湿度** (Humidity)
- **能量** (Energy)

### PMBus (Power Management Bus)
PMBus是一种基于SMBus的开放标准电源管理协议，用于电源转换器的编程、控制和实时监控。Linux PMBus驱动分为三层：
- **pmbus_core.c** - 核心驱动，提供通用功能
- **pmbus.c** - 通用PMBus设备驱动
- **设备特定驱动** - 如ltc2978.c, adm1275.c等

---

## 架构层次图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           用户空间 (User Space)                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────────┐     │
│  │    lm-sensors    │  │   libsensors     │  │   应用程序/脚本         │     │
│  └────────┬─────────┘  └────────┬─────────┘  └───────────┬────────────┘     │
│           │                     │                        │                   │
│           └─────────────────────┴────────────────────────┘                   │
│                                 │                                            │
│                        /sys/class/hwmon/hwmonX/                             │
└─────────────────────────────────┼───────────────────────────────────────────┘
                                  │ sysfs
┌─────────────────────────────────┼───────────────────────────────────────────┐
│                           内核空间 (Kernel Space)                            │
│                                 │                                            │
│  ┌──────────────────────────────▼──────────────────────────────────────┐    │
│  │                      HWMON Core (hwmon.c)                            │    │
│  │  ┌─────────────────────────────────────────────────────────────┐    │    │
│  │  │ - hwmon_device_register_with_info()                         │    │    │
│  │  │ - devm_hwmon_device_register_with_info()                    │    │    │
│  │  │ - 创建/管理sysfs属性                                         │    │    │
│  │  │ - 热区域(Thermal Zone)集成                                   │    │    │
│  │  └─────────────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                 │                                            │
│        ┌────────────────────────┴────────────────────────┐                  │
│        │                                                 │                  │
│        ▼                                                 ▼                  │
│  ┌────────────────────┐                    ┌─────────────────────────────┐  │
│  │  普通HWMON驱动     │                    │     PMBus子系统              │  │
│  │  (如 lm75, lm90)   │                    │                             │  │
│  │                    │                    │  ┌───────────────────────┐  │  │
│  │ - 直接实现hwmon_ops│                    │  │  PMBus Core           │  │  │
│  │ - 自己管理I2C通信  │                    │  │  (pmbus_core.c)       │  │  │
│  │                    │                    │  │                       │  │  │
│  └─────────┬──────────┘                    │  │ - 设备检测/初始化     │  │  │
│            │                               │  │ - 数据格式转换        │  │  │
│            │                               │  │ - sysfs属性创建       │  │  │
│            │                               │  │ - 调节器支持          │  │  │
│            │                               │  │ - debugfs支持         │  │  │
│            │                               │  └───────────┬───────────┘  │  │
│            │                               │              │              │  │
│            │                               │      ┌───────┴───────┐      │  │
│            │                               │      │               │      │  │
│            │                               │      ▼               ▼      │  │
│            │                               │ ┌─────────┐   ┌──────────┐  │  │
│            │                               │ │ pmbus.c │   │ 设备特定 │  │  │
│            │                               │ │ (通用)  │   │ 驱动     │  │  │
│            │                               │ │         │   │          │  │  │
│            │                               │ │ 自动    │   │ ltc2978  │  │  │
│            │                               │ │ 检测    │   │ adm1275  │  │  │
│            │                               │ │ 能力    │   │ max34440 │  │  │
│            │                               │ └────┬────┘   │ ...      │  │  │
│            │                               │      │        └────┬─────┘  │  │
│            │                               │      └─────────────┘        │  │
│            │                               └─────────────────────────────┘  │
│            │                                           │                    │
│            └───────────────────────────────────────────┘                    │
│                                 │                                            │
│  ┌──────────────────────────────▼──────────────────────────────────────┐    │
│  │                       I2C/SMBus Core                                 │    │
│  │  - i2c_smbus_read_word_data()                                       │    │
│  │  - i2c_smbus_write_word_data()                                      │    │
│  │  - i2c_smbus_read_byte_data()                                       │    │
│  │  - PEC (Packet Error Checking) 支持                                 │    │
│  └──────────────────────────────┬──────────────────────────────────────┘    │
│                                 │                                            │
└─────────────────────────────────┼───────────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼───────────────────────────────────────────┐
│                            硬件层 (Hardware)                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────────┐     │
│  │  温度传感器芯片   │  │   电源管理芯片    │  │   PMBus电源转换器      │     │
│  │  (LM75, TMP401)  │  │  (INA2xx系列)    │  │  (LTC2978, ADM1275)   │     │
│  └──────────────────┘  └──────────────────┘  └────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 模块关系图

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                              Linux Kernel HWMON/PMBus 模块关系                  │
└───────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────┐
                              │   <linux/hwmon.h>│
                              │   公共API定义    │
                              └────────┬────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
          ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
          │   hwmon_ops     │ │hwmon_channel_info│ │ hwmon_chip_info │
          │                 │ │                 │ │                 │
          │ • is_visible    │ │ • type          │ │ • ops           │
          │ • read          │ │ • config        │ │ • info          │
          │ • write         │ │                 │ │                 │
          │ • read_string   │ │                 │ │                 │
          └────────┬────────┘ └────────┬────────┘ └────────┬────────┘
                   │                   │                   │
                   └───────────────────┴───────────────────┘
                                       │
                                       ▼
                         ┌─────────────────────────┐
                         │     hwmon.c (核心)       │
                         │                         │
                         │ • 设备注册/注销          │
                         │ • sysfs属性管理         │
                         │ • Thermal Zone集成     │
                         │ • PEC支持              │
                         └────────────┬────────────┘
                                      │
                                      │ 被各驱动使用
           ┌──────────────────────────┼──────────────────────────┐
           │                          │                          │
           ▼                          ▼                          ▼
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│   普通HWMON驱动      │   │  PMBus Core层        │   │   其他HWMON驱动     │
│                     │   │  (pmbus_core.c)      │   │                     │
│ lm75.c              │   │                     │   │ coretemp.c          │
│ lm90.c              │   │ ┌─────────────────┐ │   │ k10temp.c           │
│ tmp102.c            │   │ │<drivers/hwmon/  │ │   │ it87.c              │
│ adt7475.c           │   │ │ pmbus/pmbus.h>  │ │   │ nct6775.c           │
│ ...                 │   │ │                 │ │   │ ...                 │
└─────────────────────┘   │ │ PMBus内部API     │ │   └─────────────────────┘
                          │ │ 寄存器定义       │ │
                          │ │ 虚拟寄存器       │ │
                          │ └────────┬────────┘ │
                          │          │          │
                          │          ▼          │
                          │ ┌─────────────────┐ │
                          │ │pmbus_driver_info│ │
                          │ │                 │ │
                          │ │ • pages         │ │
                          │ │ • format[]      │ │
                          │ │ • func[]        │ │
                          │ │ • m[], b[], R[] │ │
                          │ │ • read_word_data│ │
                          │ │ • write_word_data│ │
                          │ │ • identify      │ │
                          │ └────────┬────────┘ │
                          │          │          │
                          └──────────┼──────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │   pmbus.c       │    │   ltc2978.c     │    │   adm1275.c     │
    │   (通用驱动)     │    │                 │    │                 │
    │                 │    │ LTC2978/2977    │    │ ADM1275/1276    │
    │ • 自动检测      │    │ LTC2974/2975    │    │ ADM1278/1293    │
    │ • 动态能力发现   │    │ LTC3880/3883    │    │                 │
    │                 │    │ ...             │    │                 │
    └─────────────────┘    └─────────────────┘    └─────────────────┘
              │                      │                      │
              └──────────────────────┼──────────────────────┘
                                     │
                                     ▼
                          ┌─────────────────────┐
                          │   pmbus_do_probe()  │
                          │                     │
                          │ • 初始化通用设置     │
                          │ • 调用identify()    │
                          │ • 发现传感器        │
                          │ • 创建sysfs属性     │
                          │ • 注册hwmon设备     │
                          │ • 注册调节器        │
                          │ • 设置debugfs       │
                          └─────────────────────┘
```

---

## HWMON核心子系统

### 核心文件
- `drivers/hwmon/hwmon.c` - HWMON核心实现
- `include/linux/hwmon.h` - 公共API头文件

### 主要功能

#### 1. 设备注册
```c
// 推荐使用的注册函数
struct device *devm_hwmon_device_register_with_info(
    struct device *dev,           // 父设备
    const char *name,             // hwmon名称
    void *drvdata,                // 驱动私有数据
    const struct hwmon_chip_info *chip,    // 芯片信息
    const struct attribute_group **extra_groups  // 额外属性组
);
```

#### 2. 传感器类型
| 类型 | 枚举值 | 描述 |
|------|--------|------|
| hwmon_chip | 虚拟 | 芯片级属性 |
| hwmon_temp | 温度 | 温度传感器 (毫摄氏度) |
| hwmon_in | 电压 | 电压传感器 (毫伏) |
| hwmon_curr | 电流 | 电流传感器 (毫安) |
| hwmon_power | 功率 | 功率传感器 (微瓦) |
| hwmon_energy | 能量 | 能量传感器 (微焦) |
| hwmon_humidity | 湿度 | 湿度传感器 |
| hwmon_fan | 风扇 | 风扇转速 (RPM) |
| hwmon_pwm | PWM | PWM控制 |

#### 3. hwmon_ops回调结构
```c
struct hwmon_ops {
    umode_t (*is_visible)(const void *drvdata,
                          enum hwmon_sensor_types type,
                          u32 attr, int channel);
    int (*read)(struct device *dev,
                enum hwmon_sensor_types type,
                u32 attr, int channel, long *val);
    int (*read_string)(struct device *dev,
                       enum hwmon_sensor_types type,
                       u32 attr, int channel, const char **str);
    int (*write)(struct device *dev,
                 enum hwmon_sensor_types type,
                 u32 attr, int channel, long val);
};
```

---

## PMBus子系统

### 核心文件
- `drivers/hwmon/pmbus/pmbus_core.c` - PMBus核心实现
- `drivers/hwmon/pmbus/pmbus.c` - 通用PMBus驱动
- `drivers/hwmon/pmbus/pmbus.h` - 内部API头文件
- `include/linux/pmbus.h` - 平台数据头文件

### PMBus协议特点

1. **基于SMBus**: 使用标准SMBus协议进行通信
2. **命令标准化**: 定义了0x00-0xFF的标准命令
3. **分页支持**: 多输出设备使用PAGE命令选择输出
4. **相位支持**: 多相电源使用PHASE命令选择相位
5. **PEC支持**: 支持数据包错误检查(Packet Error Checking)

---

### PMBus数据格式详解

PMBus支持四种主要数据格式，每种格式有其特定的转换公式和应用场景：

#### 1. Linear格式 (LINEAR11)

**格式结构**: 5位有符号指数 + 11位有符号尾数

```
┌─────────────────────────────────────────────────────────────┐
│ Bit 15-11 │      Bit 10-0        │
│ Exponent  │      Mantissa        │
│ (5-bit)   │      (11-bit)        │
└─────────────────────────────────────────────────────────────┘
```

**寄存器值→实际值转换** (`pmbus_reg2data_linear`):
```c
exponent = ((s16)data) >> 11;           // 提取指数，符号扩展
mantissa = ((s16)((data & 0x7ff) << 5)) >> 5;  // 提取尾数，符号扩展

val = mantissa;
if (sensor_class != PSC_FAN)
    val = val * 1000;      // 转换为毫单位
if (sensor_class == PSC_POWER)
    val = val * 1000;      // 功率转换为微单位

if (exponent >= 0)
    val <<= exponent;
else
    val >>= -exponent;
```

**公式**: `实际值 = mantissa × 2^exponent`

**实际值→寄存器值转换** (`pmbus_data2reg_linear`):
```c
// 归一化尾数到10位范围 (511-1023)
while (val >= MAX_LIN_MANTISSA && exponent < 15) {
    exponent++;
    val >>= 1;
}
while (val < MIN_LIN_MANTISSA && exponent > -15) {
    exponent--;
    val <<= 1;
}
mantissa = DIV_ROUND_CLOSEST(val, 1000);
return (mantissa & 0x7ff) | ((exponent << 11) & 0xf800);
```

#### 2. Linear16格式

**格式结构**: 纯16位无符号整数，指数由`VOUT_MODE`寄存器定义

```
┌─────────────────────────────────────────────────────────────┐
│                    Bit 15-0                                 │
│                    Mantissa (16-bit unsigned)               │
└─────────────────────────────────────────────────────────────┘
```

**主要用于**: 输出电压 (`PSC_VOLTAGE_OUT`)

**转换公式**: `实际值 = data × 2^exponent`

其中`exponent`从`VOUT_MODE`寄存器的低5位获取（有符号）:
```c
vout_mode = pmbus_read_byte_data(client, page, PMBUS_VOUT_MODE);
exponent = ((s8)(vout_mode << 3)) >> 3;  // 符号扩展5位到8位
data->exponent[page] = exponent;
```

#### 3. Direct格式

**格式结构**: 16位有符号整数，使用三个系数(m, b, R)进行转换

**寄存器值→实际值转换** (`pmbus_reg2data_direct`):
```
X = (1/m) × (Y × 10^(-R) - b)
```

其中：
- `Y` = 原始寄存器值（16位有符号）
- `m` = 斜率系数（mantissa）
- `b` = 偏移系数（offset）
- `R` = 缩放指数

**源码实现**:
```c
static s64 pmbus_reg2data_direct(struct pmbus_data *data,
                                 struct pmbus_sensor *sensor)
{
    s64 b, val = (s16)sensor->data;
    s32 m, R;

    m = data->info->m[sensor->class];
    b = data->info->b[sensor->class];
    R = data->info->R[sensor->class];

    if (m == 0)
        return 0;

    // X = 1/m * (Y * 10^-R - b)
    R = -R;
    // 非风扇/PWM类型缩放到毫单位
    if (!(sensor->class == PSC_FAN || sensor->class == PSC_PWM)) {
        R += 3;
        b *= 1000;
    }
    // 功率类型额外缩放到微单位
    if (sensor->class == PSC_POWER) {
        R += 3;
        b *= 1000;
    }

    while (R > 0) { val *= 10; R--; }
    while (R < 0) { val = div_s64(val + 5LL, 10L); R++; }  // 四舍五入

    val = div_s64(val - b, m);
    return val;
}
```

**实际值→寄存器值转换** (`pmbus_data2reg_direct`):
```
Y = (m × X + b) × 10^R
```

```c
static u16 pmbus_data2reg_direct(struct pmbus_data *data,
                                 struct pmbus_sensor *sensor, s64 val)
{
    s64 b;
    s32 m, R;

    m = data->info->m[sensor->class];
    b = data->info->b[sensor->class];
    R = data->info->R[sensor->class];

    // 功率调整
    if (sensor->class == PSC_POWER) {
        R -= 3;
        b *= 1000;
    }
    // 非风扇/PWM调整
    if (!(sensor->class == PSC_FAN || sensor->class == PSC_PWM)) {
        R -= 3;
        b *= 1000;
    }

    val = val * m + b;

    while (R > 0) { val *= 10; R--; }
    while (R < 0) { val = div_s64(val + 5LL, 10L); R++; }

    return (u16)clamp_val(val, S16_MIN, S16_MAX);
}
```

#### 4. VID格式 (Voltage Identification)

**用途**: Intel/AMD VRM规范的电压标识码

**支持的VRM版本**:
| 版本 | 枚举值 | 电压范围 | 步进 |
|------|--------|----------|------|
| VR11 | `vr11` | 0.5V - 1.6V | 6.25mV |
| VR12 | `vr12` | 0.25V - 1.52V | 5mV |
| VR13 | `vr13` | 0.5V - 1.52V | 10mV |
| IMVP9 | `imvp9` | 0.2V - 1.52V | 10mV |
| AMD 625mV | `amd625mv` | 0.2V - 1.55V | 6.25mV |

**VR11转换公式**:
```c
if (val >= 0x02 && val <= 0xb2)
    rv = DIV_ROUND_CLOSEST(160000 - (val - 2) * 625, 100);  // mV
```

**VR12转换公式**:
```c
if (val >= 0x01)
    rv = 250 + (val - 1) * 5;  // mV
```

**VR13转换公式**:
```c
if (val >= 0x01)
    rv = 500 + (val - 1) * 10;  // mV
```

#### 5. IEEE754半精度浮点格式

**格式结构**: 1位符号 + 5位指数 + 10位尾数

```
┌─────────────────────────────────────────────────────────────┐
│ Bit 15 │  Bit 14-10  │      Bit 9-0          │
│ Sign   │  Exponent   │      Mantissa         │
│ (1-bit)│  (5-bit)    │      (10-bit)         │
└─────────────────────────────────────────────────────────────┘
```

**转换公式**:
- **次正规数** (exponent=0): `(-1)^sign × 2^(-14) × 0.mantissa`
- **正规数** (exponent=1-30): `(-1)^sign × 2^(exponent-15) × 1.mantissa`
- **NaN/Inf** (exponent=31): 转换为最大值65504

**源码实现** (`pmbus_reg2data_ieee754`):
```c
sign = sensor->data & 0x8000;
exponent = (sensor->data >> 10) & 0x1f;
val = sensor->data & 0x3ff;

if (exponent == 0) {            // 次正规数
    exponent = -(14 + 10);
} else if (exponent == 0x1f) {  // NaN
    exponent = 0;
    val = 65504;
} else {                        // 正规数
    exponent -= (15 + 10);
    val |= 0x400;               // 隐含的1
}

// 缩放到毫/微单位
if (sensor->class != PSC_FAN)
    val = val * 1000L;
if (sensor->class == PSC_POWER)
    val = val * 1000L;

if (exponent >= 0)
    val <<= exponent;
else
    val >>= -exponent;

if (sign)
    val = -val;
```

---

### PMBus标准寄存器完整列表

#### 控制寄存器 (0x00-0x1F)

| 寄存器 | 地址 | 类型 | 描述 |
|--------|------|------|------|
| PAGE | 0x00 | R/W | 页面/输出选择 |
| OPERATION | 0x01 | R/W | 操作控制命令 |
| ON_OFF_CONFIG | 0x02 | R/W | 开关配置 |
| CLEAR_FAULTS | 0x03 | W | 清除所有故障 |
| PHASE | 0x04 | R/W | 相位选择 |
| WRITE_PROTECT | 0x10 | R/W | 写保护配置 |
| CAPABILITY | 0x19 | R | 设备能力 |
| QUERY | 0x1A | R | 命令支持查询 |
| SMBALERT_MASK | 0x1B | R/W | SMBAlert掩码 |

#### 输出电压配置 (0x20-0x2F)

| 寄存器 | 地址 | 类型 | 描述 |
|--------|------|------|------|
| VOUT_MODE | 0x20 | R/W | 输出电压模式 |
| VOUT_COMMAND | 0x21 | R/W | 输出电压命令值 |
| VOUT_TRIM | 0x22 | R/W | 输出电压微调 |
| VOUT_CAL_OFFSET | 0x23 | R/W | 校准偏移 |
| VOUT_MAX | 0x24 | R/W | 输出电压上限 |
| VOUT_MARGIN_HIGH | 0x25 | R/W | 高边际电压 |
| VOUT_MARGIN_LOW | 0x26 | R/W | 低边际电压 |
| VOUT_TRANSITION_RATE | 0x27 | R/W | 电压转换速率 |
| VOUT_DROOP | 0x28 | R/W | 下垂控制 |
| VOUT_SCALE_LOOP | 0x29 | R/W | 环路缩放 |
| VOUT_SCALE_MONITOR | 0x2A | R/W | 监控缩放 |

#### 系数读取 (0x30-0x31)

| 寄存器 | 地址 | 类型 | 描述 |
|--------|------|------|------|
| COEFFICIENTS | 0x30 | Block | m/b/R系数读取 |
| POUT_MAX | 0x31 | R/W | 最大输出功率 |

#### 风扇配置 (0x3A-0x3F)

| 寄存器 | 地址 | 类型 | 描述 |
|--------|------|------|------|
| FAN_CONFIG_12 | 0x3A | R/W | 风扇1/2配置 |
| FAN_COMMAND_1 | 0x3B | R/W | 风扇1命令 |
| FAN_COMMAND_2 | 0x3C | R/W | 风扇2命令 |
| FAN_CONFIG_34 | 0x3D | R/W | 风扇3/4配置 |
| FAN_COMMAND_3 | 0x3E | R/W | 风扇3命令 |
| FAN_COMMAND_4 | 0x3F | R/W | 风扇4命令 |

#### 故障限制 (0x40-0x6B)

| 寄存器 | 地址 | 描述 |
|--------|------|------|
| VOUT_OV_FAULT_LIMIT | 0x40 | 输出过压故障限制 |
| VOUT_OV_FAULT_RESPONSE | 0x41 | 过压故障响应 |
| VOUT_OV_WARN_LIMIT | 0x42 | 输出过压警告限制 |
| VOUT_UV_WARN_LIMIT | 0x43 | 输出欠压警告限制 |
| VOUT_UV_FAULT_LIMIT | 0x44 | 输出欠压故障限制 |
| IOUT_OC_FAULT_LIMIT | 0x46 | 输出过流故障限制 |
| IOUT_OC_WARN_LIMIT | 0x4A | 输出过流警告限制 |
| IOUT_UC_FAULT_LIMIT | 0x4B | 输出欠流故障限制 |
| OT_FAULT_LIMIT | 0x4F | 过温故障限制 |
| OT_WARN_LIMIT | 0x51 | 过温警告限制 |
| UT_WARN_LIMIT | 0x52 | 欠温警告限制 |
| UT_FAULT_LIMIT | 0x53 | 欠温故障限制 |
| VIN_OV_FAULT_LIMIT | 0x55 | 输入过压故障限制 |
| VIN_OV_WARN_LIMIT | 0x57 | 输入过压警告限制 |
| VIN_UV_WARN_LIMIT | 0x58 | 输入欠压警告限制 |
| VIN_UV_FAULT_LIMIT | 0x59 | 输入欠压故障限制 |
| IIN_OC_FAULT_LIMIT | 0x5B | 输入过流故障限制 |
| IIN_OC_WARN_LIMIT | 0x5D | 输入过流警告限制 |
| POUT_OP_FAULT_LIMIT | 0x68 | 输出过功率故障限制 |
| POUT_OP_WARN_LIMIT | 0x6A | 输出过功率警告限制 |
| PIN_OP_WARN_LIMIT | 0x6B | 输入过功率警告限制 |

#### 状态寄存器 (0x78-0x82)

| 寄存器 | 地址 | 位宽 | 描述 |
|--------|------|------|------|
| STATUS_BYTE | 0x78 | 8-bit | 状态字节 |
| STATUS_WORD | 0x79 | 16-bit | 状态字 |
| STATUS_VOUT | 0x7A | 8-bit | 输出电压状态 |
| STATUS_IOUT | 0x7B | 8-bit | 输出电流状态 |
| STATUS_INPUT | 0x7C | 8-bit | 输入状态 |
| STATUS_TEMPERATURE | 0x7D | 8-bit | 温度状态 |
| STATUS_CML | 0x7E | 8-bit | 通信/逻辑/内存状态 |
| STATUS_OTHER | 0x7F | 8-bit | 其他状态 |
| STATUS_MFR_SPECIFIC | 0x80 | 8-bit | 厂商特定状态 |
| STATUS_FAN_12 | 0x81 | 8-bit | 风扇1/2状态 |
| STATUS_FAN_34 | 0x82 | 8-bit | 风扇3/4状态 |

#### 遥测寄存器 (0x88-0x97)

| 寄存器 | 地址 | 描述 |
|--------|------|------|
| READ_VIN | 0x88 | 读取输入电压 |
| READ_IIN | 0x89 | 读取输入电流 |
| READ_VCAP | 0x8A | 读取电容电压 |
| READ_VOUT | 0x8B | 读取输出电压 |
| READ_IOUT | 0x8C | 读取输出电流 |
| READ_TEMPERATURE_1 | 0x8D | 读取温度1 |
| READ_TEMPERATURE_2 | 0x8E | 读取温度2 |
| READ_TEMPERATURE_3 | 0x8F | 读取温度3 |
| READ_FAN_SPEED_1 | 0x90 | 读取风扇1转速 |
| READ_FAN_SPEED_2 | 0x91 | 读取风扇2转速 |
| READ_FAN_SPEED_3 | 0x92 | 读取风扇3转速 |
| READ_FAN_SPEED_4 | 0x93 | 读取风扇4转速 |
| READ_DUTY_CYCLE | 0x94 | 读取占空比 |
| READ_FREQUENCY | 0x95 | 读取开关频率 |
| READ_POUT | 0x96 | 读取输出功率 |
| READ_PIN | 0x97 | 读取输入功率 |

#### 识别寄存器 (0x98-0xAE)

| 寄存器 | 地址 | 描述 |
|--------|------|------|
| REVISION | 0x98 | PMBus修订版本 |
| MFR_ID | 0x99 | 制造商ID |
| MFR_MODEL | 0x9A | 型号 |
| MFR_REVISION | 0x9B | 制造商修订版本 |
| MFR_LOCATION | 0x9C | 生产位置 |
| MFR_DATE | 0x9D | 生产日期 |
| MFR_SERIAL | 0x9E | 序列号 |
| MFR_VIN_MIN | 0xA0 | 最小输入电压规格 |
| MFR_VIN_MAX | 0xA1 | 最大输入电压规格 |
| MFR_IIN_MAX | 0xA2 | 最大输入电流规格 |
| MFR_PIN_MAX | 0xA3 | 最大输入功率规格 |
| MFR_VOUT_MIN | 0xA4 | 最小输出电压规格 |
| MFR_VOUT_MAX | 0xA5 | 最大输出电压规格 |
| MFR_IOUT_MAX | 0xA6 | 最大输出电流规格 |
| MFR_POUT_MAX | 0xA7 | 最大输出功率规格 |
| IC_DEVICE_ID | 0xAD | IC设备ID |
| IC_DEVICE_REV | 0xAE | IC设备修订版本 |
| MFR_MAX_TEMP_1 | 0xC0 | 温度1最大规格 |
| MFR_MAX_TEMP_2 | 0xC1 | 温度2最大规格 |
| MFR_MAX_TEMP_3 | 0xC2 | 温度3最大规格 |

---

### 状态位详解

#### STATUS_WORD位定义

**低8位 (STATUS_BYTE)**:
| 位 | 宏定义 | 描述 |
|----|--------|------|
| 0 | `PB_STATUS_NONE_ABOVE` | 无上述故障 |
| 1 | `PB_STATUS_CML` | 通信/内存/逻辑故障 |
| 2 | `PB_STATUS_TEMPERATURE` | 温度故障/警告 |
| 3 | `PB_STATUS_VIN_UV` | 输入欠压 |
| 4 | `PB_STATUS_IOUT_OC` | 输出过流 |
| 5 | `PB_STATUS_VOUT_OV` | 输出过压 |
| 6 | `PB_STATUS_OFF` | 设备关闭 |
| 7 | `PB_STATUS_BUSY` | 设备忙 |

**高8位**:
| 位 | 宏定义 | 描述 |
|----|--------|------|
| 8 | `PB_STATUS_UNKNOWN` | 未知故障 |
| 9 | `PB_STATUS_OTHER` | 其他故障 |
| 10 | `PB_STATUS_FANS` | 风扇故障 |
| 11 | `PB_STATUS_POWER_GOOD_N` | 电源不良 |
| 12 | `PB_STATUS_WORD_MFR` | 厂商特定故障 |
| 13 | `PB_STATUS_INPUT` | 输入故障 |
| 14 | `PB_STATUS_IOUT_POUT` | 输出电流/功率故障 |
| 15 | `PB_STATUS_VOUT` | 输出电压故障 |

#### STATUS_IOUT位定义

| 位 | 宏定义 | 描述 |
|----|--------|------|
| 0 | `PB_POUT_OP_WARNING` | 输出功率警告 |
| 1 | `PB_POUT_OP_FAULT` | 输出功率故障 |
| 2 | `PB_POWER_LIMITING` | 功率限制中 |
| 3 | `PB_CURRENT_SHARE_FAULT` | 均流故障 |
| 4 | `PB_IOUT_UC_FAULT` | 输出欠流故障 |
| 5 | `PB_IOUT_OC_WARNING` | 输出过流警告 |
| 6 | `PB_IOUT_OC_LV_FAULT` | 输出过流低压故障 |
| 7 | `PB_IOUT_OC_FAULT` | 输出过流故障 |

#### STATUS_VOUT/STATUS_INPUT电压位定义

| 位 | 宏定义 | 描述 |
|----|--------|------|
| 3 | `PB_VOLTAGE_VIN_OFF` | 输入电压关闭 |
| 4 | `PB_VOLTAGE_UV_FAULT` | 欠压故障 |
| 5 | `PB_VOLTAGE_UV_WARNING` | 欠压警告 |
| 6 | `PB_VOLTAGE_OV_WARNING` | 过压警告 |
| 7 | `PB_VOLTAGE_OV_FAULT` | 过压故障 |

#### STATUS_TEMPERATURE位定义

| 位 | 宏定义 | 描述 |
|----|--------|------|
| 4 | `PB_TEMP_UT_FAULT` | 欠温故障 |
| 5 | `PB_TEMP_UT_WARNING` | 欠温警告 |
| 6 | `PB_TEMP_OT_WARNING` | 过温警告 |
| 7 | `PB_TEMP_OT_FAULT` | 过温故障 |

#### STATUS_FAN位定义

| 位 | 宏定义 | 描述 |
|----|--------|------|
| 0 | `PB_FAN_AIRFLOW_WARNING` | 风量警告 |
| 1 | `PB_FAN_AIRFLOW_FAULT` | 风量故障 |
| 2 | `PB_FAN_FAN2_SPEED_OVERRIDE` | 风扇2速度覆盖 |
| 3 | `PB_FAN_FAN1_SPEED_OVERRIDE` | 风扇1速度覆盖 |
| 4 | `PB_FAN_FAN2_WARNING` | 风扇2警告 |
| 5 | `PB_FAN_FAN1_WARNING` | 风扇1警告 |
| 6 | `PB_FAN_FAN2_FAULT` | 风扇2故障 |
| 7 | `PB_FAN_FAN1_FAULT` | 风扇1故障 |

#### STATUS_CML位定义

| 位 | 宏定义 | 描述 |
|----|--------|------|
| 0 | `PB_CML_FAULT_OTHER_MEM_LOGIC` | 其他内存/逻辑故障 |
| 1 | `PB_CML_FAULT_OTHER_COMM` | 其他通信故障 |
| 3 | `PB_CML_FAULT_PROCESSOR` | 处理器故障 |
| 4 | `PB_CML_FAULT_MEMORY` | 内存故障 |
| 5 | `PB_CML_FAULT_PACKET_ERROR` | 数据包错误 |
| 6 | `PB_CML_FAULT_INVALID_DATA` | 无效数据 |
| 7 | `PB_CML_FAULT_INVALID_COMMAND` | 无效命令 |

---

### 虚拟寄存器机制

虚拟寄存器是PMBus核心为支持非标准功能而定义的抽象寄存器，从`0x100`开始编号。

#### 虚拟寄存器类型

**1. 历史记录读取寄存器**:
```c
PMBUS_VIRT_READ_TEMP_AVG,      // 平均温度
PMBUS_VIRT_READ_TEMP_MIN,      // 最低温度
PMBUS_VIRT_READ_TEMP_MAX,      // 最高温度
PMBUS_VIRT_READ_VIN_AVG,       // 平均输入电压
PMBUS_VIRT_READ_VIN_MIN,       // 最低输入电压
PMBUS_VIRT_READ_VIN_MAX,       // 最高输入电压
PMBUS_VIRT_READ_IIN_AVG,       // 平均输入电流
PMBUS_VIRT_READ_IIN_MIN,       // 最低输入电流
PMBUS_VIRT_READ_IIN_MAX,       // 最高输入电流
PMBUS_VIRT_READ_PIN_AVG,       // 平均输入功率
PMBUS_VIRT_READ_PIN_MIN,       // 最低输入功率
PMBUS_VIRT_READ_PIN_MAX,       // 最高输入功率
PMBUS_VIRT_READ_POUT_AVG,      // 平均输出功率
PMBUS_VIRT_READ_POUT_MIN,      // 最低输出功率
PMBUS_VIRT_READ_POUT_MAX,      // 最高输出功率
PMBUS_VIRT_READ_VOUT_AVG,      // 平均输出电压
PMBUS_VIRT_READ_VOUT_MIN,      // 最低输出电压
PMBUS_VIRT_READ_VOUT_MAX,      // 最高输出电压
PMBUS_VIRT_READ_IOUT_AVG,      // 平均输出电流
PMBUS_VIRT_READ_IOUT_MIN,      // 最低输出电流
PMBUS_VIRT_READ_IOUT_MAX,      // 最高输出电流
PMBUS_VIRT_READ_TEMP2_AVG,     // 温度2平均值
PMBUS_VIRT_READ_TEMP2_MIN,     // 温度2最低值
PMBUS_VIRT_READ_TEMP2_MAX,     // 温度2最高值
```

**2. 历史记录重置寄存器**:
```c
PMBUS_VIRT_RESET_TEMP_HISTORY,    // 重置温度历史
PMBUS_VIRT_RESET_VIN_HISTORY,     // 重置输入电压历史
PMBUS_VIRT_RESET_IIN_HISTORY,     // 重置输入电流历史
PMBUS_VIRT_RESET_PIN_HISTORY,     // 重置输入功率历史
PMBUS_VIRT_RESET_POUT_HISTORY,    // 重置输出功率历史
PMBUS_VIRT_RESET_VOUT_HISTORY,    // 重置输出电压历史
PMBUS_VIRT_RESET_IOUT_HISTORY,    // 重置输出电流历史
PMBUS_VIRT_RESET_TEMP2_HISTORY,   // 重置温度2历史
```

**3. 电压监控虚拟寄存器**:
```c
PMBUS_VIRT_READ_VMON,             // 读取监控电压
PMBUS_VIRT_VMON_UV_WARN_LIMIT,    // VMON欠压警告限制
PMBUS_VIRT_VMON_OV_WARN_LIMIT,    // VMON过压警告限制
PMBUS_VIRT_VMON_UV_FAULT_LIMIT,   // VMON欠压故障限制
PMBUS_VIRT_VMON_OV_FAULT_LIMIT,   // VMON过压故障限制
PMBUS_VIRT_STATUS_VMON,           // VMON状态
```

**4. 风扇控制虚拟寄存器**:
```c
PMBUS_VIRT_FAN_TARGET_1,          // 风扇1目标转速
PMBUS_VIRT_FAN_TARGET_2,          // 风扇2目标转速
PMBUS_VIRT_FAN_TARGET_3,          // 风扇3目标转速
PMBUS_VIRT_FAN_TARGET_4,          // 风扇4目标转速
PMBUS_VIRT_PWM_1,                 // PWM1占空比
PMBUS_VIRT_PWM_2,                 // PWM2占空比
PMBUS_VIRT_PWM_3,                 // PWM3占空比
PMBUS_VIRT_PWM_4,                 // PWM4占空比
PMBUS_VIRT_PWM_ENABLE_1,          // PWM1使能
PMBUS_VIRT_PWM_ENABLE_2,          // PWM2使能
PMBUS_VIRT_PWM_ENABLE_3,          // PWM3使能
PMBUS_VIRT_PWM_ENABLE_4,          // PWM4使能
```

**5. 采样配置虚拟寄存器**:
```c
PMBUS_VIRT_SAMPLES,               // 通用采样数
PMBUS_VIRT_IN_SAMPLES,            // 输入采样数
PMBUS_VIRT_CURR_SAMPLES,          // 电流采样数
PMBUS_VIRT_POWER_SAMPLES,         // 功率采样数
PMBUS_VIRT_TEMP_SAMPLES,          // 温度采样数
```

#### 虚拟寄存器工作机制

虚拟寄存器通过驱动的`read_word_data`/`write_word_data`回调实现：

```c
// 核心层处理流程
static int _pmbus_read_word_data(struct i2c_client *client, int page,
                                 int phase, int reg)
{
    struct pmbus_data *data = i2c_get_clientdata(client);
    const struct pmbus_driver_info *info = data->info;
    int status;

    // 首先尝试驱动特定实现
    if (info->read_word_data) {
        status = info->read_word_data(client, page, phase, reg);
        if (status != -ENODATA)
            return status;
    }

    // 如果是虚拟寄存器，使用内置处理
    if (reg >= PMBUS_VIRT_BASE)
        return pmbus_read_virt_reg(client, page, reg);

    // 否则读取标准PMBus寄存器
    return pmbus_read_word_data(client, page, phase, reg);
}
```

#### 驱动实现虚拟寄存器示例 (ltc2978.c)

```c
static int ltc2978_read_word_data(struct i2c_client *client,
                                  int page, int phase, int reg)
{
    const struct pmbus_driver_info *info = pmbus_get_driver_info(client);
    struct ltc2978_data *data = to_ltc2978_data(info);
    int ret;

    switch (reg) {
    case PMBUS_VIRT_READ_VIN_MAX:
        // 映射到厂商特定寄存器
        ret = pmbus_read_word_data(client, page, phase,
                                   LTC2978_MFR_VIN_PEAK);
        if (ret >= 0) {
            if (lin11_to_val(ret) > lin11_to_val(data->vin_max))
                data->vin_max = ret;
            ret = data->vin_max;
        }
        break;
    case PMBUS_VIRT_READ_VOUT_MAX:
        ret = pmbus_read_word_data(client, page, phase,
                                   LTC2978_MFR_VOUT_PEAK);
        if (ret >= 0) {
            // 更新并返回最大值
            if (data->vout_max[page] < ret)
                data->vout_max[page] = ret;
            ret = data->vout_max[page];
        }
        break;
    case PMBUS_VIRT_RESET_VIN_HISTORY:
    case PMBUS_VIRT_RESET_VOUT_HISTORY:
        // 返回0表示支持此功能
        ret = 0;
        break;
    default:
        ret = -ENODATA;  // 让核心层处理
        break;
    }
    return ret;
}

static int ltc2978_write_word_data(struct i2c_client *client,
                                   int page, int reg, u16 word)
{
    int ret;

    switch (reg) {
    case PMBUS_VIRT_RESET_VIN_HISTORY:
        // 清除VIN峰值
        ret = pmbus_write_byte(client, 0, LTC2978_MFR_CLEAR_PEAKS);
        break;
    case PMBUS_VIRT_RESET_VOUT_HISTORY:
        // 清除VOUT峰值
        ret = pmbus_write_byte(client, page, LTC2978_MFR_CLEAR_PEAKS);
        break;
    default:
        ret = -ENODATA;
        break;
    }
    return ret;
}
```

---

### 页面与相位管理

PMBus设备可能有多个输出通道（页面）和多相电源，通过`pmbus_set_page()`统一管理：

```c
int pmbus_set_page(struct i2c_client *client, int page, int phase)
{
    struct pmbus_data *data = i2c_get_clientdata(client);
    int rv;

    if (page < 0)
        return 0;

    // 如果不是虚拟页面，且有多页且页面变化，则切换页面
    if (!(data->info->func[page] & PMBUS_PAGE_VIRTUAL) &&
        data->info->pages > 1 && page != data->currpage) {
        pmbus_wait(client);
        rv = i2c_smbus_write_byte_data(client, PMBUS_PAGE, page);
        pmbus_update_ts(client, true);
        if (rv < 0)
            return rv;

        // 读回验证
        pmbus_wait(client);
        rv = i2c_smbus_read_byte_data(client, PMBUS_PAGE);
        pmbus_update_ts(client, false);
        if (rv < 0)
            return rv;
        if (rv != page)
            return -EIO;
    }
    data->currpage = page;

    // 如果有相位且相位变化，且不是虚拟相位，则切换相位
    if (data->info->phases[page] && data->currphase != phase &&
        !(data->info->func[page] & PMBUS_PHASE_VIRTUAL)) {
        pmbus_wait(client);
        rv = i2c_smbus_write_byte_data(client, PMBUS_PHASE, phase);
        pmbus_update_ts(client, true);
        if (rv)
            return rv;
    }
    data->currphase = phase;

    return 0;
}
```

**页面配置示例**:
```c
static struct pmbus_driver_info ltc3880_info = {
    .pages = 2,  // 2个输出
    .phases[0] = 0,
    .phases[1] = 0,
    .func[0] = PMBUS_HAVE_VIN | PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT
             | PMBUS_HAVE_TEMP | PMBUS_HAVE_STATUS_VOUT
             | PMBUS_HAVE_STATUS_IOUT | PMBUS_HAVE_STATUS_INPUT
             | PMBUS_HAVE_STATUS_TEMP,
    .func[1] = PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT | PMBUS_HAVE_TEMP
             | PMBUS_HAVE_STATUS_VOUT | PMBUS_HAVE_STATUS_IOUT
             | PMBUS_HAVE_STATUS_TEMP,
    // ...
};
```

**虚拟页面/相位**:

有些设备的页面/相位只是软件概念，不需要实际写入`PAGE`/`PHASE`寄存器：
```c
#define PMBUS_PHASE_VIRTUAL    BIT(30)  // 相位是虚拟的
#define PMBUS_PAGE_VIRTUAL     BIT(31)  // 页面是虚拟的
```

---

### 访问延迟机制

某些PMBus设备需要在连续访问之间添加延迟：

```c
struct pmbus_driver_info {
    // ...
    int access_delay;    // 所有访问后的延迟（微秒）
    int write_delay;     // 仅写操作后的延迟（微秒）
};

// 等待实现
static void pmbus_wait(struct i2c_client *client)
{
    struct pmbus_data *data = i2c_get_clientdata(client);
    const struct pmbus_driver_info *info = data->info;
    s64 delta;

    if (info->access_delay) {
        delta = ktime_us_delta(ktime_get(), data->access_time);
        if (delta < info->access_delay)
            fsleep(info->access_delay - delta);
    } else if (info->write_delay) {
        delta = ktime_us_delta(ktime_get(), data->write_time);
        if (delta < info->write_delay)
            fsleep(info->write_delay - delta);
    }
}

// 更新时间戳
static void pmbus_update_ts(struct i2c_client *client, bool write_op)
{
    struct pmbus_data *data = i2c_get_clientdata(client);
    const struct pmbus_driver_info *info = data->info;

    if (info->access_delay) {
        data->access_time = ktime_get();
    } else if (info->write_delay && write_op) {
        data->write_time = ktime_get();
    }
}
```

**使用示例**:
```c
static struct pmbus_driver_info adm1275_info = {
    // ...
    .access_delay = 100,  // 100微秒访问延迟
};
```

---

### 故障处理机制

#### 故障清除

```c
// 清除单页故障
static void pmbus_clear_fault_page(struct i2c_client *client, int page)
{
    _pmbus_write_byte(client, page, PMBUS_CLEAR_FAULTS);
}

// 清除所有页面故障
void pmbus_clear_faults(struct i2c_client *client)
{
    struct pmbus_data *data = i2c_get_clientdata(client);
    int i;

    for (i = 0; i < data->info->pages; i++)
        pmbus_clear_fault_page(client, i);
}
```

#### CML故障检测

```c
static int pmbus_check_status_cml(struct i2c_client *client)
{
    struct pmbus_data *data = i2c_get_clientdata(client);
    int status, status2;

    status = data->read_status(client, -1);
    if (status < 0 || (status & PB_STATUS_CML)) {
        status2 = _pmbus_read_byte_data(client, -1, PMBUS_STATUS_CML);
        if (status2 < 0 || (status2 & PB_CML_FAULT_INVALID_COMMAND))
            return -EIO;
    }
    return 0;
}
```

#### 寄存器存在性检查

```c
bool pmbus_check_word_register(struct i2c_client *client, int page, int reg)
{
    return pmbus_check_register(client, __pmbus_read_word_data, page, reg);
}

bool pmbus_check_byte_register(struct i2c_client *client, int page, int reg)
{
    return pmbus_check_register(client, _pmbus_read_byte_data, page, reg);
}

static bool pmbus_check_register(struct i2c_client *client,
                                 int (*func)(struct i2c_client *, int, int),
                                 int page, int reg)
{
    int rv;
    struct pmbus_data *data = i2c_get_clientdata(client);

    rv = func(client, page, reg);
    if (rv >= 0 && !(data->flags & PMBUS_SKIP_STATUS_CHECK))
        rv = pmbus_check_status_cml(client);
    if (rv < 0 && (data->flags & PMBUS_READ_STATUS_AFTER_FAILED_CHECK))
        data->read_status(client, -1);
    if (reg < PMBUS_VIRT_BASE)
        pmbus_clear_fault_page(client, -1);
    return rv >= 0;
}
```

---

### 数据缓存机制

#### 清除缓存

```c
void pmbus_clear_cache(struct i2c_client *client)
{
    struct pmbus_data *data = i2c_get_clientdata(client);
    struct pmbus_sensor *sensor;

    for (sensor = data->sensors; sensor; sensor = sensor->next)
        sensor->data = -ENODATA;
}
```

#### 设置传感器更新标志

```c
void pmbus_set_update(struct i2c_client *client, u8 reg, bool update)
{
    struct pmbus_data *data = i2c_get_clientdata(client);
    struct pmbus_sensor *sensor;

    for (sensor = data->sensors; sensor; sensor = sensor->next)
        if (sensor->reg == reg)
            sensor->update = update;
}
```

#### 传感器数据更新

```c
static void pmbus_update_sensor_data(struct i2c_client *client,
                                     struct pmbus_sensor *sensor)
{
    // 如果数据无效或需要更新，则重新读取
    if (sensor->data < 0 || sensor->update)
        sensor->data = _pmbus_read_word_data(client, sensor->page,
                                             sensor->phase, sensor->reg);
}
```

---

### 调节器框架集成

PMBus核心提供了与Linux调节器框架的集成：

```c
const struct regulator_ops pmbus_regulator_ops = {
    .enable = pmbus_regulator_enable,
    .disable = pmbus_regulator_disable,
    .is_enabled = pmbus_regulator_is_enabled,
    .get_error_flags = pmbus_regulator_get_error_flags,
    .get_status = pmbus_regulator_get_status,
    .get_voltage = pmbus_regulator_get_voltage,
    .set_voltage = pmbus_regulator_set_voltage,
    .list_voltage = pmbus_regulator_list_voltage,
};
```

#### 调节器描述宏

```c
// 带电压步进的调节器
#define PMBUS_REGULATOR_STEP(_name, _id, _voltages, _step, _min_uV) \
    [_id] = {                                                       \
        .name = (_name # _id),                                      \
        .id = (_id),                                                \
        .of_match = of_match_ptr(_name # _id),                      \
        .regulators_node = of_match_ptr("regulators"),              \
        .ops = &pmbus_regulator_ops,                                \
        .type = REGULATOR_VOLTAGE,                                  \
        .owner = THIS_MODULE,                                       \
        .n_voltages = _voltages,                                    \
        .uV_step = _step,                                           \
        .min_uV = _min_uV,                                          \
    }

// 简单调节器（无电压步进）
#define PMBUS_REGULATOR(_name, _id)   PMBUS_REGULATOR_STEP(_name, _id, 0, 0, 0)
```

#### 使用示例

```c
static const struct regulator_desc ltc2978_reg_desc[] = {
    PMBUS_REGULATOR("vout", 0),
    PMBUS_REGULATOR("vout", 1),
    PMBUS_REGULATOR("vout", 2),
    PMBUS_REGULATOR("vout", 3),
    // ...
};

static struct pmbus_driver_info ltc2978_info = {
    .pages = 8,
    .num_regulators = ARRAY_SIZE(ltc2978_reg_desc),
    .reg_desc = ltc2978_reg_desc,
    // ...
};
```

---

## 泳道图：设备初始化流程

```
┌─────────────┐ ┌──────────────┐ ┌─────────────────┐ ┌──────────────┐ ┌─────────────┐
│ I2C Core    │ │ 设备驱动     │ │ PMBus Core      │ │ HWMON Core   │ │ 用户空间    │
│             │ │(如ltc2978.c) │ │(pmbus_core.c)   │ │ (hwmon.c)    │ │             │
└──────┬──────┘ └──────┬───────┘ └────────┬────────┘ └──────┬───────┘ └──────┬──────┘
       │               │                  │                 │                │
       │ i2c_add_driver│                  │                 │                │
       │<──────────────│                  │                 │                │
       │               │                  │                 │                │
  ┌────┴────┐          │                  │                 │                │
  │检测设备  │          │                  │                 │                │
  │匹配ID   │          │                  │                 │                │
  └────┬────┘          │                  │                 │                │
       │               │                  │                 │                │
       │ probe()回调   │                  │                 │                │
       │──────────────>│                  │                 │                │
       │               │                  │                 │                │
       │               │ 创建pmbus_driver_info              │                │
       │               │ 配置m/b/R系数    │                 │                │
       │               │ 设置func[]功能位 │                 │                │
       │               │                  │                 │                │
       │               │ pmbus_do_probe() │                 │                │
       │               │─────────────────>│                 │                │
       │               │                  │                 │                │
       │               │                  │  ┌─────────────────────────────┐│
       │               │                  │  │ pmbus_init_common()         ││
       │               │                  │  │ • 检查PEC支持               ││
       │               │                  │  │ • 检查STATUS_WORD支持       ││
       │               │                  │  │ • 检查写保护状态             ││
       │               │                  │  │ • 调用identify()回调        ││
       │               │                  │  │ • 识别VOUT_MODE             ││
       │               │                  │  └─────────────────────────────┘│
       │               │                  │                 │                │
       │               │                  │  ┌─────────────────────────────┐│
       │               │                  │  │ pmbus_find_attributes()     ││
       │               │                  │  │ • 检测支持的寄存器          ││
       │               │                  │  │ • 创建电压传感器属性        ││
       │               │                  │  │ • 创建电流传感器属性        ││
       │               │                  │  │ • 创建功率传感器属性        ││
       │               │                  │  │ • 创建温度传感器属性        ││
       │               │                  │  │ • 创建风扇属性              ││
       │               │                  │  │ • 创建告警属性              ││
       │               │                  │  └─────────────────────────────┘│
       │               │                  │                 │                │
       │               │                  │ devm_hwmon_device_register_with_groups()
       │               │                  │────────────────>│                │
       │               │                  │                 │                │
       │               │                  │                 │  ┌────────────────────┐
       │               │                  │                 │  │创建hwmon设备       │
       │               │                  │                 │  │创建sysfs目录       │
       │               │                  │                 │  │/sys/class/hwmon/   │
       │               │                  │                 │  │hwmonX/             │
       │               │                  │                 │  └────────────────────┘
       │               │                  │                 │                │
       │               │                  │ 返回hwmon_dev   │                │
       │               │                  │<────────────────│                │
       │               │                  │                 │                │
       │               │                  │  ┌─────────────────────────────┐│
       │               │                  │  │ pmbus_regulator_register()  ││
       │               │                  │  │ • 注册电压调节器            ││
       │               │                  │  │ (如果支持)                 ││
       │               │                  │  └─────────────────────────────┘│
       │               │                  │                 │                │
       │               │                  │  ┌─────────────────────────────┐│
       │               │                  │  │ pmbus_init_debugfs()        ││
       │               │                  │  │ • 创建debugfs条目           ││
       │               │                  │  │ • mfr_id, mfr_model等      ││
       │               │                  │  └─────────────────────────────┘│
       │               │                  │                 │                │
       │               │ 返回成功         │                 │                │
       │               │<─────────────────│                 │                │
       │               │                  │                 │                │
       │ 返回成功      │                  │                 │                │
       │<──────────────│                  │                 │                │
       │               │                  │                 │                │
       │               │                  │                 │                │
       │               │                  │                 │  sysfs可访问    │
       │               │                  │                 │<───────────────│
       │               │                  │                 │                │
       ▼               ▼                  ▼                 ▼                ▼
```

---

## 泳道图：数据读取流程

```
┌─────────────┐ ┌──────────────┐ ┌─────────────────┐ ┌──────────────┐ ┌─────────────┐
│ 用户空间    │ │ HWMON sysfs  │ │ PMBus Core      │ │ 设备驱动     │ │ I2C/SMBus   │
│             │ │ 属性处理     │ │(pmbus_core.c)   │ │(如ltc2978.c) │ │             │
└──────┬──────┘ └──────┬───────┘ └────────┬────────┘ └──────┬───────┘ └──────┬──────┘
       │               │                  │                 │                │
       │ cat in1_input │                  │                 │                │
       │───────────────>                  │                 │                │
       │               │                  │                 │                │
       │               │ pmbus_show_sensor()               │                │
       │               │─────────────────>│                 │                │
       │               │                  │                 │                │
       │               │                  │  ┌────────────────────────────┐ │
       │               │                  │  │ mutex_lock(&update_lock)   │ │
       │               │                  │  └────────────────────────────┘ │
       │               │                  │                 │                │
       │               │                  │ pmbus_update_sensor_data()      │
       │               │                  │──────────────────────────────────│
       │               │                  │                 │                │
       │               │                  │ _pmbus_read_word_data()         │
       │               │                  │─────────────────>               │
       │               │                  │                 │                │
       │               │                  │                 │ info->read_word_data()
       │               │                  │                 │ (如果定义)     │
       │               │                  │                 │──────────────>│
       │               │                  │                 │                │
       │               │                  │                 │ 处理厂商特定  │
       │               │                  │                 │ 寄存器映射    │
       │               │                  │                 │                │
       │               │                  │                 │<──────────────│
       │               │                  │                 │                │
       │               │                  │ 或者 pmbus_read_word_data()     │
       │               │                  │─────────────────────────────────>│
       │               │                  │                 │                │
       │               │                  │                 │ pmbus_set_page()
       │               │                  │                 │───────────────>│
       │               │                  │                 │                │
       │               │                  │                 │ i2c_smbus_write_byte_data
       │               │                  │                 │ (PMBUS_PAGE, page)
       │               │                  │                 │<───────────────│
       │               │                  │                 │                │
       │               │                  │                 │ i2c_smbus_read_word_data
       │               │                  │                 │ (reg)          │
       │               │                  │                 │───────────────>│
       │               │                  │                 │                │
       │               │                  │                 │    原始数据    │
       │               │                  │                 │<───────────────│
       │               │                  │                 │                │
       │               │                  │<─────────────────────────────────│
       │               │                  │                 │                │
       │               │                  │  ┌────────────────────────────┐ │
       │               │                  │  │ pmbus_reg2data()           │ │
       │               │                  │  │ 根据format转换:            │ │
       │               │                  │  │ • linear → 毫单位          │ │
       │               │                  │  │ • direct → 使用m,b,R系数   │ │
       │               │                  │  │ • vid → 电压表查找         │ │
       │               │                  │  │ • ieee754 → 浮点转换       │ │
       │               │                  │  └────────────────────────────┘ │
       │               │                  │                 │                │
       │               │                  │  ┌────────────────────────────┐ │
       │               │                  │  │ mutex_unlock(&update_lock) │ │
       │               │                  │  └────────────────────────────┘ │
       │               │                  │                 │                │
       │               │ 返回转换后的值   │                 │                │
       │               │<─────────────────│                 │                │
       │               │                  │                 │                │
       │ sysfs_emit()  │                  │                 │                │
       │ 格式化输出    │                  │                 │                │
       │<──────────────│                  │                 │                │
       │               │                  │                 │                │
       │ "12500"       │                  │                 │                │
       │ (12.5V)       │                  │                 │                │
       ▼               ▼                  ▼                 ▼                ▼
```

---

## 关键数据结构

### PMBus驱动信息结构

```c
struct pmbus_driver_info {
    int pages;                          // 页面数量
    u8 phases[PMBUS_PAGES];            // 每页相位数
    enum pmbus_data_format format[PSC_NUM_CLASSES];  // 数据格式
    enum vrm_version vrm_version[PMBUS_PAGES];       // VRM版本
    
    // Direct模式系数
    int m[PSC_NUM_CLASSES];            // 尾数
    int b[PSC_NUM_CLASSES];            // 偏移
    int R[PSC_NUM_CLASSES];            // 指数
    
    u32 func[PMBUS_PAGES];             // 每页功能位
    u32 pfunc[PMBUS_PHASES];           // 每相功能位
    
    // 可选回调函数
    int (*read_byte_data)(struct i2c_client *client, int page, int reg);
    int (*read_word_data)(struct i2c_client *client, int page, int phase, int reg);
    int (*write_word_data)(struct i2c_client *client, int page, int reg, u16 word);
    int (*write_byte)(struct i2c_client *client, int page, u8 value);
    int (*identify)(struct i2c_client *client, struct pmbus_driver_info *info);
    
    // 调节器支持
    int num_regulators;
    const struct regulator_desc *reg_desc;
    
    // 自定义属性组
    const struct attribute_group **groups;
    
    // 访问延迟
    int access_delay;                  // 微秒
    int write_delay;                   // 微秒
};
```

### 功能位定义

```c
#define PMBUS_HAVE_VIN          BIT(0)   // 有输入电压
#define PMBUS_HAVE_VCAP         BIT(1)   // 有电容电压
#define PMBUS_HAVE_VOUT         BIT(2)   // 有输出电压
#define PMBUS_HAVE_IIN          BIT(3)   // 有输入电流
#define PMBUS_HAVE_IOUT         BIT(4)   // 有输出电流
#define PMBUS_HAVE_PIN          BIT(5)   // 有输入功率
#define PMBUS_HAVE_POUT         BIT(6)   // 有输出功率
#define PMBUS_HAVE_FAN12        BIT(7)   // 有风扇1/2
#define PMBUS_HAVE_FAN34        BIT(8)   // 有风扇3/4
#define PMBUS_HAVE_TEMP         BIT(9)   // 有温度1
#define PMBUS_HAVE_TEMP2        BIT(10)  // 有温度2
#define PMBUS_HAVE_TEMP3        BIT(11)  // 有温度3
#define PMBUS_HAVE_STATUS_VOUT  BIT(12)  // 有VOUT状态
#define PMBUS_HAVE_STATUS_IOUT  BIT(13)  // 有IOUT状态
#define PMBUS_HAVE_STATUS_INPUT BIT(14)  // 有INPUT状态
#define PMBUS_HAVE_STATUS_TEMP  BIT(15)  // 有TEMP状态
#define PMBUS_HAVE_STATUS_FAN12 BIT(16)  // 有FAN12状态
#define PMBUS_HAVE_STATUS_FAN34 BIT(17)  // 有FAN34状态
#define PMBUS_HAVE_VMON         BIT(18)  // 有电压监控
#define PMBUS_HAVE_STATUS_VMON  BIT(19)  // 有VMON状态
#define PMBUS_HAVE_PWM12        BIT(20)  // 有PWM1/2
#define PMBUS_HAVE_PWM34        BIT(21)  // 有PWM3/4
#define PMBUS_HAVE_SAMPLES      BIT(22)  // 有采样配置
```

### 传感器类

```c
enum pmbus_sensor_classes {
    PSC_VOLTAGE_IN = 0,    // 输入电压
    PSC_VOLTAGE_OUT,       // 输出电压
    PSC_CURRENT_IN,        // 输入电流
    PSC_CURRENT_OUT,       // 输出电流
    PSC_POWER,             // 功率
    PSC_TEMPERATURE,       // 温度
    PSC_FAN,               // 风扇
    PSC_PWM,               // PWM
    PSC_NUM_CLASSES
};
```

---

## Sysfs接口

### 目录结构
```
/sys/class/hwmon/hwmonX/
├── name                    # 设备名称
├── in1_input              # 输入电压 (mV)
├── in1_min                # 电压下限
├── in1_max                # 电压上限
├── in1_lcrit              # 电压临界低值
├── in1_crit               # 电压临界高值
├── in1_min_alarm          # 最小值告警
├── in1_max_alarm          # 最大值告警
├── in1_label              # 标签 (如 "vin", "vout1")
├── curr1_input            # 电流 (mA)
├── curr1_max              # 电流上限
├── curr1_crit             # 电流临界值
├── power1_input           # 功率 (µW)
├── temp1_input            # 温度 (m°C)
├── temp1_max              # 温度上限
├── temp1_crit             # 温度临界值
├── fan1_input             # 风扇转速 (RPM)
├── fan1_target            # 目标转速
├── pwm1                   # PWM占空比 (0-255)
└── pwm1_enable            # PWM使能模式
```

### 命名规范
| 类型 | 格式 | 起始编号 | 单位 |
|------|------|----------|------|
| 电压 | inX_* | 0 | 毫伏 (mV) |
| 电流 | currX_* | 1 | 毫安 (mA) |
| 功率 | powerX_* | 1 | 微瓦 (µW) |
| 温度 | tempX_* | 1 | 毫摄氏度 (m°C) |
| 风扇 | fanX_* | 1 | RPM |
| PWM | pwmX_* | 1 | 0-255 |

---

## PMBus设备驱动开发指南

### 驱动开发流程概述

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PMBus驱动开发流程                                    │
└─────────────────────────────────────────────────────────────────────────────┘

1. 分析硬件
   ├── 阅读芯片数据手册
   ├── 确定PMBus命令支持情况
   ├── 确定数据格式(Linear/Direct/VID/IEEE754)
   └── 识别厂商特定寄存器

2. 创建pmbus_driver_info结构
   ├── 设置pages (输出通道数)
   ├── 设置phases (每页相位数)
   ├── 配置format[] (各传感器类数据格式)
   ├── 配置m[], b[], R[] (Direct格式系数)
   ├── 设置func[] (功能位掩码)
   └── 设置pfunc[] (相位功能位掩码)

3. 实现可选回调函数
   ├── identify() - 动态识别设备能力
   ├── read_word_data() - 自定义读操作
   ├── write_word_data() - 自定义写操作
   ├── read_byte_data() - 自定义字节读
   └── write_byte() - 自定义字节写

4. 实现I2C驱动框架
   ├── 定义i2c_device_id表
   ├── 实现probe函数(调用pmbus_do_probe)
   └── 注册i2c_driver

5. 测试与调试
   ├── 使用i2c-tools验证通信
   ├── 检查sysfs属性
   └── 验证debugfs信息
```

---

### 最小驱动示例

```c
// SPDX-License-Identifier: GPL-2.0-or-later
/*
 * My PMBus Device Driver
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/err.h>
#include <linux/i2c.h>
#include <linux/pmbus.h>
#include "pmbus.h"

static struct pmbus_driver_info my_device_info = {
    .pages = 1,
    .format[PSC_VOLTAGE_IN] = linear,
    .format[PSC_VOLTAGE_OUT] = linear,
    .format[PSC_CURRENT_IN] = linear,
    .format[PSC_CURRENT_OUT] = linear,
    .format[PSC_POWER] = linear,
    .format[PSC_TEMPERATURE] = linear,
    .func[0] = PMBUS_HAVE_VIN | PMBUS_HAVE_VOUT 
             | PMBUS_HAVE_IIN | PMBUS_HAVE_IOUT
             | PMBUS_HAVE_PIN | PMBUS_HAVE_POUT
             | PMBUS_HAVE_TEMP | PMBUS_HAVE_TEMP2
             | PMBUS_HAVE_STATUS_VOUT | PMBUS_HAVE_STATUS_IOUT
             | PMBUS_HAVE_STATUS_INPUT | PMBUS_HAVE_STATUS_TEMP,
};

static int my_device_probe(struct i2c_client *client)
{
    return pmbus_do_probe(client, &my_device_info);
}

static const struct i2c_device_id my_device_id[] = {
    {"my_pmbus_device", 0},
    {}
};
MODULE_DEVICE_TABLE(i2c, my_device_id);

static const struct of_device_id my_device_of_match[] = {
    { .compatible = "vendor,my-pmbus-device" },
    {}
};
MODULE_DEVICE_TABLE(of, my_device_of_match);

static struct i2c_driver my_device_driver = {
    .driver = {
        .name = "my_pmbus_device",
        .of_match_table = my_device_of_match,
    },
    .probe = my_device_probe,
    .id_table = my_device_id,
};

module_i2c_driver(my_device_driver);

MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("PMBus driver for My Device");
MODULE_LICENSE("GPL");
MODULE_IMPORT_NS(PMBUS);
```

---

### 回调函数详解

#### pmbus_driver_info回调函数

```c
struct pmbus_driver_info {
    // ... 其他字段

    /*
     * 可选回调函数：映射厂商特定寄存器到PMBus标准寄存器
     * 返回值:
     *   >= 0: 成功，返回寄存器值
     *   -ENODATA: 无厂商特定寄存器，请求核心层处理标准PMBus寄存器
     *   其他负值: 错误，寄存器不存在
     */
    int (*read_byte_data)(struct i2c_client *client, int page, int reg);
    int (*read_word_data)(struct i2c_client *client, int page, int phase, int reg);
    int (*write_byte_data)(struct i2c_client *client, int page, int reg, u8 byte);
    int (*write_word_data)(struct i2c_client *client, int page, int reg, u16 word);
    int (*write_byte)(struct i2c_client *client, int page, u8 value);

    /*
     * identify: 确定支持的PMBus功能
     * 仅当驱动支持多种芯片变体且功能不能预先确定时需要
     */
    int (*identify)(struct i2c_client *client, struct pmbus_driver_info *info);
};
```

#### read_word_data回调实现模式

```c
/*
 * 厂商特定寄存器定义 (通常在0xD0-0xFF范围)
 */
#define MFR_VOUT_PEAK       0xD0
#define MFR_VIN_PEAK        0xD1
#define MFR_TEMP_PEAK       0xD2
#define MFR_IOUT_PEAK       0xD3
#define MFR_CLEAR_PEAKS     0xD4
#define MFR_SPECIAL_CONFIG  0xD5

static int my_device_read_word_data(struct i2c_client *client,
                                    int page, int phase, int reg)
{
    int ret;

    switch (reg) {
    /*
     * 历史记录最大值 - 映射到厂商特定峰值寄存器
     */
    case PMBUS_VIRT_READ_VOUT_MAX:
        ret = pmbus_read_word_data(client, page, phase, MFR_VOUT_PEAK);
        break;

    case PMBUS_VIRT_READ_VIN_MAX:
        ret = pmbus_read_word_data(client, page, phase, MFR_VIN_PEAK);
        break;

    case PMBUS_VIRT_READ_TEMP_MAX:
        ret = pmbus_read_word_data(client, page, phase, MFR_TEMP_PEAK);
        break;

    case PMBUS_VIRT_READ_IOUT_MAX:
        ret = pmbus_read_word_data(client, page, phase, MFR_IOUT_PEAK);
        break;

    /*
     * 历史重置寄存器 - 返回0表示支持此功能
     * 核心层会尝试读取这些寄存器来检测是否支持
     */
    case PMBUS_VIRT_RESET_VOUT_HISTORY:
    case PMBUS_VIRT_RESET_VIN_HISTORY:
    case PMBUS_VIRT_RESET_TEMP_HISTORY:
    case PMBUS_VIRT_RESET_IOUT_HISTORY:
        ret = 0;
        break;

    /*
     * PWM控制 (如果设备支持)
     */
    case PMBUS_VIRT_PWM_1:
        ret = pmbus_read_word_data(client, page, phase, 
                                   PMBUS_FAN_COMMAND_1);
        if (ret >= 0) {
            // 转换为0-255的PWM值
            ret = (ret * 255 + 50) / 100;
        }
        break;

    case PMBUS_VIRT_PWM_ENABLE_1:
        // 返回使能状态
        ret = 1;  // 假设始终使能
        break;

    /*
     * 采样数配置 (如果设备支持)
     */
    case PMBUS_VIRT_SAMPLES:
        ret = pmbus_read_word_data(client, page, phase, MFR_SPECIAL_CONFIG);
        if (ret >= 0) {
            // 提取采样数字段
            ret = (ret >> 4) & 0x0F;
        }
        break;

    default:
        /*
         * 返回-ENODATA让核心层处理:
         * 1. 标准PMBus寄存器
         * 2. 核心内置的虚拟寄存器处理
         */
        ret = -ENODATA;
        break;
    }

    return ret;
}
```

#### write_word_data回调实现模式

```c
static int my_device_write_word_data(struct i2c_client *client,
                                     int page, int reg, u16 word)
{
    int ret;

    switch (reg) {
    /*
     * 历史记录重置
     */
    case PMBUS_VIRT_RESET_VOUT_HISTORY:
    case PMBUS_VIRT_RESET_VIN_HISTORY:
    case PMBUS_VIRT_RESET_TEMP_HISTORY:
    case PMBUS_VIRT_RESET_IOUT_HISTORY:
        // 写入峰值清除命令
        ret = pmbus_write_byte(client, page, MFR_CLEAR_PEAKS);
        break;

    /*
     * PWM控制
     */
    case PMBUS_VIRT_PWM_1:
        // 将0-255转换为百分比
        word = (word * 100 + 127) / 255;
        ret = pmbus_write_word_data(client, page, 
                                    PMBUS_FAN_COMMAND_1, word);
        break;

    /*
     * 采样数配置
     */
    case PMBUS_VIRT_SAMPLES:
        ret = pmbus_read_word_data(client, page, 0xff, MFR_SPECIAL_CONFIG);
        if (ret >= 0) {
            // 更新采样数字段
            ret = (ret & 0xFF0F) | ((word & 0x0F) << 4);
            ret = pmbus_write_word_data(client, page, 
                                        MFR_SPECIAL_CONFIG, ret);
        }
        break;

    default:
        ret = -ENODATA;
        break;
    }

    return ret;
}
```

#### identify回调实现模式

```c
/*
 * identify回调在pmbus_init_common()中被调用
 * 用于动态确定设备能力和配置
 */
static int my_device_identify(struct i2c_client *client,
                              struct pmbus_driver_info *info)
{
    int ret, chip_id, config;
    u8 buf[I2C_SMBUS_BLOCK_MAX + 1];

    /*
     * 读取芯片ID来确定具体型号
     */
    ret = i2c_smbus_read_word_data(client, PMBUS_IC_DEVICE_ID);
    if (ret < 0)
        return ret;
    chip_id = ret;

    /*
     * 读取配置寄存器
     */
    ret = i2c_smbus_read_word_data(client, MFR_SPECIAL_CONFIG);
    if (ret < 0)
        return ret;
    config = ret;

    /*
     * 根据芯片ID配置不同功能
     */
    switch (chip_id) {
    case 0x0001:  // 单输出型号
        info->pages = 1;
        info->func[0] = PMBUS_HAVE_VIN | PMBUS_HAVE_VOUT
                      | PMBUS_HAVE_IOUT | PMBUS_HAVE_TEMP
                      | PMBUS_HAVE_STATUS_VOUT | PMBUS_HAVE_STATUS_IOUT
                      | PMBUS_HAVE_STATUS_INPUT | PMBUS_HAVE_STATUS_TEMP;
        break;

    case 0x0002:  // 双输出型号
        info->pages = 2;
        info->func[0] = PMBUS_HAVE_VIN | PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT
                      | PMBUS_HAVE_TEMP | PMBUS_HAVE_STATUS_VOUT
                      | PMBUS_HAVE_STATUS_IOUT | PMBUS_HAVE_STATUS_INPUT
                      | PMBUS_HAVE_STATUS_TEMP;
        info->func[1] = PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT
                      | PMBUS_HAVE_TEMP | PMBUS_HAVE_STATUS_VOUT
                      | PMBUS_HAVE_STATUS_IOUT | PMBUS_HAVE_STATUS_TEMP;
        break;

    case 0x0003:  // 四相型号
        info->pages = 1;
        info->phases[0] = 4;
        info->func[0] = PMBUS_HAVE_VIN | PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT
                      | PMBUS_HAVE_TEMP | PMBUS_HAVE_STATUS_VOUT
                      | PMBUS_HAVE_STATUS_IOUT | PMBUS_HAVE_STATUS_INPUT;
        // 每相的功能
        info->pfunc[0] = PMBUS_HAVE_IOUT;
        info->pfunc[1] = PMBUS_HAVE_IOUT;
        info->pfunc[2] = PMBUS_HAVE_IOUT;
        info->pfunc[3] = PMBUS_HAVE_IOUT;
        break;

    default:
        dev_err(&client->dev, "Unsupported chip ID: 0x%04x\n", chip_id);
        return -ENODEV;
    }

    /*
     * 根据配置寄存器调整功能
     */
    if (config & BIT(0)) {
        // 有风扇控制功能
        info->func[0] |= PMBUS_HAVE_FAN12 | PMBUS_HAVE_STATUS_FAN12;
    }

    if (config & BIT(1)) {
        // 有PWM控制功能
        info->func[0] |= PMBUS_HAVE_PWM12;
    }

    if (config & BIT(2)) {
        // 支持采样配置
        info->func[0] |= PMBUS_HAVE_SAMPLES;
    }

    /*
     * 读取VOUT_MODE来确定数据格式
     */
    ret = pmbus_read_byte_data(client, 0, PMBUS_VOUT_MODE);
    if (ret >= 0) {
        switch (ret >> 5) {
        case 0:  // Linear模式
            info->format[PSC_VOLTAGE_OUT] = linear;
            break;
        case 1:  // VID模式
            info->format[PSC_VOLTAGE_OUT] = vid;
            info->vrm_version[0] = vr13;
            break;
        case 2:  // Direct模式
            info->format[PSC_VOLTAGE_OUT] = direct;
            // 需要设置m, b, R系数
            break;
        case 3:  // IEEE754半精度
            info->format[PSC_VOLTAGE_OUT] = ieee754;
            break;
        }
    }

    return 0;
}
```

---

### Direct模式系数配置详解

```c
/*
 * Direct格式转换公式:
 * 
 * 寄存器值 → 实际值:  X = (1/m) × (Y × 10^(-R) - b)
 * 实际值 → 寄存器值:  Y = (m × X + b) × 10^R
 *
 * 其中:
 *   X = 实际物理量值 (毫伏, 毫安, 毫瓦等)
 *   Y = 寄存器原始值 (16位有符号整数)
 *   m = 斜率系数 (mantissa)
 *   b = 偏移系数 (offset)
 *   R = 缩放指数
 *
 * 示例: 假设数据手册给出:
 *   电压: m=8, b=0, R=-2 (分辨率: 12.5mV)
 *   电流: m=1, b=0, R=-1 (分辨率: 100mA)
 *   温度: m=1, b=273, R=0 (开尔文到摄氏转换)
 */

static struct pmbus_driver_info my_direct_device_info = {
    .pages = 1,
    
    // 数据格式
    .format[PSC_VOLTAGE_IN] = direct,
    .format[PSC_VOLTAGE_OUT] = direct,
    .format[PSC_CURRENT_IN] = direct,
    .format[PSC_CURRENT_OUT] = direct,
    .format[PSC_POWER] = direct,
    .format[PSC_TEMPERATURE] = direct,
    
    // 输入电压系数: 1mV分辨率
    // X = Y × 10^(-3) / 1 = Y mV
    .m[PSC_VOLTAGE_IN] = 1,
    .b[PSC_VOLTAGE_IN] = 0,
    .R[PSC_VOLTAGE_IN] = -3,
    
    // 输出电压系数: 0.5mV分辨率
    // X = Y × 10^(-3) / 2 = Y/2 mV
    .m[PSC_VOLTAGE_OUT] = 2,
    .b[PSC_VOLTAGE_OUT] = 0,
    .R[PSC_VOLTAGE_OUT] = -3,
    
    // 输入电流系数: 10mA分辨率
    // X = Y × 10^(-2) / 1 = Y × 10 mA
    .m[PSC_CURRENT_IN] = 1,
    .b[PSC_CURRENT_IN] = 0,
    .R[PSC_CURRENT_IN] = -2,
    
    // 输出电流系数: 1mA分辨率
    // X = Y × 10^(-3) / 1 = Y mA
    .m[PSC_CURRENT_OUT] = 1,
    .b[PSC_CURRENT_OUT] = 0,
    .R[PSC_CURRENT_OUT] = -3,
    
    // 功率系数: 10mW分辨率
    // 注意: 核心层会自动缩放到微瓦
    .m[PSC_POWER] = 1,
    .b[PSC_POWER] = 0,
    .R[PSC_POWER] = -2,
    
    // 温度系数: 0.1°C分辨率
    // X = Y × 10^(-1) / 1 = Y/10 °C
    .m[PSC_TEMPERATURE] = 1,
    .b[PSC_TEMPERATURE] = 0,
    .R[PSC_TEMPERATURE] = -1,
    
    .func[0] = PMBUS_HAVE_VIN | PMBUS_HAVE_VOUT
             | PMBUS_HAVE_IIN | PMBUS_HAVE_IOUT
             | PMBUS_HAVE_PIN | PMBUS_HAVE_POUT
             | PMBUS_HAVE_TEMP,
};
```

#### 使用COEFFICIENTS命令动态获取系数

```c
/*
 * 某些PMBus设备支持COEFFICIENTS命令(0x30)
 * 可以在运行时查询m, b, R系数
 */
static int my_device_probe(struct i2c_client *client)
{
    struct pmbus_driver_info *info;
    
    info = devm_kmemdup(&client->dev, &my_device_info,
                        sizeof(*info), GFP_KERNEL);
    if (!info)
        return -ENOMEM;
    
    // 使用平台数据标志启用系数命令支持
    // pmbus_core会自动调用COEFFICIENTS命令读取系数
    // 需要在platform_data中设置PMBUS_USE_COEFFICIENTS_CMD标志
    
    return pmbus_do_probe(client, info);
}
```

---

### 多设备变体支持模式

```c
/*
 * 支持多个设备变体的驱动架构
 */

/* 私有数据结构 */
struct my_device_data {
    int chip_id;
    u16 mfr_config;
    s16 temp_offset;
    // 其他设备特定数据
};

/* 获取私有数据的辅助宏 */
#define to_my_device_data(info) \
    container_of(info, struct my_device_data, info)

/* 设备变体枚举 */
enum my_device_type {
    MY_DEVICE_A = 0,
    MY_DEVICE_B,
    MY_DEVICE_C,
    MY_DEVICE_D,
};

/* 各变体的基础配置 */
static const struct pmbus_driver_info my_device_info[] = {
    [MY_DEVICE_A] = {
        .pages = 1,
        .format[PSC_VOLTAGE_OUT] = linear,
        .format[PSC_CURRENT_OUT] = linear,
        .format[PSC_TEMPERATURE] = linear,
        .func[0] = PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT | PMBUS_HAVE_TEMP
                 | PMBUS_HAVE_STATUS_VOUT | PMBUS_HAVE_STATUS_IOUT
                 | PMBUS_HAVE_STATUS_TEMP,
    },
    [MY_DEVICE_B] = {
        .pages = 2,
        .format[PSC_VOLTAGE_OUT] = direct,
        .format[PSC_CURRENT_OUT] = direct,
        .format[PSC_TEMPERATURE] = linear,
        .m[PSC_VOLTAGE_OUT] = 8,
        .b[PSC_VOLTAGE_OUT] = 0,
        .R[PSC_VOLTAGE_OUT] = -2,
        .m[PSC_CURRENT_OUT] = 1,
        .b[PSC_CURRENT_OUT] = 0,
        .R[PSC_CURRENT_OUT] = -2,
        .func[0] = PMBUS_HAVE_VIN | PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT
                 | PMBUS_HAVE_TEMP | PMBUS_HAVE_STATUS_VOUT
                 | PMBUS_HAVE_STATUS_IOUT | PMBUS_HAVE_STATUS_INPUT
                 | PMBUS_HAVE_STATUS_TEMP,
        .func[1] = PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT
                 | PMBUS_HAVE_STATUS_VOUT | PMBUS_HAVE_STATUS_IOUT,
    },
    // ... 更多变体
};

static int my_device_probe(struct i2c_client *client)
{
    const struct i2c_device_id *id = i2c_match_id(my_device_id, client);
    struct pmbus_driver_info *info;
    struct my_device_data *data;
    int ret;

    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    // 复制基础配置
    info = devm_kmemdup(&client->dev, 
                        &my_device_info[id->driver_data],
                        sizeof(*info), GFP_KERNEL);
    if (!info)
        return -ENOMEM;

    // 设置回调函数
    info->read_word_data = my_device_read_word_data;
    info->write_word_data = my_device_write_word_data;
    info->identify = my_device_identify;

    // 保存私有数据
    data->chip_id = id->driver_data;
    
    return pmbus_do_probe(client, info);
}

static const struct i2c_device_id my_device_id[] = {
    {"my_device_a", MY_DEVICE_A},
    {"my_device_b", MY_DEVICE_B},
    {"my_device_c", MY_DEVICE_C},
    {"my_device_d", MY_DEVICE_D},
    {}
};
```

---

### 添加调节器支持

```c
#include <linux/regulator/driver.h>

/* 定义调节器描述符 */
static const struct regulator_desc my_device_reg_desc[] = {
    PMBUS_REGULATOR("vout", 0),
    PMBUS_REGULATOR("vout", 1),
};

/* 带电压配置的调节器 */
static const struct regulator_desc my_device_reg_desc_with_voltage[] = {
    PMBUS_REGULATOR_STEP("vout", 0, 256, 5000, 500000),  // 0.5V-1.78V, 5mV步进
    PMBUS_REGULATOR_STEP("vout", 1, 256, 5000, 500000),
};

static struct pmbus_driver_info my_regulator_device_info = {
    .pages = 2,
    .format[PSC_VOLTAGE_OUT] = linear,
    .format[PSC_CURRENT_OUT] = linear,
    .func[0] = PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT | PMBUS_HAVE_STATUS_VOUT,
    .func[1] = PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT | PMBUS_HAVE_STATUS_VOUT,
    
    // 调节器配置
    .num_regulators = ARRAY_SIZE(my_device_reg_desc),
    .reg_desc = my_device_reg_desc,
};
```

---

### 设备轮询与等待模式

某些设备在特定操作后需要等待：

```c
/*
 * 自定义轮询等待函数
 * 适用于需要等待设备完成操作的场景
 */
#define MY_DEVICE_BUSY_BIT      BIT(7)
#define MY_DEVICE_MAX_WAIT_MS   100

static int my_device_wait_ready(struct i2c_client *client)
{
    int ret, tries = MY_DEVICE_MAX_WAIT_MS;

    while (tries--) {
        ret = pmbus_read_byte_data(client, 0, PMBUS_STATUS_CML);
        if (ret < 0)
            return ret;
        
        if (!(ret & MY_DEVICE_BUSY_BIT))
            return 0;
        
        usleep_range(1000, 2000);  // 等待1-2ms
    }

    return -ETIMEDOUT;
}

static int my_device_write_word_data(struct i2c_client *client,
                                     int page, int reg, u16 word)
{
    int ret;

    // 对于某些命令需要等待设备就绪
    if (reg == PMBUS_VOUT_COMMAND || reg == PMBUS_OPERATION) {
        ret = my_device_wait_ready(client);
        if (ret)
            return ret;
    }

    // 执行写操作...
    
    // 写后等待
    if (reg == PMBUS_VOUT_COMMAND) {
        ret = my_device_wait_ready(client);
        if (ret)
            return ret;
    }

    return -ENODATA;  // 让核心层处理
}
```

---

### 添加自定义sysfs属性

```c
/*
 * 为设备添加核心层不提供的自定义sysfs属性
 */

static ssize_t my_special_attr_show(struct device *dev,
                                    struct device_attribute *attr,
                                    char *buf)
{
    struct i2c_client *client = to_i2c_client(dev->parent);
    int ret;

    ret = pmbus_read_word_data(client, 0, 0xff, MFR_SPECIAL_CONFIG);
    if (ret < 0)
        return ret;

    return sysfs_emit(buf, "%d\n", ret);
}

static ssize_t my_special_attr_store(struct device *dev,
                                     struct device_attribute *attr,
                                     const char *buf, size_t count)
{
    struct i2c_client *client = to_i2c_client(dev->parent);
    unsigned long val;
    int ret;

    ret = kstrtoul(buf, 10, &val);
    if (ret)
        return ret;

    ret = pmbus_write_word_data(client, 0, MFR_SPECIAL_CONFIG, val);
    if (ret < 0)
        return ret;

    return count;
}

static DEVICE_ATTR_RW(my_special_attr);

static struct attribute *my_device_attrs[] = {
    &dev_attr_my_special_attr.attr,
    NULL
};

static const struct attribute_group my_device_attr_group = {
    .attrs = my_device_attrs,
};

static const struct attribute_group *my_device_attr_groups[] = {
    &my_device_attr_group,
    NULL
};

static struct pmbus_driver_info my_device_info = {
    .pages = 1,
    // ... 其他配置
    .groups = my_device_attr_groups,
};
```

---

### 使用pmbus_lock保护访问

```c
/*
 * 对于需要原子操作多个寄存器的场景
 * 使用pmbus_lock_interruptible/pmbus_unlock保护
 */
static int my_device_atomic_operation(struct i2c_client *client)
{
    int ret, val1, val2;

    ret = pmbus_lock_interruptible(client);
    if (ret)
        return ret;

    // 读取多个相关寄存器
    val1 = pmbus_read_word_data(client, 0, 0xff, PMBUS_READ_VIN);
    if (val1 < 0) {
        pmbus_unlock(client);
        return val1;
    }

    val2 = pmbus_read_word_data(client, 0, 0xff, PMBUS_READ_IIN);
    if (val2 < 0) {
        pmbus_unlock(client);
        return val2;
    }

    // 根据读取值执行某些操作...

    pmbus_unlock(client);

    return 0;
}

---

## 调试与诊断

### DebugFS接口
PMBus核心自动创建debugfs条目：
```
/sys/kernel/debug/pmbus/hwmonX/
├── mfr_id              # 制造商ID
├── mfr_model           # 型号
├── mfr_revision        # 修订版本
├── mfr_location        # 位置
├── mfr_date            # 生产日期
├── mfr_serial          # 序列号
├── status0             # 页面0状态字
├── status0_vout        # 页面0 VOUT状态
├── status0_iout        # 页面0 IOUT状态
├── status0_input       # 页面0输入状态
├── status0_temp        # 页面0温度状态
└── status0_cml         # 页面0 CML状态
```

### 常用调试命令

```bash
# 查看所有hwmon设备
ls /sys/class/hwmon/

# 查看设备详情
cat /sys/class/hwmon/hwmonX/name
cat /sys/class/hwmon/hwmonX/in1_input

# 使用i2c-tools直接访问
i2cdetect -y 1
i2cdump -y 1 0x40
i2cget -y 1 0x40 0x88 w  # 读取READ_VIN

# 查看debugfs信息
cat /sys/kernel/debug/pmbus/hwmon*/mfr_id

# 监控传感器变化
watch -n 1 "cat /sys/class/hwmon/hwmon*/temp*_input"

# 使用sensors工具
sensors
```

---

## 参考资源

### 官方文档
- [Documentation/hwmon/index.rst](Documentation/hwmon/index.rst)
- [Documentation/hwmon/pmbus-core.rst](Documentation/hwmon/pmbus-core.rst)
- [Documentation/hwmon/sysfs-interface.rst](Documentation/hwmon/sysfs-interface.rst)
- [Documentation/hwmon/hwmon-kernel-api.rst](Documentation/hwmon/hwmon-kernel-api.rst)

### 源代码位置
- `drivers/hwmon/hwmon.c` - HWMON核心
- `drivers/hwmon/pmbus/pmbus_core.c` - PMBus核心
- `drivers/hwmon/pmbus/pmbus.c` - 通用PMBus驱动
- `drivers/hwmon/pmbus/pmbus.h` - PMBus内部API
- `include/linux/hwmon.h` - HWMON公共API
- `include/linux/pmbus.h` - PMBus平台数据

### 外部链接
- [PMBus规范 (pmbus.org)](https://pmbus.org/)
- [lm-sensors项目](https://github.com/lm-sensors/lm-sensors)

---

## 总结

Linux HWMON/PMBus子系统提供了一个强大且灵活的硬件监控框架：

1. **分层架构**: HWMON Core → PMBus Core → 设备驱动，职责清晰
2. **标准化接口**: sysfs接口统一，用户空间工具通用
3. **可扩展性强**: 虚拟寄存器机制支持厂商特定功能
4. **易于开发**: 核心处理大部分工作，设备驱动只需配置
5. **功能丰富**: 支持调节器、thermal zone、debugfs等集成
