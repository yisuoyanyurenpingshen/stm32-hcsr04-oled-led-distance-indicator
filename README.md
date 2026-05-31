# stm32-hcsr04-oled-led-distance-indicator
基于 STM32 HCSR04,LED  和 OLED 的简易测距小程序



## 项目简介

本项目基于 **STM32F103C8T6 + HAL 库** 实现了一个简单的超声波距离检测小仪器。

系统使用 **HC-SR04 超声波模块**测量前方物体距离，并通过：

* **OLED 屏幕**实时显示距离信息；
* **3 个 LED**根据距离远近显示不同等级；
* 使用 **DWT->CYCCNT** 实现微秒级延时；
* 使用 `timeout` 防止 Echo 等待时程序卡死。

这是一个面向嵌入式初学者的练习项目，主要用于学习 STM32 GPIO、I2C OLED、超声波测距、微秒延时、模块封装和基础 Debug 思维。

---

## 项目功能

* HC-SR04 超声波测距
* OLED 实时显示距离
* OLED 显示 Echo 高电平时间
* 三个 LED 根据距离等级点亮
* DWT 微秒延时
* 超时保护，测距失败时返回 `999`
* 支持 HAL 库开发
* 代码初步模块化，超声波部分封装为 `hc_sr04.c` 和 `hc_sr04.h`

---

## 硬件材料

| 模块                 | 说明           |
| ------------------ | ------------ |
| STM32F103C8T6      | 主控板          |
| HC-SR04            | 超声波测距模块      |
| OLED 0.96 寸 I2C 屏幕 | 显示距离         |
| LED × 3            | 距离等级提示       |
| 220Ω 电阻 × 3        | LED 限流       |
| 10kΩ / 18kΩ 电阻     | Echo 分压保护    |
| 杜邦线                | 接线           |
| 外接 5V 电源           | 给 HC-SR04 供电 |

---

## 硬件接线

### HC-SR04 超声波模块

| HC-SR04 引脚 | STM32 引脚 | 说明                |
| ---------- | -------- | ----------------- |
| VCC        | 外接 5V    | 超声波模块供电           |
| GND        | GND      | 必须与 STM32 共地      |
| Trig       | PA8      | STM32 输出触发信号      |
| Echo       | PA9      | STM32 输入回波信号，需要分压 |

注意：如果 HC-SR04 使用 5V 供电，Echo 输出通常也是 5V。STM32 引脚一般只能承受 3.3V，因此 Echo 不建议直接接 PA9。

建议分压接法：

```text
HC-SR04 Echo ---- 10kΩ ---- PA9 ---- 18kΩ ---- GND
```

这样可以把 Echo 的 5V 高电平降低到约 3.2V，保护 STM32 引脚。

---

### OLED 接线

OLED 使用 I2C 通信。

| OLED 引脚 | STM32 引脚 |
| ------- | -------- |
| VCC     | 3.3V     |
| GND     | GND      |
| SCL     | PB6      |
| SDA     | PB7      |

OLED 的 I2C 地址通常为：

```c
0x3C
```

在 STM32 HAL 中使用时一般写成：

```c
#define OLED_ADDR (0x3C << 1)
```

也就是：

```c
0x78
```

---

### LED 接线

| LED  | STM32 引脚 | 说明     |
| ---- | -------- | ------ |
| LED1 | PB0      | 距离等级 1 |
| LED2 | PB1      | 距离等级 2 |
| LED3 | PB10     | 距离等级 3 |

推荐接法：

```text
PBx ---- 220Ω ---- LED 正极
LED 负极 ---- GND
```

如果使用这种接法，一般是：

```text
GPIO_PIN_SET   = LED 亮
GPIO_PIN_RESET = LED 灭
```

---

## 距离等级逻辑

| 距离范围             | LED 状态    |
| ---------------- | --------- |
| 测距失败 / 距离 > 50cm | 全灭        |
| 30cm < 距离 ≤ 50cm | 亮 1 个 LED |
| 15cm < 距离 ≤ 30cm | 亮 2 个 LED |
| 距离 ≤ 15cm        | 亮 3 个 LED |

核心逻辑：

```c
if (distance_cm == 999 || distance_cm > 50)
{
    led_level = 0;
}
else if (distance_cm > 30)
{
    led_level = 1;
}
else if (distance_cm > 15)
{
    led_level = 2;
}
else
{
    led_level = 3;
}

LED_Show_Level(led_level);
```

---

## OLED 显示内容

OLED 当前显示三行内容：

```text
HC-SR04 Meter
Dist: xx cm
Echo: xxxx us
```

如果没有检测到回波，则显示：

```text
HC-SR04 Meter
Dist: No Echo
Echo: 0 us
```

`sprintf()` 用于把数字转换成字符串，再交给 OLED 显示函数：

```c
sprintf(line2, "Dist: %lu cm", distance_cm);
OLED_ShowString(2, 0, line2);
```

显示流程可以理解为：

```text
数字变量
↓
sprintf 拼成字符串
↓
OLED_ShowString 显示字符串
```

---

## HC-SR04 测距原理

HC-SR04 的基本工作流程：

1. STM32 给 Trig 引脚一个 10us 高电平；
2. HC-SR04 发射超声波；
3. Echo 引脚输出一个高电平脉冲；
4. Echo 高电平持续时间表示超声波往返时间；
5. 根据声速换算距离。

Trig 触发代码：

