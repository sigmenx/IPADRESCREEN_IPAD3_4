# STM32（HAL库）嵌入式比赛快速指南（完善版）
本文档基于 STM32F103 + HAL 库 编写，覆盖嵌入式比赛高频外设与实战技巧。比赛时仅需根据题目修改**外设编号、GPIO 端口、GPIO 引脚、波特率、PWM 频率、ADC 通道**等参数，即可快速复用代码。

---

## 一、工程基础与初始化总览
### 1.1 main 函数标准调用顺序
```c
#include "main.h"
I2C_HandleTypeDef hi2c1;
UART_HandleTypeDef huart1;
ADC_HandleTypeDef hadc1;
TIM_HandleTypeDef htim2;
TIM_HandleTypeDef htim3;
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART1_UART_Init();
    MX_I2C1_Init();
    MX_ADC1_Init();
    MX_TIM2_Init();        // 普通定时器中断
    MX_TIM3_PWM_Init();    // PWM
    HAL_TIM_Base_Start_IT(&htim2);          // 启动定时器中断
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1); // 启动 PWM 输出
    while (1)
    {
        // 主循环代码
    }
}
```

### 1.2 外设初始化通用套路
所有外设初始化均遵循统一流程，可按此模板快速排查问题：
1. 定义 GPIO_InitTypeDef 或外设配置结构体
2. 开启 GPIO 时钟
3. 开启外设时钟
4. 配置 GPIO 模式
5. 配置外设参数
6. 执行 `HAL_xxx_Init()`
7. 若使用中断，配置 NVIC 优先级
8. 在 main() 中调用启动函数

常用外设启动函数汇总：
```c
HAL_TIM_Base_Start_IT(&htim2);              // 启动定时器中断
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);   // 启动 PWM
HAL_UART_Receive_IT(&huart1, &rx_data, 1);  // 启动串口中断接收
HAL_ADC_Start(&hadc1);                      // 启动 ADC
```

### 1.3 不同系列芯片通用注意事项
F1 系列多数外设无需手动指定复用功能；若使用 STM32F4、G4、H7 等芯片，GPIO 复用配置需额外添加 `Alternate` 参数，示例如下：
- 串口：`GPIO_InitStruct.Alternate = GPIO_AF7_USART1;`
- I2C：`GPIO_InitStruct.Alternate = GPIO_AF4_I2C1;`
- PWM：`GPIO_InitStruct.Alternate = GPIO_AF2_TIM3;`

UART 完整示例：
```c
GPIO_InitStruct.Pin = GPIO_PIN_9 | GPIO_PIN_10;
GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
GPIO_InitStruct.Pull = GPIO_PULLUP;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
GPIO_InitStruct.Alternate = GPIO_AF7_USART1;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
```

### 1.4 CubeMX 工程代码放置规则
CubeMX 重新生成工程时会覆盖自动生成区域，自定义代码必须放入对应 `USER CODE` 块内：
```c
/* USER CODE BEGIN 0 */
// 自定义函数、全局变量声明
/* USER CODE END 0 */
```
```c
/* USER CODE BEGIN 2 */
// 初始化后启动外设：PWM、UART中断、ADC DMA、定时器中断等
/* USER CODE END 2 */
```
```c
/* USER CODE BEGIN WHILE */
while (1)
{
    /* USER CODE END WHILE */
    /* USER CODE BEGIN 3 */
    // 主循环业务逻辑
    /* USER CODE END 3 */
}
/* USER CODE END WHILE */
```

---

## 二、核心外设开发详解
### 2.1 GPIO 普通输入/输出
#### 2.1.1 初始化代码模板
示例：LED 接 PC13（推挽输出）、按键接 PA0（上拉输入）
```c
void MX_GPIO_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    /* 1. 开启 GPIO 时钟 */
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOC_CLK_ENABLE();
    /* 2. 设置 LED 默认电平 */
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
    /* 3. 配置 PC13 为普通推挽输出 */
    GPIO_InitStruct.Pin = GPIO_PIN_13;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
    /* 4. 配置 PA0 为输入 */
    GPIO_InitStruct.Pin = GPIO_PIN_0;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
}
```
#### 2.1.3 常用操作函数
```c
HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET); // 输出低电平
HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);   // 输出高电平
HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);                // 翻转电平
GPIO_PinState key = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0);
```

