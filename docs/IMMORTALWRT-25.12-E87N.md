# 在 ImmortalWrt 25.12 中集成 EdgePi E87N 风扇控制

本文适用于：

- EdgePi E87N（MT7987A）；
- ImmortalWrt 25.12，`mediatek/filogic`；
- Linux标准`pwm-fan`驱动；
- `luci-app-Airpifanctrl`。

## 1. DTS要求

E87N DTS需要包含PWM1 pinctrl，并启用`pwm-fan`：

```dts
&fan {
    pwms = <&pwm 1 50000 0>;
    status = "okay";
};

&pio {
    pwm_fan_pins: pwm-fan-pins {
        mux {
            function = "pwm";
            groups = "pwm1_0";
        };
    };
};

&pwm {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pwm_fan_pins>;
};
```

ChenMercy/immortalwrt的`mt7987a-edgepi-e87n.dts`已经包含该配置：

<https://github.com/ChenMercy/immortalwrt/tree/openwrt-25.12>

## 2. 将软件包加入源码

```sh
cd /path/to/immortalwrt
mkdir -p package/mtk/applications
git clone https://github.com/ChenMercy/openwrt-luci-app-Airpifanctrl.git \
  package/mtk/applications/luci-app-Airpifanctrl
```

确认Makefile语法完整：

```sh
tail -n 1 package/mtk/applications/luci-app-Airpifanctrl/Makefile
```

应显示：

```make
$(eval $(call BuildPackage,luci-app-Airpifanctrl))
```

确认脚本权限：

```sh
chmod 755 \
  package/mtk/applications/luci-app-Airpifanctrl/root/etc/init.d/Airpifanctrl \
  package/mtk/applications/luci-app-Airpifanctrl/root/usr/bin/fancts.sh
```

## 3. 选择内核驱动和LuCI包

```sh
printf '%s\n' \
  'CONFIG_PACKAGE_kmod-hwmon-pwmfan=y' \
  'CONFIG_PACKAGE_luci-app-Airpifanctrl=y' >> .config
make defconfig

grep -E '^CONFIG_PACKAGE_(kmod-hwmon-pwmfan|luci-app-Airpifanctrl)=y' .config
```

不需要添加私有`kmod-Airpi-gpio-fan`。E87N使用Linux标准`pwm-fan`驱动。

## 4. 单独编译软件包

```sh
export FORCE_UNSAFE_CONFIGURE=1
make package/luci-app-Airpifanctrl/clean
make package/luci-app-Airpifanctrl/compile V=s
```

ImmortalWrt 25.12使用APK软件包系统。检查生成文件：

```sh
find bin/packages -type f -name '*Airpifanctrl*.apk'
```

## 5. 编译完整E87N固件

```sh
make menuconfig
```

选择：

```text
Target System: MediaTek Ralink ARM
Subtarget: Filogic 8x0 (MT798x)
Target Profile: EdgePi E87N
```

然后编译：

```sh
export FORCE_UNSAFE_CONFIGURE=1
make -j$(nproc) V=s
```

固件输出：

```text
bin/targets/mediatek/filogic/*edgepi_e87n*squashfs-sysupgrade.bin
```

检查manifest：

```sh
grep -E 'Airpifanctrl|hwmon-pwmfan' \
  bin/targets/mediatek/filogic/*edgepi_e87n*.manifest
```

## 6. 实机验收

刷机后执行：

```sh
ls -l /sys/class/hwmon
find /sys/devices/platform/pwm-fan -name pwm1 -print
cat /etc/fanvall
/etc/init.d/Airpifanctrl enable
/etc/init.d/Airpifanctrl restart
pgrep -a fancts.sh
```

已验证的E87N路径为：

```text
/sys/devices/platform/pwm-fan/hwmon/hwmon2/pwm1
```

读取当前PWM：

```sh
cat /sys/devices/platform/pwm-fan/hwmon/hwmon2/pwm1
```

默认`/etc/fanvall=2`，预期PWM为`192`。

确认thermal控制已禁用：

```sh
cat /sys/class/thermal/thermal_zone0/mode
```

预期：

```text
disabled
```

## 7. 注意事项

- `hwmon2`编号由探测顺序决定；若设备编号变化，应先用`find /sys/devices/platform/pwm-fan -name pwm1`确认实际路径。
- 不要同时让thermal framework和`fancts.sh`控制同一个PWM通道。
- `/etc/fanvall`必须保留换行，默认值为`2`。
- 风扇不转时先检查DTS、`kmod-hwmon-pwmfan`和`/sys/devices/platform/pwm-fan`，不要直接把问题归因于LuCI。
