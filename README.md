# STM32F407 PWM 呼吸灯工程说明

## 一、项目概述

本工程基于 **STM32F407ZGT6** 主控芯片，使用 **STM32CubeMX + Keil MDK (HAL库)** 开发，通过两个独立定时器输出 **10 kHz PWM** 波形：

- **TIM1_CH1（PA8）**：对外 PWM 输出引脚，高级定时器，需开启 MOE 主输出
- **TIM14_CH1（PF9）**：直接驱动板载 LED（DS0），实现呼吸灯效果

程序运行后，板载 LED 呈现 **暗 → 亮 → 暗** 的平滑呼吸渐变，同时 PA8 引脚同步输出同频率、同占空比的 PWM 波形。

---

## 二、硬件平台

| 项目 | 参数 |
|:---|:---|
| 主控芯片 | STM32F407ZGT6 |
| 核心板 | 正点原子 F407 核心板 |
| 下载方式 | FlyMcu 串口 ISP 烧录（CH340） |
| 外部晶振 | 8 MHz |
| 板载 LED | DS0 接 PF9，低电平点亮 |

---

## 三、CubeMX 配置详解

### 3.1 系统时钟配置

| 参数 | 值 | 说明 |
|:---|:---|:---|
| HSE | Crystal/Ceramic Resonator | 使用外部 8 MHz 无源晶振 |
| PLLM | 8 | 8 MHz ÷ 8 = 1 MHz（VCO 输入最佳频率） |
| PLLN | 336 | 1 MHz × 336 = 336 MHz（VCO 输出） |
| PLLP | 2 | 336 MHz ÷ 2 = 168 MHz（系统主频） |
| AHB 分频 | /1 | HCLK = 168 MHz |
| APB1 分频 | /4 | PCLK1 = 42 MHz，定时器时钟 = 84 MHz |
| APB2 分频 | /2 | PCLK2 = 84 MHz，定时器时钟 = 168 MHz |

> **定时器时钟规则**：当 APB 预分频系数 ≠ 1 时，该总线上的定时器时钟频率 = 2 × PCLK。

### 3.2 GPIO 配置（PE2 / PE3）

| 参数 | 值 | 说明 |
|:---|:---|:---|
| GPIO output level | Low | 上电默认输出低电平 |
| GPIO mode | Output Push Pull | 推挽输出，驱动能力强 |
| Pull-up/Pull-down | No pull-up/no pull-down | 推挽输出无需上下拉 |
| Maximum output speed | High | 高速翻转，波形边沿陡 |

### 3.3 TIM1 配置（高级定时器，PA8 输出）

| 参数 | 值 | 说明 |
|:---|:---|:---|
| Clock Source | Internal Clock | 使用内部总线时钟 |
| Prescaler (PSC) | 167 | 168 MHz ÷ (167+1) = 1 MHz 计数频率 |
| Counter Mode | Up | 向上计数 |
| Counter Period (ARR) | 99 | (99+1) 个计数步 = 100 μs |
| PWM Mode | Mode 1 | CNT < CCR 时输出有效电平 |
| Pulse (CCR1) | 0 | 初始占空比 0% |
| Channel Polarity | High | 有效电平为高电平 |

> **输出频率**：168 MHz ÷ (167+1) ÷ (99+1) = **10 kHz**
>
> **注意**：TIM1 是高级定时器，必须额外开启 MOE（Master Output Enable）主输出，否则引脚无波形。

### 3.4 TIM14 配置（通用定时器，PF9 输出驱动 LED）

| 参数 | 值 | 说明 |
|:---|:---|:---|
| Clock Source | Internal Clock | 使用内部总线时钟 |
| Prescaler (PSC) | 83 | 84 MHz ÷ (83+1) = 1 MHz 计数频率 |
| Counter Mode | Up | 向上计数 |
| Counter Period (ARR) | 99 | (99+1) 个计数步 = 100 μs |
| PWM Mode | Mode 1 | CNT < CCR 时输出有效电平 |
| Pulse (CCR1) | 0 | 初始占空比 0% |
| Channel Polarity | High | 有效电平为高电平 |

> **输出频率**：84 MHz ÷ (83+1) ÷ (99+1) = **10 kHz**
>
> TIM14 是通用定时器，无 MOE 主输出开关，调用启动函数即可直接输出。

---

## 四、核心代码

### 4.1 main.c 用户代码区域

```c
/* USER CODE BEGIN 2 */
// 启动 TIM1 通道1 PWM + 开启高级定时器主输出 MOE（必须）
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
__HAL_TIM_MOE_ENABLE(&htim1);

// 启动 TIM14 通道1 PWM（通用定时器，无需 MOE）
HAL_TIM_PWM_Start(&htim14, TIM_CHANNEL_1);

uint16_t ccr_val = 0;   // 占空比控制变量，范围 0~100
int8_t dir = 1;         // 方向：1 = 占空比增大，-1 = 占空比减小
/* USER CODE END 2 */
```

```c
/* USER CODE BEGIN 3 */
// 同步设置两路 PWM 的比较值，保持占空比一致
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, ccr_val);
__HAL_TIM_SET_COMPARE(&htim14, TIM_CHANNEL_1, ccr_val);

// 双向渐变
ccr_val += dir * 5;
if (ccr_val >= 100) dir = -1;   // 到最大值，开始减小
if (ccr_val <= 0)  dir = 1;     // 到最小值，开始增大

HAL_Delay(100);                  // 每 100ms 步进一次，控制呼吸速度
/* USER CODE END 3 */
```

