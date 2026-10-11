# STM32F407 DAQ Firmware

基于 STM32F407ZGT6 的数据采集固件项目，使用 STM32CubeMX 配置外设，使用 Keil MDK 构建与调试。后续逐步实现 DMA 分块采集、FreeRTOS 任务协作、故障诊断与可恢复固件升级。

## 当前状态

当前提交保存正在开发中的工程。LED 与 USART1 已完成基础实板验证；TIM2 触发 ADC 的配置和启动流程已写入源码，但当前主循环尚未读取 ADC 转换结果，也尚未输出采样数据，仍保留 `HAL_Delay(500U)`。本版本的完整采集链路尚未验证。

| 模块 | 当前实现 | 验证状态 |
| --- | --- | --- |
| 系统时钟 | 外部 8 MHz HSE，经 PLL 配置为 168 MHz | 基础工程已运行 |
| LED | PF9、PF10 输出配置，PF9 在主循环翻转 | 已完成基础实板验证 |
| USART1 | PA9 / PA10，115200 baud，8N1，输出启动日志 | 已完成基础实板验证 |
| ADC1 | PA5 / Channel 5，12 位，TIM2 TRGO 上升沿触发 | 完整读取与输出流程待完成 |
| TIM2 | PSC=8399，ARR=999，Update Event 作为 TRGO | 已配置并加入启动调用，时序待实测 |
| DMA / FreeRTOS / MPU / Bootloader | 计划实现 | 尚未实现 |

按当前时钟和分频设置计算，TIM2 触发频率为 `84 MHz / 8400 / 1000 = 10 Hz`。这是配置推算值，不能替代实测结果；当前软件也尚未证明能逐次读取全部转换结果。ADC 当前采样时间为 3 Cycles，后续需结合输入电路核对并与 `.ioc` 同步调整。

## 硬件与工具

- 开发板：正点原子探索者 STM32F407 V3 系列。
- MCU：STM32F407ZGT6，Cortex-M4。
- IDE：Keil MDK；已有构建日志记录的版本为 5.29。
- 编译器：工程选择 ARM Compiler 5.06 update 6，build 750。
- 配置工具：`.ioc` 记录 STM32CubeMX 6.18.1。
- 固件包：STM32Cube FW_F4 V1.28.3。
- 调试接口：SWD。

## 工程结构

```text
.
├── README.md
├── LICENSE
├── .gitignore
└── v1/
    ├── v1.ioc                  # CubeMX 外设与时钟配置
    ├── Core/
    │   ├── Inc/                # 工程头文件
    │   └── Src/                # 主程序、外设配置、中断入口
    ├── Drivers/
    │   ├── CMSIS/              # 芯片与内核基础支持
    │   └── STM32F4xx_HAL_Driver/# ST HAL 外设驱动
    └── MDK-ARM/
        ├── v1.uvprojx          # Keil 工程入口
        ├── startup_stm32f407xx.s
        └── RTE/                # 工程运行环境配置
```

当前用户逻辑仍位于 `main.c` 的 `USER CODE` 区域。后续按硬件接线与操作接口、采集流程与数据处理的职责，逐步拆分 BSP 和 App 模块。

## 构建与调试

1. 安装 Keil MDK、ARM Compiler 5，以及 STM32F4 对应的设备支持包。工程当前引用 `Keil.STM32F4xx_DFP.3.1.1`。
2. 打开 `v1/MDK-ARM/v1.uvprojx`，选择 `v1` target，并确认使用 ARM Compiler 5。
3. 执行 Rebuild，检查编译错误和警告。已有构建日志包含未使用变量警告，本次上传未重新编译。
4. 通过 SWD 下载程序，并按实际使用的调试器配置连接。
5. 使用 115200 baud、8N1 打开 USART1 对应串口，检查启动日志和 LED 行为。采样值输出将在后续实现。

如需修改外设配置，打开 `v1/v1.ioc` 并由 CubeMX 重新生成；用户逻辑保留在 `USER CODE` 区域或独立模块中，避免修改生成区域导致后续覆盖。Keil 输出目录、构建日志和个人 IDE 配置已加入 `.gitignore`。

## 后续开发

- 完成 TIM2 触发 ADC 的结果读取、统计和串口输出，并记录实板验证结果。
- 建立 BSP 与 App 模块边界，明确初始化、启动、读取、停止和错误处理职责。
- 接入 DMA 分块采集，明确缓冲区所有权、溢出策略和数据缺口统计。
- 接入 FreeRTOS，测量任务唤醒与数据处理耗时。
- 实现 MPU 权限边界及异常现场记录。
- 实现可恢复固件升级、试运行确认和失败回退，并进行复位与断电测试。

## 许可证

仓库自有代码适用根目录的 `LICENSE`。`Drivers` 中的 ST HAL、CMSIS 和设备支持代码保留各自的许可证与版权声明，使用时遵循相应目录的许可证条款。
