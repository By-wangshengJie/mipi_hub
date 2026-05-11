这是一份基于我们本次排错和开发过程总结的**《RK3588开发板第三方触摸屏（sec_ts）适配与排错指南》**。

这份指南不仅适用于本次的三星 `sec_ts` 屏幕，其**“自底向上”的排错逻辑和驱动移植方法**同样适用于任何第三方 I2C 触摸屏（如汇顶、敦泰等）在 Linux 平台上的适配。

---

# RK3588 触摸屏（sec_ts）适配与排错全纪录指南

## 第一阶段：硬件连接与设备树（DTS）配置

### 1. 硬件引脚确认
首先需要通过万用表或原厂图纸确认触摸屏的以下 4 根核心引脚：
*   **VCC/GND**：供电（通常为 3.3V）。
*   **SCL / SDA**：I2C 通信引脚（例如连接到 RK3588 的 `I2C5`）。
*   **INT**：中断引脚（例如连接到 `GPIO3_C0`）。
*   **RST**：复位引脚（可选，若无则不接，例如 `GPIO3_C1`）。

在板子上执行 `i2cdetect -y 5`，如果能扫到设备地址（如 `0x48`），说明 I2C 硬件通信正常。

### 2. 设备树配置
在内核设备树（如 `rk3588-evb.dtsi`）中，挂载 I2C 设备节点并配置引脚和电源：
```dts
&i2c5 {
	status = "okay";
	clock-frequency = <400000>;

	SEC_TS@48 {
		compatible = "sec,sec_ts";  /* 核心：与驱动匹配的标识 */
		reg = <0x48>;
		
		interrupt-parent = <&gpio3>;
		interrupts = <RK_PC0 IRQ_TYPE_LEVEL_LOW>; /* 配置 INT 触发电平 */
		reset-gpios = <&gpio3 RK_PC1 GPIO_ACTIVE_LOW>;
		
		tp-supply = <&vcc3v3_lcd_dsi0>; /* 关联电源节点 */
		status = "okay";
	};
};
```

---

## 第二阶段：排错思路（当触摸屏无反应时）

**核心原则：不要靠猜测，用底层节点数据说话。**

如果烧录设备树后触摸无反应，请依次执行以下命令排查：

1.  **检查驱动是否挂载到 I2C 设备：**
    `ls -l /sys/bus/i2c/devices/5-0048/driver`
    *如果提示不存在，说明驱动未加载或未匹配。*
2.  **检查内核是否有该驱动代码：**
    `ls /sys/bus/i2c/drivers/`
    *如果在列表中**找不到**你设备树里写的 `sec_ts`，说明内核源码中根本没有这个驱动！必须手动移植。*
3.  **检查中断是否工作：**
    `cat /proc/interrupts | grep touch`
    *如果触摸屏幕时数值增加，说明硬件和底层中断通了，问题在上层输入子系统。*

---

## 第三阶段：第三方驱动移植（核心步骤）

当确认内核源码中没有厂家的触摸屏驱动（如三星的 S6D6FT0 触控 IC）时，需要从 GitHub 或屏幕厂家获取驱动源码包（`.c`, `.h`, `Kconfig`, `Makefile` 等）。

### 1. 放入内核源码目录
将下载的 `sec_ts` 文件夹整个复制到内核的输入子系统目录下：
```bash
cp -r sec_ts/ /mnt/armbian/kernel-rk3588/drivers/input/touchscreen/
```

### 2. 将驱动接入内核编译系统
编辑 `drivers/input/touchscreen/Kconfig`，在文件末尾（`endif` 前一行）添加：
```text
source "drivers/input/touchscreen/sec_ts/Kconfig"
```

编辑 `drivers/input/touchscreen/Makefile`，在文件末尾添加：
```makefile
obj-y += sec_ts/
```

### 3. 配置并开启驱动
在内核根目录执行：
```bash
make menuconfig
```
按 `/` 搜索 `SEC_TS`，找到该触摸屏选项，按 `Y` 键将其设置为 `[*]`（内建进内核），保存退出。

---

## 第四阶段：解决内核版本 API 冲突（编译报错处理）

第三方驱动往往是为老版本内核（如 Linux 4.x/5.10）编写的。在较新的内核（如 Linux 6.1+）中编译时，极易遇到 **I2C API 返回值类型不兼容** 的报错。

**典型报错：**
> `error: initialization of ‘void (*)(struct i2c_client *)’ from incompatible pointer type ‘int (*)(struct i2c_client *)’`

**解决办法：**
新版内核要求 I2C 驱动的 `remove` 函数必须是无返回值的 `void` 类型。打开报错的 C 文件（如 `sec_ts_main.c`），找到 `remove` 函数并修改：

**修改前（老旧代码）：**
```c
static int sec_ts_remove(struct i2c_client *client) {
    // ... 代码逻辑 ...
    return 0; 
}
```
**修改后（适配新内核）：**
```c
static void sec_ts_remove(struct i2c_client *client) {
    // ... 代码逻辑不变 ...
    // return 0;  // 删掉或注释掉 return
}
```
修改完毕后，重新执行 `make` 编译内核并烧录到开发板即可。

---

## 第五阶段：功能验证与系统操作

烧录新内核重启后，使用 `evtest` 工具验证底层多点触摸（MT）协议是否正常：
```bash
sudo apt-get install evtest
sudo evtest
```
选中触摸屏对应的 event 编号。如果用双指触摸屏幕时，终端能稳定输出 `ABS_MT_SLOT`（追踪不同手指）、`ABS_MT_POSITION_X/Y`（坐标）以及 `ABS_MT_TOUCH_MAJOR`（按压面积），则说明底层驱动移植 **100% 成功**。

### Ubuntu 桌面系统下的触摸操作提示
底层打通后，Ubuntu（通过 `libinput`）会自动接管触摸屏，支持原生手势：
*   **左键/点击**：单指轻触。
*   **右键**：单指长按屏幕同一位置 1~2 秒后松开（部分桌面环境可能需要开启“辅助功能 -> 模拟次要点击”）。
*   **拖动**：单指按住目标 0.5 秒后滑动。
*   **缩放**：双指捏合/张开。

*(注：若发现触摸方向与显示方向反了/倒转，无需修改底层驱动，在 Ubuntu 中添加一条对应的 Udev 坐标矩阵校准规则即可。)*