#### 2.1.4 按键消抖示例
```c
#define KEY_GPIO_Port GPIOB
#define KEY_Pin       GPIO_PIN_0
uint8_t Key_Scan(void)
{
    if (HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_RESET)  // 假设低电平按下
    {
        HAL_Delay(20);
        if (HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_RESET)
        {
            while (HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_RESET);
            return 1;
        }
    }
    return 0;
}
```

---

### 2.2 PWM 输出
#### 2.2.1 初始化代码模板
示例：TIM3_CH1，输出引脚 PA6，定时器时钟 72 MHz，PWM 频率 1 kHz，初始占空比 50%
```c
void MX_TIM3_PWM_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    TIM_OC_InitTypeDef sConfigOC = {0};
    /* 1. 开启 TIM3 和 GPIOA 时钟 */
    __HAL_RCC_TIM3_CLK_ENABLE();
    __HAL_RCC_GPIOA_CLK_ENABLE();
    /* 2. 配置 PA6 为 TIM3_CH1 复用推挽输出 */
    GPIO_InitStruct.Pin = GPIO_PIN_6;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    /* 3. 配置 TIM3 基本参数 */
    htim3.Instance = TIM3;
    htim3.Init.Prescaler = 72 - 1;              // 72MHz / 72 = 1MHz
    htim3.Init.CounterMode = TIM_COUNTERMODE_UP;
    htim3.Init.Period = 1000 - 1;              // 1MHz / 1000 = 1kHz
    htim3.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
    htim3.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;
    if (HAL_TIM_PWM_Init(&htim3) != HAL_OK)
    {
        Error_Handler();
    }
    /* 4. 配置 PWM 通道 */
    sConfigOC.OCMode = TIM_OCMODE_PWM1;
    sConfigOC.Pulse = 500;                     // 占空比 50%
    sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
    sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;
    if (HAL_TIM_PWM_ConfigChannel(&htim3, &sConfigOC, TIM_CHANNEL_1) != HAL_OK)
    {
        Error_Handler();
    }
}
```

#### 2.2.2 CubeMX 配置要点
1. 选择目标定时器（如 TIM2），开启 `PWM Generation CHx`
2. 配置 Prescaler、Counter Period 参数
3. 对应引脚自动切换为 `TIMx_CHy` 复用功能
4. 生成代码后，需在用户代码中手动启动 PWM

#### 2.2.3 频率与占空比计算
计算公式：
```text
PWM频率 = TIM时钟 / ((Prescaler + 1) * (Period + 1))
占空比 = Pulse / (Period + 1)
```
示例：72 MHz 定时器时钟，目标 1 kHz PWM
- Prescaler = 72 - 1
- Period = 1000 - 1
- PWM频率 = 72 MHz / 72 / 1000 = 1 kHz

若 Period = 999，占空比对应关系：
| Pulse | 占空比 |
|---:|---:|
| 0 | 0% |
| 250 | 25% |
| 500 | 50% |
| 999 | 接近 100% |

#### 2.2.4 常用操作函数
直接修改占空比：
```c
__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 700); // 70% 占空比
```

通用封装函数：
```c
void PWM_SetDuty(uint16_t duty)
{
    if (duty > 1000)
        duty = 1000;
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, duty);
}
```

通用比例式封装（支持任意定时器/通道）：
```c
void PWM_SetDuty(TIM_HandleTypeDef *htim, uint32_t channel, float duty)
{
    if (duty < 0.0f) duty = 0.0f;
    if (duty > 1.0f) duty = 1.0f;
    uint32_t arr = __HAL_TIM_GET_AUTORELOAD(htim);
    uint32_t ccr = (uint32_t)((arr + 1) * duty);
    __HAL_TIM_SET_COMPARE(htim, channel, ccr);
}
// 调用示例：PWM_SetDuty(&htim2, TIM_CHANNEL_1, 0.75f);
```

---

