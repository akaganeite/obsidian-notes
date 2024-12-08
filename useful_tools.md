## RISCV-GNU-TOOL

[riscv-gnu-toolchain - XiangShan 官方文档 (xiangshan-doc.readthedocs.io)](https://xiangshan-doc.readthedocs.io/zh-cn/latest/compiler/gnu_toolchain/)

```
cmake ccache ninja-build cmake-curses-gui libxml2-utils ncurses-dev curl git doxygen device-tree-compiler u-boot-tools python3-dev python3-pip python-is-python3 protobuf-compiler python3-protobuf
```

## git

https://github.com/jesseduffield/lazygit
https://www.youtube.com/watch?v=CPLdltN7wgE&ab_channel=JesseDuffield



## ubuntu/clash-dashboard



[192.168.104.61:9090 - yacd](http://192.168.104.61:9090/ui/#/proxies)







[Rust to Assembly: Understanding the Inner Workings of Rust (eventhelix.com)](https://www.eventhelix.com/rust/)

## zsh

[Linux 全局安装配置 zsh + oh-my-zsh - sysin | SYStem INside | 软件与技术分享](https://sysin.org/blog/linux-zsh-all/)

## riscv中断

[详解RISC v中断 - LightningStar - 博客园 (cnblogs.com)](https://www.cnblogs.com/harrypotterjackson/p/17548837.html#_label13)

[6.S081——补充材料——RISC-V架构中的异常与中断详解_risc-v 中断设计-CSDN博客](https://blog.csdn.net/zzy980511/article/details/130642258)



## GDB

[【笔记】rCore (RISC-V)：GDB 使用记录 | 苦瓜小仔 (zjp-cn.github.io)](https://zjp-cn.github.io/posts/rcore-gdb/)

## rust配置工具链

[[Rust\] 嵌入式 riscv64 Rust 开发环境搭建_risc rust-CSDN博客](https://blog.csdn.net/wangyijieonline/article/details/130363131)



## readelf&&objdump

## rustc target

[Creating a custom target - The Embedonomicon (rust-embedded.org)](https://docs.rust-embedded.org/embedonomicon/custom-target.html)



1. get rid of feature panic-handler, print, and system-alloc, we have our own implementation
2. keep only 'fde-static and' 'fde-gnu-eh-frame-hdr' feature, these two options are enough for unwinding
3. keep all the arch related code,modify 
4. Implement backtrace which will print the PC val of each call stack 
5. add unwind_test in cfg(kernel_test)



unwind: fork the open source lib 'unwinding'

1.open source repo:https://github.com/nbdd0121/unwinding.git
2.this commit forks the src/ dir of the repo
3.our unwinding code is based on this repo

Signed-off-by xiaobei zhang <zhangxiaobei@iie.ac.cn>







DISK=build/aarch64/minimal/harddrive.img 

MOUNT_DIR=/mnt/efi_boot 

DTB_DIR=$MOUNT_DIR/dtb/broadcom 

WORKPLACE=/home/zhangxb/redox/tryredox 

DTS=$WORKPLACE/redox_firmware/platform/raspberry_pi/rpi3/bcm2837-rpi-3-b-plus.dts

dtc -I dts -O dtb $DTS > ./bcm2837-rpi-3-b.dtb

cp bcm2837-rpi-3-b.dtb



Im trying to run redox with raspberry pi 3b + on qemu. But the kernel panics while initializing irq controller? Is there anything i can do to fix this?

![截屏2024-11-11 11.35.33](./assets/截屏2024-11-11 11.35.33.png)

Here is some files i modified:

`.config`

```
PODMAN_BUILD?=0
ARCH?=aarch64
CONFIG_NAME?=minimal
CONFIG?=minimal
BOARD?=raspi3bp
```

`redox_firmware/platform/raspberry_pi/rpi3/bcm2837-rpi-3-b-plus.dts` 

- i added 2 lines in soc{}

```
  soc {
#address-cells = <1>;
#size-cells = <1>;
      ranges = <0x7e000000 0x3f000000 0x1000000>,
          <0x40000000 0x40000000 0x00001000>;
  };
```



aa





++ for bin in "${BINS[@]}"
++ echo -p bcm2835-sdhcid

+/home/zhangxb/redox/tryredox/redox/cookbook/target/release/cookbook_redoxer build --release --manifest-path /home/zhangxb/redox/tryredox/redox/cookbook/recipes/core/drivers/source/Cargo.toml -p acpid -p fbcond -p inputd -p lived -p nvmed -p pcid -p vesad -p bcm2835-sdhcid





   Compiling bcm2835-sdhcid v0.1.0 (/home/zhangxb/redox/tryredox/redox/cookbook/recipes/core/drivers/source/storage/bcm2835-sdhcid)

cp -v target/aarch64-unknown-redox/release/bcm2835-sdhcid /home/zhangxb/redox/tryredox/redox/cookbook/recipes/core/drivers-initfs/target/aarch64-unknown-redox/stage.tmp/lib/drivers

'target/aarch64-unknown-redox/release/bcm2835-sdhcid' -> '/home/zhangxb/redox/tryredox/redox/cookbook/recipes/core/drivers-initfs/target/aarch64-unknown-redox/stage.tmp/lib/drivers/bcm2835-sdhcid'

```
Compiling bcm2835-sdhcid v0.1.0 (/home/zhangxb/redox/tryredox/redox/cookbook/recipes/core/drivers/source/storage/bcm2835-sdhcid)

cp -v target/aarch64-unknown-redox/release/bcm2835-sdhcid /home/zhangxb/redox/tryredox/redox/cookbook/recipes/core/drivers-initfs/target/aarch64-unknown-redox/stage.tmp/lib/drivers

'target/aarch64-unknown-redox/release/bcm2835-sdhcid' -> '/home/zhangxb/redox/tryredox/redox/cookbook/recipes/core/drivers-initfs/target/aarch64-unknown-redox/stage.tmp/lib/drivers/bcm2835-sdhcid'
```