```c
HAL_GPIO_WritePin(HC_SR04_TRIG_PORT, HC_SR04_TRIG_PIN, GPIO_PIN_RESET);
delay_us(2);

HAL_GPIO_WritePin(HC_SR04_TRIG_PORT, HC_SR04_TRIG_PIN, GPIO_PIN_SET);
delay_us(10);

HAL_GPIO_WritePin(HC_SR04_TRIG_PORT, HC_SR04_TRIG_PIN, GPIO_PIN_RESET);
```

---

## 距离计算公式

声音在空气中的速度约为：

```text
340 m/s = 0.034 cm/us
```

Echo 时间是声音的往返时间，因此真实距离需要除以 2：

```text
distance_cm = duration_us × 0.034 / 2
```

化简后近似为：

```c
distance_cm = duration_us / 58;
```

所以代码中使用：

```c
distance = duration / 58;
```

其中：

* `duration`：Echo 高电平持续时间，单位 us；
* `distance`：测得距离，单位 cm。

---

## DWT 微秒延时

普通的 `HAL_Delay()` 是毫秒级延时，无法产生 10us 触发脉冲。

因此本项目使用 DWT 计数器实现微秒延时。

初始化 DWT：

```c
void HC_SR04_Init(void)
{
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT->CYCCNT = 0;
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;
}
```

微秒延时函数：

```c
static void delay_us(uint32_t us)
{
    uint32_t start = DWT->CYCCNT;
    uint32_t ticks = us * (HAL_RCC_GetHCLKFreq() / 1000000);

    while ((DWT->CYCCNT - start) < ticks);
}
```

如果 STM32 主频为 72MHz，那么：

```text
1us ≈ 72 个 CPU 周期
10us ≈ 720 个 CPU 周期
```

---

## timeout 防卡死机制

如果 HC-SR04 没有收到回波，Echo 可能一直不变化。

如果没有超时保护，程序会一直卡在 while 循环中。

本项目使用：

```c
timeout = 30000;
while (HAL_GPIO_ReadPin(HC_SR04_ECHO_PORT, HC_SR04_ECHO_PIN) == GPIO_PIN_RESET)
{
    if (timeout-- == 0)
    {
        echo_time_us = 0;
        return 999;
    }

    delay_us(1);
}
```

这里的 `timeout = 30000` 代表最多等待约 30ms。

如果超时，则返回：

```c
999
```

表示测距失败。

---

## 代码结构

```text
Core/
├── Inc/
│   ├── hc_sr04.h
│   ├── oled.h
│   └── ...
│
├── Src/
│   ├── main.c
│   ├── hc_sr04.c
│   ├── oled.c
│   └── ...
```

### hc_sr04.h

负责声明超声波模块的接口：

```c
void HC_SR04_Init(void);
uint32_t HC_SR04_Read_cm(void);
extern uint32_t echo_time_us;
```

### hc_sr04.c

负责实现：

* DWT 微秒延时；
* Trig 触发；
* Echo 高电平计时；
* 距离计算；
* timeout 防卡死。

### main.c

负责整体逻辑：

* 读取距离；
* 判断 LED 等级；
* 控制 LED；
* OLED 显示距离。

---

## Debug 记录

在调试过程中遇到过以下问题：

### 1. PB1 不按预期亮

原因是之前存在测试闪烁代码，与距离等级控制代码同时修改 GPIO，导致 LED 状态被覆盖。

解决方法：

* 删除测试闪烁代码；
* 使用 `LED_Show_Level(level)` 统一控制 LED。

### 2. 超声波测距显示 999

`999` 表示测距失败，通常可能由以下原因导致：

* HC-SR04 没有正常供电；
* STM32 和外接电源没有共地；
* Trig / Echo 接反；
* Echo 没有经过分压；
* PA8 / PA9 引脚配置错误；
* Echo 信号没有被 STM32 正确读取。

### 3. 建议先判断状态，再执行动作

本项目最终采用：

```text
读取距离
↓
判断 led_level
↓
统一执行 LED_Show_Level(led_level)
↓
OLED 显示距离和状态
```

这样比一边判断一边执行更清晰，也更方便 Debug。

---

## 当前效果

* 手远离超声波模块时，LED 全灭；
* 手靠近模块时，LED 逐渐增加点亮数量；
* OLED 显示当前距离和 Echo 时间；
* 程序可以处理无回波情况，不会卡死。

---

## 后续计划

* 加入 PWM，实现 LED 渐亮渐暗；
* OLED 显示 LED Level；
* 增加按键切换模式；
* 将 LED 控制也封装成 `led.c` / `led.h`；
* 增加摄像头模块，实现视觉感知；
* 绘制 PCB 或制作外壳。

---

## 项目总结

这个项目是一个基于 STM32 HAL 库的超声波感知小仪器。通过 HC-SR04 获取距离信息，再根据距离远近控制三个 LED，并在 OLED 上显示测距结果。

通过本项目，我学习了：

* GPIO 输出控制 LED；
* GPIO 输入读取 Echo；
* I2C OLED 显示；
* DWT 微秒延时；
* HC-SR04 超声波测距原理；
* timeout 防止程序卡死；
* 使用 `sprintf()` 将数字转换成字符串；
* 通过中间变量 `led_level` 分离状态判断和动作执行；
* 初步进行 `.c` / `.h` 模块封装。

本项目是从基础 GPIO 控制走向传感器测量、OLED 显示和嵌入式模块化开发的一次练习。