### 2.3 UART / USART 串口
#### 2.3.1 初始化代码模板
示例：USART1，TX=PA9，RX=PA10，波特率 115200
```c
void MX_USART1_UART_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    /* 1. 开启 USART1 和 GPIOA 时钟 */
    __HAL_RCC_USART1_CLK_ENABLE();
    __HAL_RCC_GPIOA_CLK_ENABLE();
    /* 2. 配置 PA9 为 USART1_TX */
    GPIO_InitStruct.Pin = GPIO_PIN_9;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    /* 3. 配置 PA10 为 USART1_RX */
    GPIO_InitStruct.Pin = GPIO_PIN_10;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    /* 4. 配置 USART1 参数 */
    huart1.Instance = USART1;
    huart1.Init.BaudRate = 115200;
    huart1.Init.WordLength = UART_WORDLENGTH_8B;
    huart1.Init.StopBits = UART_STOPBITS_1;
    huart1.Init.Parity = UART_PARITY_NONE;
    huart1.Init.Mode = UART_MODE_TX_RX;
    huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
    huart1.Init.OverSampling = UART_OVERSAMPLING_16;
    if (HAL_UART_Init(&huart1) != HAL_OK)
    {
        Error_Handler();
    }
    /* 5. 中断优先级 */
    HAL_NVIC_SetPriority(USART1_IRQn, 1, 0);
    HAL_NVIC_EnableIRQ(USART1_IRQn);
}
```

#### 2.3.2 CubeMX 配置要点
1. 开启目标 USARTx 外设
2. 常用参数：波特率 115200、数据位 8 Bits、无校验、停止位 1
3. TX/RX 引脚自动配置为复用功能
4. 若使用接收中断，在 NVIC 中开启对应串口中断
5. 若使用 DMA 接收，需配置 DMA 与 USART IDLE 中断

#### 2.3.3 基础收发操作
发送字符串：
```c
uint8_t msg[] = "Hello STM32\r\n";
HAL_UART_Transmit(&huart1, msg, sizeof(msg) - 1, 100);
```

轮询接收 1 字节：
```c
uint8_t rx_data;
HAL_UART_Receive(&huart1, &rx_data, 1, 100);
```

#### 2.3.4 中断接收配置
**单字节中断接收**
初始化开启中断：
```c
uint8_t rx_data;
HAL_UART_Receive_IT(&huart1, &rx_data, 1);
```

中断优先级配置（需加入初始化函数）：
```c
HAL_NVIC_SetPriority(USART1_IRQn, 1, 0);
HAL_NVIC_EnableIRQ(USART1_IRQn);
```

中断服务函数：
```c
void USART1_IRQHandler(void)
{
    HAL_UART_IRQHandler(&huart1);
}
```

接收回调函数：
```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // rx_data 中就是接收到的数据
        HAL_UART_Receive_IT(&huart1, &rx_data, 1); // 再次开启接收
    }
}
```

**多字节缓存中断接收**
全局变量：
```c
uint8_t uart_rx_ch;
uint8_t uart_rx_buf[128];
volatile uint16_t uart_rx_idx = 0;
```

回调函数：
```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        if (uart_rx_idx < sizeof(uart_rx_buf) - 1)
        {
            uart_rx_buf[uart_rx_idx++] = uart_rx_ch;
        }
        if (uart_rx_ch == '\n')
        {
            uart_rx_buf[uart_rx_idx] = '\0';
            printf("recv: %s", uart_rx_buf);
            uart_rx_idx = 0;
        }
        HAL_UART_Receive_IT(&huart1, &uart_rx_ch, 1);  // 必须重新开启
    }
}
```

---

### 2.4 I2C / IIC 总线
#### 2.4.1 初始化代码模板
示例：I2C1，SCL=PB6，SDA=PB7，标准模式 100 kHz
```c
void MX_I2C1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    /* 1. 开启 GPIOB 和 I2C1 时钟 */
    __HAL_RCC_GPIOB_CLK_ENABLE();
    __HAL_RCC_I2C1_CLK_ENABLE();
    /* 2. 配置 PB6/PB7 为 I2C 复用开漏 */
    GPIO_InitStruct.Pin = GPIO_PIN_6 | GPIO_PIN_7;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_OD;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
    /* 3. 配置 I2C1 参数 */
    hi2c1.Instance = I2C1;
    hi2c1.Init.ClockSpeed = 100000;             // 100kHz
    hi2c1.Init.DutyCycle = I2C_DUTYCYCLE_2;
    hi2c1.Init.OwnAddress1 = 0;
    hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
    hi2c1.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
    hi2c1.Init.OwnAddress2 = 0;
    hi2c1.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
    hi2c1.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE;
    if (HAL_I2C_Init(&hi2c1) != HAL_OK)
    {
        Error_Handler();
    }
}
```

#### 2.4.2 CubeMX 配置要点
1. 开启 I2C1 或 I2C2
2. 配置速率：Standard Mode 100 kHz / Fast Mode 400 kHz
3. SCL/SDA 自动配置为复用开漏模式
4. 确认硬件存在上拉电阻（多数模块自带 4.7k 上拉）
5. 普通比赛场景轮询模式即可满足需求，按需开启中断/DMA