### 4.2 代码逻辑说明

1. **两路 PWM 同步**：同一个 `ccr_val` 变量同时写入 TIM1 和 TIM14 的 CCR 寄存器，保证 PA8 和 PF9 占空比完全一致。
2. **双向呼吸**：`dir` 变量控制增减方向，CCR 在 0~100 之间往复循环，实现暗→亮→暗的呼吸效果。
3. **步进与速度**：每次变化 5（占空比 5%），间隔 100ms，一个完整呼吸周期约为 **4 秒**。

---

## 五、PWM 呼吸灯原理

### 5.1 PWM 基本概念

PWM（Pulse Width Modulation，脉冲宽度调制）：**频率固定，通过改变一个周期内高电平的时间占比来控制平均输出电压**。

- 占空比 = 高电平时间 ÷ 周期总时间 × 100%
- 占空比越高，平均电压越高，LED 亮度越高

### 5.2 定时器生成 PWM 的过程

向上计数 + PWM 模式 1：

```
计数器 CNT:  0 → 1 → 2 → ... → CCR → ... → ARR → 0 → ...
输出电平:    高电平（有效） ──────────────→ 低电平（无效）
```

- CNT < CCR：输出有效电平（高）
- CNT ≥ CCR：输出无效电平（低）

**CCR 值越大** → 高电平时间越长 → 占空比越高 → LED 越亮。

### 5.3 为什么 LED 会呈现呼吸效果

- PWM 频率 10 kHz，每秒闪烁 10000 次，远超人眼视觉暂留极限（约 50 Hz），人眼看不到闪烁，只能感知平均亮度。
- 代码缓慢改变 CCR 值（每 100ms 变 5%），占空比连续平滑变化，LED 亮度随之平滑过渡，形成呼吸效果。

### 5.4 板载 LED 电平对应关系

板载 DS0 为**共阳极接法**：LED 阳极接 3.3V，阴极接 PF9。

| CCR 值 | 占空比 | PF9 电平 | LED 状态 |
|:---|:---|:---|:---|
| 0 | 0% | 全程低 | 最亮 |
| 50 | 50% | 高低各半 | 半亮 |
| 100 | 100% | 全程高 | 熄灭 |

> 极性设为 High 时，CCR 越大 → 高电平占比越高 → LED 越暗。代码中 CCR 从 0→100→0 循环，即呈现暗→亮→暗的呼吸效果。

---

## 六、FlyMcu 烧录步骤

### 6.1 Keil 生成 HEX 文件

1. 打开 Keil 工程，点击魔术棒（Options for Target）
2. 切换到 **Output** 标签
3. 勾选 `Create HEX File`
4. 按 `F7` 编译工程，生成 `.hex` 文件

### 6.2 FlyMcu 参数配置

| 参数 | 设置值 |
|:---|:---|
| 串口 | CH340 对应的 COM 口 |
| 波特率 | 115200 |
| 芯片类型 | STM32F407ZG |
| 编程文件 | 选择编译生成的 .hex 文件 |
| 下载选项 | ✅ DTR 低电平复位，RTS 高电平进入 BootLoader |
| 编程选项字节 | ❌ **取消勾选**（避免修改芯片配置） |

### 6.3 烧录操作

1. 板子通过 Type-C 接入 USB_TTL 口
2. 打开 FlyMcu，加载 hex 文件
3. 点击「开始编程」
4. 看到软件提示 DTR 置高时，**按一下板子复位键**
5. 等待进度条 100%，提示「编程完成」
6. 烧录完成后按复位或重新上电，程序自动运行

> **一键下载时序说明**：STM32 仅在复位上电瞬间采样 BOOT0 引脚电平。一键下载电路偶尔时序偏差，导致芯片未进入 Bootloader，此时手动按一次复位即可对齐时序。

---

## 七、工程验收标准

| 验收项 | 预期结果 |
|:---|:---|
| 板载 LED 呼吸效果 | DS0 平滑暗→亮→暗循环，无跳变、无肉眼闪烁 |
| PWM 频率 | 10 kHz，误差 < 1% |
| PA8 输出 | 同步输出 10 kHz PWM，占空比与 LED 同步变化 |
| PE2 / PE3 | 推挽输出，默认低电平 |
| 系统时钟 | 168 MHz |

---

## 八、常见问题排查

| 现象 | 可能原因 | 解决方法 |
|:---|:---|:---|
| FlyMcu 连接失败，收到 0x4C 乱码 | 一键下载选项勾选错误 / BOOT0 未拉高 | 手动 BOOT0 接 3.3V，取消一键下载选项 |
| FlyMcu 收到 0x2A，烧录失败 | 一键下载时序不对，芯片仍跑旧程序 | 点开始编程后手动按一次复位键 |
| 烧录成功但 LED 不亮 | 选项字节被改写 / 读保护 | 取消「编程选项字节」，重新烧录 |
| TIM1 PA8 无波形 | MOE 未开启 | 确认 `__HAL_TIM_MOE_ENABLE()` 在 `HAL_TIM_PWM_Start` 之后调用 |
| LED 亮暗方向反了 | PWM 极性设置相反 | 修改 TIM14 Channel Polarity 为 High 或 Low |
| LED 呼吸速度太快/太慢 | HAL_Delay 参数不合适 | 调整 `HAL_Delay(100)` 中的数值 |
