





cmake .. \    -DCMAKE_TOOLCHAIN_FILE=../toolchain-arm64.cmake \    -DCMAKE_INSTALL_PREFIX=/usr/local/sdl2-arm64 \    -DSDL_PULSEAUDIO=OFF \   -DSDL_SNDIO=OFF \    -DSDL_WAYLAND=OFF \    -DSDL_DIRECTFB=OFF \    -DSDL_OPENGLES=OFF\ -DSDL_UNIX_CONSOLE_BUILD=ON  \ -DSDL_DRM=OFF \    -DSDL_GBM=OFF

```
cmake .. \
    -DCMAKE_TOOLCHAIN_FILE=../toolchain-arm64.cmake \
    -DCMAKE_INSTALL_PREFIX=/usr/local/sdl2-arm64 \
    -DSDL_PULSEAUDIO=OFF \
    -DSDL_SNDIO=OFF \
    -DSDL_WAYLAND=OFF \
    -DSDL_DIRECTFB=OFF \
    -DSDL_OPENGLES=OFF \
    -DSDL_UNIX_CONSOLE_BUILD=ON \ 
    -DSDL_DRM=OFF \    
    -DSDL_GBM=OFF
```





boot cmd:

```
load mmc 0:2 0x80000 kernel8.img
go 0x80000
```





## redox

mmcnr@7e300000 {
                        compatible = "brcm,bcm2835-mmc", "brcm,bcm2835-sdhci";
                        reg = <0x7e300000 0x00000100>;
                        interrupts = <0x00000002 0x0000001e>;
                        clocks = <0x00000008 0x0000001c>;
                        dmas = <0x0000000c 0x0000000b>;
                        dma-names = "rx-tx";
                        brcm,overclock-50 = <0x00000000>;
                        non-removable;
                        status = "okay";
                        pinctrl-names = "default";
                        pinctrl-0 = <0x0000001b>;
                        bus-width = <0x00000004>;
                        #address-cells = <0x00000001>;
                        #size-cells = <0x00000000>;
                        phandle = <0x00000030>;
                        wifi@1 {
                                reg = <0x00000001>;
                                compatible = "brcm,bcm4329-fmac";
                                phandle = <0x0000008a>;
                        };
                };



mmc@7e300000 {
                        compatible = "brcm,bcm2835-mmc", "brcm,bcm2835-sdhci";
                        reg = <0x7e300000 0x00000100>;
                        interrupts = <0x00000002 0x0000001e>;
                        clocks = <0x00000008 0x0000001c>;
                        status = "disabled";
                        dmas = <0x0000000c 0x0000000b>;
                        dma-names = "rx-tx";
                        brcm,overclock-50 = <0x00000000>;
                        pinctrl-names = "default";
                        pinctrl-0 = <0x00000014>;
                        bus-width = <0x00000004>;
                        phandle = <0x0000002f>;
                };



mmc@7e202000 {
                        compatible = "brcm,bcm2835-sdhost";
                        reg = <0x7e202000 0x00000100>;
                        interrupts = <0x00000002 0x00000018>;
                        clocks = <0x00000008 0x00000014>;
                        status = "okay";
                        dmas = <0x0000000c 0x2000000d>;
                        dma-names = "rx-tx";
                        bus-width = <0x00000004>;
                        brcm,overclock-50 = <0x00000000>;
                        brcm,pio-limit = <0x00000001>;
                        firmware = <0x00000006>;
                        pinctrl-names = "default";
                        pinctrl-0 = <0x0000000d>;
                        phandle = <0x0000002e>;
                };zc





```
import matplotlib.pyplot as plt
import numpy as np

# 数据
tasks = [
    "Task 1",
    "Task 2",
    "Task 3",
    "Task 4 (long)"
]
durations = [0.4, 0.3, 0.8, 1.5]  # 每个任务的持续时间
start_times = [0]  # 初始时间
for i in range(1, len(durations)):
    start_times.append(start_times[i - 1] + durations[i - 1])

# 绘制图
fig, ax = plt.subplots(figsize=(8, 4))

# 绘制柱状图（短柱子 0.0-1.5 范围）
for i, (start, duration) in enumerate(zip(start_times[:-1], durations[:-1])):  # 不绘制最后一个长柱子
    ax.barh(i, duration, left=start, color="lightgray", edgecolor="black",height=1)

# 绘制长柱子（1.5 开始）
ax.barh(len(tasks) - 1, durations[-1], left=1.5, color="lightgray", edgecolor="black",height=1)

# 设置横坐标范围
ax.set_xlim(0, 3)  # 总范围为 0-3

# 绘制断轴符号 "//"（1.5-3 范围）
d = 0.02  # 斜杠符号大小
kwargs = dict(transform=ax.transAxes, color='k', clip_on=False)
ax.plot((0.5 - d, 0.5 + d), (-d, +d), **kwargs)  # 左侧斜杠
ax.plot((0.5 - d, 0.5 + d), (1 - d, 1 + d), **kwargs)

# 自定义 x 轴刻度（0.0-1.5 的范围以 0.2 间隔）
ax.set_xticks(np.arange(0, 1.6, 0.2))
ax.tick_params(axis="x", which="major", labelsize=10)

# 删除 1.5-3.0 范围的刻度
ax.spines['right'].set_visible(False)
ax.spines['top'].set_visible(False)
ax.set_xticks([], minor=True)
ax.set_yticks(range(len(tasks)))
ax.set_yticklabels(tasks)

# 添加标题和标签
ax.set_xlabel("Time (s)", fontsize=12)
ax.set_title("Task Execution with Broken Axis", fontsize=14)

# 调整布局
plt.tight_layout()

# 保存并显示
plt.savefig("output.png")
plt.show()

```