#### 2.4.3 地址使用注意事项
HAL 库 I2C 函数要求传入**8 位地址格式**，即手册给出的 7 位地址左移一位：
```c
#define DEV_ADDR  (0x68 << 1)
```
这是比赛中最常见的通信失败原因。

#### 2.4.4 常用读写操作
主机发送/接收：
```c
uint8_t data = 0x55;
/* 主机发送 */
HAL_I2C_Master_Transmit(&hi2c1, 0x50 << 1, &data, 1, 100);
/* 主机接收 */
HAL_I2C_Master_Receive(&hi2c1, 0x50 << 1, &data, 1, 100);
```

寄存器读写：
```c
// 写寄存器
uint8_t value = 0x12;
HAL_I2C_Mem_Write(&hi2c1,          // 1. I2C外设句柄指针，指定使用哪一路I2C外设（这里是I2C1）
                  0x50 << 1,       // 2. 从设备8位地址：7位设备地址左移1位，最低位为0表示写方向
                  reg,              // 3. 目标从设备的内部寄存器地址，要往哪个寄存器写数据
                  I2C_MEMADD_SIZE_8BIT, // 4. 从设备寄存器地址的位宽：8位地址长度（部分芯片是16位）
                  &value,           // 5. 待写入数据的缓冲区指针，存放要写入寄存器的数据
                  1,                // 6. 要写入的数据字节数量
                  100);             // 7. 操作超时时间，单位：毫秒，超时未完成则返回错误

// 读寄存器
uint8_t reg = 0x00;
uint8_t value;
HAL_I2C_Mem_Read(&hi2c1,           // 1. I2C外设句柄指针，指定使用哪一路I2C外设
                 0x50 << 1,        // 2. 从设备基地址：7位地址左移1位，HAL库内部会自动把最低位置1表示读方向
                 reg,               // 3. 目标从设备的内部寄存器起始地址，从哪个寄存器开始读
                 I2C_MEMADD_SIZE_8BIT, // 4. 从设备寄存器地址的位宽：8位地址长度
                 &value,            // 5. 接收数据的缓冲区指针，读取到的数据会存入这个地址
                 1,                 // 6. 要读取的数据字节数量
                 100);              // 7. 操作超时时间，单位：毫秒，超时未完成则返回错误
```

连续读多个寄存器：
```c
uint8_t buf[6];
HAL_I2C_Mem_Read(&hi2c1, 0x68 << 1, 0x3B, I2C_MEMADD_SIZE_8BIT, buf, 6, 100);
```

16 位数据合成（传感器高字节在前场景）：
```c
int16_t raw = (int16_t)((buf[0] << 8) | buf[1]);
```

#### 2.4.5 I2C 设备扫描器
用于快速确认设备地址与通信链路是否正常：
```c
void I2C_Scan(void)
{
    printf("I2C scan start\r\n");
    for (uint8_t addr = 1; addr < 127; addr++)
    {
        if (HAL_I2C_IsDeviceReady(&hi2c1, addr << 1, 1, 10) == HAL_OK)
        {
            printf("Found I2C device: 0x%02X\r\n", addr);
        }
    }
    printf("I2C scan end\r\n");
}
```

---

### 2.5 ADC 模数转换
#### 2.5.1 初始化代码模板
示例：ADC1，通道 ADC_CHANNEL_1，引脚 PA1
```c
void MX_ADC1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    ADC_ChannelConfTypeDef sConfig = {0};
    /* 1. 开启 ADC1 和 GPIOA 时钟 */
    __HAL_RCC_ADC1_CLK_ENABLE();
    __HAL_RCC_GPIOA_CLK_ENABLE();
    /* 2. 配置 PA1 为模拟输入 */
    GPIO_InitStruct.Pin = GPIO_PIN_1;
    GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    /* 3. 配置 ADC1 参数 */
    hadc1.Instance = ADC1;
    hadc1.Init.ScanConvMode = ADC_SCAN_DISABLE;
    hadc1.Init.ContinuousConvMode = DISABLE;
    hadc1.Init.DiscontinuousConvMode = DISABLE;
    hadc1.Init.ExternalTrigConv = ADC_SOFTWARE_START;
    hadc1.Init.DataAlign = ADC_DATAALIGN_RIGHT;
    hadc1.Init.NbrOfConversion = 1;
    if (HAL_ADC_Init(&hadc1) != HAL_OK)
    {
        Error_Handler();
    }
    /* 4. 配置 ADC 通道 */
    sConfig.Channel = ADC_CHANNEL_1;
    sConfig.Rank = ADC_REGULAR_RANK_1;
    sConfig.SamplingTime = ADC_SAMPLETIME_55CYCLES_5;
    if (HAL_ADC_ConfigChannel(&hadc1, &sConfig) != HAL_OK)
    {
        Error_Handler();
    }
}
```

