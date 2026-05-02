这份指导书基于你在 RK3588 平台上适配 1080x2160 MIPI DSI 屏幕（带 In-cell 触摸）的实际调试经历总结而成。

---

# RK3588 MIPI DSI 屏幕与 In-cell 触摸适配指导书

## 一、 硬件接口与引脚定义
在适配开始前，必须通过原理图确认物理连接。本案例中的 40-pin FPC 接口关键定义如下：
*   **LCD_RST (复位)**：GPIO3_B2 (引脚编号 106) 或 GPIO3_C1 (引脚编号 113) —— *本案最终确认为屏幕复位由 GPIO3_B2 控制*。
*   **TP_INT (中断)**：GPIO3_C0 (引脚编号 112)。
*   **TP_SCL/SDA (I2C5)**：本案使用 `i2c5_m0` 组（GPIO3_C7/GPIO3_D0，编号 119/120）。
*   **电源控制**：GPIO4_A4 (引脚编号 132) 控制屏幕 3.3V 主供电。

---

## 二、 显示部分适配 (MIPI DSI)

### 1. 电源节点 (Regulator) 配置
MIPI 屏幕对电源极其敏感。必须确保电源在启动时正确开启。
*   **关键属性**：使用 `regulator-fixed`。
*   **陷阱**：属性名应使用 `power-supply` 确保兼容性。
*   **调试技巧**：如果屏幕启动后几秒熄灭，日志显示 `disabling`，请在电源节点添加 `regulator-always-on;`。

### 2. DSI 节点与 Panel 配置
*   **Compatible**：通常使用 `"simple-panel-dsi"`。
*   **初始化序列 (panel-init-sequence)**：对于高分辨率 In-cell 屏幕，**必须使用厂家提供的长序列指令**（如包含 `9F A5 A5` 解锁指令的序列）。这些指令不仅初始化显示，还负责开启屏幕内部触摸模块的 LDO 和时钟。
*   **时序参数 (display-timings)**：根据屏幕规格书严格填写 `hactive`, `vactive` 及 `porch` 参数。

### 3. VOP2 路由与 U-Boot Logo 调试
*   **Logo 不亮排查**：若 eDP 能亮而 MIPI 不亮，检查 U-Boot 是否识别到 DSI 控制器。
*   **缩放错误**：若日志报 `scale factor is out of range`，说明当前视频端口（VP）无法处理该分辨率的缩放。
    *   **对策**：尝试更换视频通道（如从 `vp2` 切换到 `vp3`）。

---

## 三、 触摸部分适配 (I2C Touch)

### 1. I2C 引脚复用 (Pinmux) 冲突排查
这是最容易忽略的死穴。RK3588 的 I2C5 默认常与以太网 (GMAC1) 冲突。
*   **检查方法**：执行 `sudo cat /sys/kernel/debug/pinctrl/.../pinmux-pins | grep -E "114|115|119|120"`。
*   **修复**：若显示引脚被 `ethernet` 占用，必须在 DTS 中禁用 `&gmac1` 或修改其引脚复用。
*   **物理确认**：使用 `echo` 控制 GPIO 119/120 电平，配合万用表确认物理线路是否通往排线。

### 2. 触摸节点配置
*   **地址确认**：通过 `i2cdetect -y 5` 确认地址（本案为 `0x48`）。
*   **驱动匹配**：地址 `0x48` 多为 Samsung 或 FocalTech 的 In-cell 方案，对应 `compatible = "sec,sec_ts";`。
*   **中断与复位**：根据测量结果填写。注意：In-cell 屏触摸复位常与 LCD 复位共用。

---

## 四、 命令行调试工具箱

### 1. GPIO 计算与控制
*   **计算公式**：`编号 = 控制器*32 + 小组*8 + 编号` (A=0, B=1, C=2, D=3)。
*   **测试复位/电源**：
    ```bash
    echo 132 > /sys/class/gpio/export
    echo out > /sys/class/gpio/gpio132/direction
    echo 1 > /sys/class/gpio/gpio132/value # 拉高电源
    ```

### 2. I2C 调试
*   **扫描总线**：`i2cdetect -y 5`。
    *   出现 `48`：硬件连通但无驱动。
    *   出现 `UU`：驱动加载成功。
    *   出现 `timeout`：总线挂死，检查引脚复用冲突或电源。
*   **读取寄存器**：`i2cget -y 5 0x48 0x00`。

### 3. 触摸输入测试
*   **工具**：`evtest`。
*   **操作**：选择对应的输入设备，手指触摸看是否有坐标输出。

---

## 五、 适配总结清单

| 检查项 | 状态 | 关键点 |
| :--- | :--- | :--- |
| **电源** | 必查 | 测量 FPC 第 17 脚是否有 1.8V，确认电源 GPIO 是否拉高。 |
| **复位** | 必查 | 确认 `LCD_RST` (Pin 1) 为高电平。 |
| **MIPI 指令** | 关键 | 必须使用完整厂家序列以激活 In-cell 触摸模块。 |
| **I2C 复用** | 核心 | 排除 GMAC1 冲突，确保 `pinmux-pins` 指向 `i2c5`。 |
| **设备树属性** | 细节 | 使用 `power-supply` 而非 `vdd-supply` 提高驱动兼容性。 |

**注意**：在不修改设备树的情况下，手动拉高电源 GPIO 和复位 GPIO 是验证硬件链路是否正常的唯一可靠手段。只有 `i2cdetect` 看到地址且不报 `timeout`，后续的软件适配才有意义。