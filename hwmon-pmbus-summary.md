# Linux Kernel HWMON 与 PMBus 子系统详细总结

## 目录
1. [概述](#概述)
2. [架构层次图](#架构层次图)
3. [模块关系图](#模块关系图)
4. [HWMON核心子系统](#hwmon核心子系统)
5. [PMBus子系统](#pmbus子系统)
6. [泳道图：设备初始化流程](#泳道图设备初始化流程)
7. [泳道图：数据读取流程](#泳道图数据读取流程)
8. [关键数据结构](#关键数据结构)
9. [Sysfs接口](#sysfs接口)
10. [PMBus设备驱动开发指南](#pmbus设备驱动开发指南)

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
4. **多种数据格式**:
   - **Linear**: 5位指数 + 11位尾数
   - **Linear16**: 16位值，指数由VOUT_MODE定义
   - **Direct**: 使用m, b, R系数: X = 1/m × (Y × 10^(-R) - b)
   - **VID**: 电压标识码
   - **IEEE754**: 半精度浮点

### PMBus标准寄存器

| 寄存器 | 地址 | 描述 |
|--------|------|------|
| PAGE | 0x00 | 页面选择 |
| OPERATION | 0x01 | 操作控制 |
| CLEAR_FAULTS | 0x03 | 清除故障 |
| CAPABILITY | 0x19 | 设备能力 |
| VOUT_MODE | 0x20 | 输出电压模式 |
| READ_VIN | 0x88 | 读取输入电压 |
| READ_IIN | 0x89 | 读取输入电流 |
| READ_VOUT | 0x8B | 读取输出电压 |
| READ_IOUT | 0x8C | 读取输出电流 |
| READ_TEMPERATURE_1 | 0x8D | 读取温度1 |
| READ_FAN_SPEED_1 | 0x90 | 读取风扇转速1 |
| READ_POUT | 0x96 | 读取输出功率 |
| READ_PIN | 0x97 | 读取输入功率 |
| STATUS_WORD | 0x79 | 状态字 |
| STATUS_VOUT | 0x7A | 输出电压状态 |
| STATUS_IOUT | 0x7B | 输出电流状态 |
| STATUS_INPUT | 0x7C | 输入状态 |
| STATUS_TEMPERATURE | 0x7D | 温度状态 |

### 虚拟寄存器
PMBus核心定义了虚拟寄存器(从0x100开始)用于支持非标准功能：
- `PMBUS_VIRT_READ_TEMP_AVG` - 平均温度
- `PMBUS_VIRT_READ_VIN_MIN/MAX` - 电压历史记录
- `PMBUS_VIRT_RESET_*_HISTORY` - 重置历史记录
- `PMBUS_VIRT_FAN_TARGET_*` - 风扇目标转速
- `PMBUS_VIRT_PWM_*` - PWM控制

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

### 最小驱动示例

```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/i2c.h>
#include <linux/pmbus.h>
#include "pmbus.h"

static struct pmbus_driver_info my_device_info = {
    .pages = 1,
    .format[PSC_VOLTAGE_IN] = linear,
    .format[PSC_VOLTAGE_OUT] = linear,
    .format[PSC_CURRENT_OUT] = linear,
    .format[PSC_TEMPERATURE] = linear,
    .func[0] = PMBUS_HAVE_VIN | PMBUS_HAVE_VOUT 
             | PMBUS_HAVE_IOUT | PMBUS_HAVE_TEMP
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

static struct i2c_driver my_device_driver = {
    .driver = {
        .name = "my_pmbus_device",
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

### 支持厂商特定命令

```c
static int my_device_read_word_data(struct i2c_client *client,
                                    int page, int phase, int reg)
{
    int ret;
    
    switch (reg) {
    case PMBUS_VIRT_READ_VOUT_MAX:
        // 读取厂商特定寄存器并映射到虚拟寄存器
        ret = pmbus_read_word_data(client, page, phase, 
                                   MFR_VOUT_PEAK);
        break;
    case PMBUS_VIRT_RESET_VOUT_HISTORY:
        // 返回0表示支持此功能
        ret = 0;
        break;
    default:
        // 返回-ENODATA让核心处理标准命令
        ret = -ENODATA;
        break;
    }
    
    return ret;
}

static int my_device_write_word_data(struct i2c_client *client,
                                     int page, int reg, u16 word)
{
    switch (reg) {
    case PMBUS_VIRT_RESET_VOUT_HISTORY:
        // 执行历史重置命令
        return pmbus_write_byte(client, page, MFR_RESET_HISTORY);
    default:
        return -ENODATA;
    }
}

static struct pmbus_driver_info my_device_info = {
    .pages = 1,
    .read_word_data = my_device_read_word_data,
    .write_word_data = my_device_write_word_data,
    // ... 其他配置
};
```

### Direct模式系数配置

```c
// 对于使用Direct数据格式的设备
static struct pmbus_driver_info my_device_info = {
    .pages = 1,
    .format[PSC_VOLTAGE_OUT] = direct,
    .format[PSC_CURRENT_OUT] = direct,
    
    // 系数: X = 1/m × (Y × 10^(-R) - b)
    // 假设文档给出: m=10, b=0, R=-2
    .m[PSC_VOLTAGE_OUT] = 10,
    .b[PSC_VOLTAGE_OUT] = 0,
    .R[PSC_VOLTAGE_OUT] = -2,
    
    .m[PSC_CURRENT_OUT] = 100,
    .b[PSC_CURRENT_OUT] = 0,
    .R[PSC_CURRENT_OUT] = -1,
    // ...
};
```

### 动态识别函数

```c
static int my_device_identify(struct i2c_client *client,
                              struct pmbus_driver_info *info)
{
    int ret, device_id;
    
    // 读取设备ID寄存器
    ret = i2c_smbus_read_word_data(client, MFR_DEVICE_ID);
    if (ret < 0)
        return ret;
    
    device_id = ret;
    
    // 根据设备型号配置功能
    switch (device_id) {
    case DEVICE_VARIANT_A:
        info->pages = 2;
        info->func[1] = PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT;
        break;
    case DEVICE_VARIANT_B:
        info->pages = 4;
        for (int i = 0; i < 4; i++)
            info->func[i] = PMBUS_HAVE_VOUT | PMBUS_HAVE_IOUT;
        break;
    default:
        return -ENODEV;
    }
    
    return 0;
}

static struct pmbus_driver_info my_device_info = {
    .identify = my_device_identify,
    // 其他字段由identify()动态配置
};
```

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