#### 2.5.2 CubeMX 配置要点（单通道轮询）
1. 开启 ADC 外设，选择对应输入通道
2. 设置分辨率：常用 12-bit
3. 采样时间：信号源阻抗大时适当加长
4. Continuous Conversion 可按需开启，常规使用软件触发即可

#### 2.5.3 读取与电压换算
单通道读取函数：
```c
uint16_t ADC_Read(void)
{
    uint16_t adc_value = 0;
    HAL_ADC_Start(&hadc1);
    if (HAL_ADC_PollForConversion(&hadc1, 100) == HAL_OK)
    {
        adc_value = HAL_ADC_GetValue(&hadc1);
    }
    HAL_ADC_Stop(&hadc1);
    return adc_value;
}
```

ADC 原始值转电压（12 位 ADC，参考电压 3.3V）：
```c
float ADC_ToVoltage(uint16_t adc_value)
{
    return adc_value * 3.3f / 4095.0f;
}
```

带外部分压的电压计算（示例：100k/100k 分压）：
```c
float BatteryVoltage(uint16_t raw)
{
    float v_adc = raw * 3.3f / 4095.0f;
    return v_adc * (100.0f + 100.0f) / 100.0f;
}
```

---

### 2.6 基础定时器（定时中断）
#### 2.6.1 初始化代码模板
示例：TIM2，1 ms 进入一次中断，定时器时钟 72 MHz
```c
void MX_TIM2_Init(void)
{
    /* 1. 开启 TIM2 时钟 */
    __HAL_RCC_TIM2_CLK_ENABLE();
    /* 2. 配置 TIM2 */
    htim2.Instance = TIM2;
    htim2.Init.Prescaler = 72 - 1;              // 72MHz / 72 = 1MHz
    htim2.Init.CounterMode = TIM_COUNTERMODE_UP;
    htim2.Init.Period = 1000 - 1;              // 1MHz / 1000 = 1kHz，也就是 1ms
    htim2.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
    htim2.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;
    if (HAL_TIM_Base_Init(&htim2) != HAL_OK)
    {
        Error_Handler();
    }
    /* 3. 配置 NVIC */
    HAL_NVIC_SetPriority(TIM2_IRQn, 1, 0);
    HAL_NVIC_EnableIRQ(TIM2_IRQn);
}
```

#### 2.6.2 CubeMX 配置要点
1. 开启目标 TIMx，Clock Source 选择 Internal Clock
2. 配置 Prescaler 和 Period 参数
3. 在 NVIC 中开启 TIMx global interrupt
4. 生成代码后，需手动调用启动函数开启中断

#### 2.6.3 中断频率计算
计算公式：
```text
中断频率 = TIM时钟 / ((Prescaler + 1) * (Period + 1))
```
示例：72 MHz 定时器时钟，目标 1 ms 中断
- Prescaler = 72 - 1
- Period = 1000 - 1

#### 2.6.4 中断回调与使用示例
中断服务函数：
```c
void TIM2_IRQHandler(void)
{
    HAL_TIM_IRQHandler(&htim2);
}
```

定时器更新回调：
```c
volatile uint32_t timer_count = 0;
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        timer_count++;
        // 每 1000ms 执行一次
        if (timer_count >= 1000)
        {
            timer_count = 0;
            HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
        }
    }
}
```

主循环多任务调度示例：
```c
uint32_t last_led_tick = 0;
uint32_t last_adc_tick = 0;
while (1)
{
    uint32_t now = HAL_GetTick();
    if (now - last_led_tick >= 500)
    {
        last_led_tick = now;
        LED_Toggle();
    }
    if (now - last_adc_tick >= 100)
    {
        last_adc_tick = now;
        uint16_t adc = ADC_Read();
        printf("ADC=%u\r\n", adc);
    }
}
```

---

### 2.7 外部中断 EXTI
#### 2.7.1 初始化代码模板
示例：按键 PA0，下降沿触发，使用 EXTI0_IRQn
```c
void MX_EXTI_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    /* 1. 开启 GPIOA 时钟 */
    __HAL_RCC_GPIOA_CLK_ENABLE();
    /* 2. 配置 PA0 为外部中断输入 */
    GPIO_InitStruct.Pin = GPIO_PIN_0;
    GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    /* 3. 配置 NVIC 中断优先级 */
    HAL_NVIC_SetPriority(EXTI0_IRQn, 2, 0);
    HAL_NVIC_EnableIRQ(EXTI0_IRQn);
}
```

#### 2.7.2 CubeMX 配置要点
1. 对应 GPIO Mode 选择 `GPIO_EXTI`，触发方式可选上升沿/下降沿/双边沿
2. 配置上下拉：低电平按下一般用上拉，高电平按下一般用下拉
3. 在 NVIC 中开启对应 EXTI 中断

#### 2.7.3 中断回调与消抖方案
基础中断服务函数与回调：
```c
void EXTI0_IRQHandler(void)
{
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_0)
    {
        HAL_Delay(10); // 简单消抖，比赛可用
        if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET)
        {
            HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
        }
    }
}
```

标志位式消抖（中断内仅置标志，主循环处理）：
```c
volatile uint32_t key_last_tick = 0;
volatile uint8_t key_flag = 0;
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_0)
    {
        uint32_t now = HAL_GetTick();
        if (now - key_last_tick > 20)
        {
            key_last_tick = now;
            key_flag = 1;
        }
    }
}
```
主循环处理：
```c
while (1)
{
    if (key_flag)
    {
        key_flag = 0;
        LED_Toggle();
    }
}
```

#### 2.7.4 引脚与中断线对应说明
PB12、PC13 等引脚（Pin 10~15）共用 EXTI15_10 中断线：
- 中断号：`EXTI15_10_IRQn`
- 中断服务函数示例：
```c
void EXTI15_10_IRQHandler(void)
{
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_13);
}
```

---

## 三、通用开发技巧
### 3.1 延时与非阻塞编程
#### 3.1.1 HAL_Delay 阻塞延时
```c
HAL_Delay(1000);  // 延时 1000 ms
```
适合简单流程，多任务场景不推荐使用。

#### 3.1.2 HAL_GetTick 非阻塞延时
比赛多任务场景优先推荐：
```c
uint32_t last_tick = 0;
while (1)
{
    if (HAL_GetTick() - last_tick >= 1000)
    {
        last_tick = HAL_GetTick();
        LED_Toggle();
    }
    // 其他任务可并行运行
}
```

#### 3.1.3 超时等待模板
```c
uint8_t Wait_Pin_Level(GPIO_TypeDef *port, uint16_t pin, GPIO_PinState target, uint32_t timeout_ms)
{
    uint32_t start = HAL_GetTick();
    while (HAL_GPIO_ReadPin(port, pin) != target)
    {
        if (HAL_GetTick() - start > timeout_ms)
        {
            return 0;  // timeout
        }
    }
    return 1;  // success
}
```

### 3.2 变量格式转换与位操作
#### 3.2.1 数值与字符串互转
整数转字符串：
```c
char buf[32];
int value = 123;
sprintf(buf, "%d", value);
```

浮点转字符串：
```c
char buf[32];
float v = 3.1415f;
sprintf(buf, "%.2f", v);
```
> 注意：部分嵌入式工程默认不支持 printf 浮点输出。若打印 float 异常，可检查链接选项，或使用整数放大法：
> ```c
> int mv = (int)(v * 1000);
> printf("V=%d.%03d\r\n", mv / 1000, mv % 1000);
> ```

字符串转数值：
```c
int a = atoi("123");
float f = atof("3.14");
```

sscanf 命令解析：
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main()
{
   int day, year;
   char weekday[20], month[20], dtm[100];

   strcpy( dtm, "Saturday March 25 1989" );
   sscanf( dtm, "%s %s %d  %d", weekday, month, &day, &year );

   printf("%s %d, %d = %s\n", month, day, year, weekday );
    
   return(0);
}
```

#### 3.2.2 常用格式符
| 格式符 | 含义 |
|---|---|
| `%d` | int |
| `%u` | unsigned int |
| `%ld` | long |
| `%lu` | unsigned long |
| `%f` | float / double |
| `%c` | char |
| `%s` | string |
| `%02X` | 两位十六进制，大写，不足补 0 |

#### 3.2.3 位操作常用写法
```c
uint8_t data = 0;
// 置位 bit3
data |= (1 << 3);
// 清零 bit3
data &= ~(1 << 3);
// 翻转 bit3
data ^= (1 << 3);
// 判断 bit3
if (data & (1 << 3))
{
    // bit3 is 1
}
```

### 3.3 中断编程原则
1. 中断内只执行短操作，尽量仅置标志位
2. 复杂逻辑统一放到主循环中处理
3. 中断内避免长时间 `printf` 输出
4. 中断内禁止随意调用 `HAL_Delay()`
5. 多个中断共享的变量必须加 `volatile` 修饰
6. 同个外设多个通道共享回调函数时，必须判断实例

示例：
```c
volatile uint8_t data_ready = 0;
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_1)
    {
        data_ready = 1;
    }
}
while (1)
{
    if (data_ready)
    {
        data_ready = 0;
        // 读取传感器、计算、打印等耗时操作
    }
}
```

### 3.4 DMA 简要说明
DMA 适用场景：
1. UART 大量数据接收
2. ADC 多通道连续采样
3. SPI / I2C 大批量数据传输
4. 降低 CPU 占用，提升系统效率

比赛最常用 ADC DMA 示例：
```c
uint16_t adc_buf[4];
HAL_ADC_Start_DMA(&hadc1, (uint32_t *)adc_buf, 4);
```

> 提示：UART DMA + IDLE 实现稍复杂，比赛中若仅需接收不定长数据，优先使用单字节中断法，稳定且易调试。

### 3.5 错误处理与调试
#### 3.5.1 HAL 函数返回值说明
| 返回值 | 含义 |
|---|---|
| `HAL_OK` | 执行成功 |
| `HAL_ERROR` | 执行错误 |
| `HAL_BUSY` | 外设正忙 |
| `HAL_TIMEOUT` | 执行超时 |

#### 3.5.2 返回值检查示例
```c
HAL_StatusTypeDef ret;
ret = HAL_I2C_Mem_Read(&hi2c1, 0x68 << 1, 0x75, I2C_MEMADD_SIZE_8BIT, &value, 1, 100);
if (ret != HAL_OK)
{
    printf("I2C read error: %d\r\n", ret);
}
```

#### 3.5.3 断言宏封装
```c
#define CHECK_HAL(x) do {                         \
    HAL_StatusTypeDef _ret = (x);                 \
    if (_ret != HAL_OK) {                         \
        printf("HAL error %d at %s:%d\r\n",       \
               _ret, __FILE__, __LINE__);         \
    }                                             \
} while (0)
```
调用示例：
```c
CHECK_HAL(HAL_I2C_Master_Transmit(&hi2c1, 0x68 << 1, data, 2, 100));
```

### 3.6 常用宏定义模板
```c
#define ARRAY_SIZE(x)       (sizeof(x) / sizeof((x)[0]))
#define LIMIT(x, min, max)  ((x) < (min) ? (min) : ((x) > (max) ? (max) : (x)))
#define ABS(x)              ((x) >= 0 ? (x) : -(x))
```


---

## 五、比赛实战指南

### 5.2 必背 HAL 函数速查表
| 外设分类 | 函数名 |
|---|---|
| GPIO | `HAL_GPIO_WritePin` |
| GPIO | `HAL_GPIO_ReadPin` |
| GPIO | `HAL_GPIO_TogglePin` |
| 延时 | `HAL_Delay` |
| 系统计时 | `HAL_GetTick` |
| UART | `HAL_UART_Transmit` |
| UART | `HAL_UART_Receive` |
| UART | `HAL_UART_Receive_IT` |
| I2C | `HAL_I2C_IsDeviceReady` |
| I2C | `HAL_I2C_Master_Transmit` |
| I2C | `HAL_I2C_Master_Receive` |
| I2C | `HAL_I2C_Mem_Read` |
| I2C | `HAL_I2C_Mem_Write` |
| ADC | `HAL_ADC_Start` |
| ADC | `HAL_ADC_PollForConversion` |
| ADC | `HAL_ADC_GetValue` |
| ADC DMA | `HAL_ADC_Start_DMA` |
| 定时器 | `HAL_TIM_Base_Start_IT` |
| PWM | `HAL_TIM_PWM_Start` |
| PWM | `__HAL_TIM_SET_COMPARE` |

### 5.3 现场排错清单
#### 编译不通过
1. 自定义函数是否提前声明
2. 是否包含对应外设头文件
3. 变量名是否与 CubeMX 生成的句柄一致（如 `hi2c1` / `huart2`）
4. 代码是否写在 `USER CODE` 区域外，导致重新生成后丢失
5. 自定义 C 文件是否加入工程编译
6. Keil 是否开启 C99，或变量声明是否放在语句前

#### 下载/调试异常
1. SYS Debug 是否选择 Serial Wire
2. BOOT0 引脚是否为低电平
3. 芯片供电是否正常
4. SWDIO / SWCLK 引脚是否被其他功能占用
5. NRST 复位引脚连接是否正常

#### 外设无响应
1. 外设对应引脚是否选择正确
2. 复用功能配置是否正确
3. 外设初始化函数是否被调用
4. 外设启动函数是否调用
   - PWM：`HAL_TIM_PWM_Start`
   - TIM 中断：`HAL_TIM_Base_Start_IT`
   - UART 中断：`HAL_UART_Receive_IT`
   - ADC DMA：`HAL_ADC_Start_DMA`
5. NVIC 中断是否使能
6. 外设模块与主控是否共地
7. 逻辑电平是否兼容

#### 串口无输出
1. TX/RX 是否接反
2. 两端波特率、数据位、停止位是否一致
3. 设备之间是否共地
4. `printf` 是否完成重定向
5. 串口号是否对应正确（`huart1` / `huart2`）
6. USB-TTL 模块是否正常工作

#### I2C 通信失败
1. 设备地址是否左移一位
2. SCL/SDA 是否接上拉电阻
3. SCL/SDA 线序是否接反
4. 设备之间是否共地
5. 先用 I2C 扫描器确认设备是否在线
6. 设备供电电压是否正确
7. 先降低到 100 kHz 标准模式调试
比赛/工程中最常用的经验取值如下：

| 系统电压 | 通信速率 | 推荐上拉电阻阻值 | 适用场景 |
|---------|---------|----------------|---------|
| 3.3V（最常用） | 标准模式 100kHz | 4.7kΩ ~ 10kΩ | STM32 + 绝大多数传感器、OLED 等 |
| 3.3V | 快速模式 400kHz | 2.2kΩ ~ 4.7kΩ | 高速传感器、多设备总线 |
| 5V | 标准模式 100kHz | 4.7kΩ ~ 10kΩ | 老式 5V 设备、部分传感器 |
| 5V | 快速模式 400kHz | 1kΩ ~ 2.2kΩ | 5V 高速总线 |

> 比赛万金油：3.3V 系统 + 100kHz 速率，直接用 **4.7kΩ** 上拉电阻接 3.3V，99% 的场景都能正常工作。

### 5.4 快速排错记忆口诀
1. **PWM 不动：先 Start，再查通道。**
2. **I2C 不通：先扫地址，再看左移。**
3. **串口没字：查 TX/RX、波特率、printf 重定向。**
4. **中断不进：查 NVIC、回调名、启动函数。**
5. **ADC 乱跳：查 Analog、采样时间、滤波、参考电压。**
6. **CubeMX 反复生成：代码必须放 USER CODE 区域。**
7. **比赛先跑通最小功能，再封装，再优化。**

### 5.5 赛前练习题
1. LED 500 ms 闪烁
2. 按键控制 LED 翻转，实现消抖
3. 串口输入 `LED ON` / `LED OFF` 控制 LED 状态
4. 串口输入 `PWM 75` 动态设置 PWM 占空比
5. ADC 读取电位器数值，串口打印对应电压
6. I2C 扫描总线上的设备地址
7. 读取 I2C 传感器的 WHO_AM_I 寄存器并打印
8. 定时器实现 1 ms 计数，主循环每 1 秒打印一次计数值
9. 外部中断按键触发事件，主循环处理对应逻辑
10. 多任务综合：LED 闪烁 + ADC 周期打印 + UART 命令解析 + PWM 输出

### 5.6 最小化答题策略
比赛时不要一开始就搭建复杂框架，建议按以下顺序推进：
1. 先确认硬件连接正确
2. 优先调通串口 printf 作为调试输出
3. 先跑通基础 GPIO 功能
4. 逐个外设调试验证，逐个功能叠加
5. 每完成一个功能及时备份工程
6. 复杂功能采用「标志位 + 主循环处理」模式
7. 外设异常时，先用最小测试代码排除硬件问题
