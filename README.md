# luci-app-Airpifanctrl

EdgePi E87N / AirPi PWM风扇控制LuCI插件。

本仓库由原始AirPi风扇控制程序整理而成，适用于带`pwm-fan`设备的MediaTek Filogic固件。EdgePi E87N已验证使用Linux标准`pwm-fan`驱动，不需要私有`kmod-Airpi-gpio-fan`模块。

## 功能

- LuCI风扇模式选择；
- 静音、低速、常规、狂暴和温控模式；
- 默认`/etc/fanvall=2`（常规模式）；
- 启动时禁用`thermal_zone0`对PWM的自动覆盖；
- 支持按SoC温度自动调速；
- 可选读取蜂窝模组温度。

## EdgePi E87N模式

| `/etc/fanvall` | 模式 | PWM |
|---|---|---:|
| `0` | 静音 | 64 |
| `1` | 低速 | 128 |
| `2` | 常规 | 192 |
| `3` | 狂暴 | 255 |
| 其他/空 | 自动温控 | 64–255 |

E87N实机PWM路径：

```text
/sys/devices/platform/pwm-fan/hwmon/hwmon2/pwm1
```

## 加入ImmortalWrt 25.12

完整教程：[`docs/IMMORTALWRT-25.12-E87N.md`](docs/IMMORTALWRT-25.12-E87N.md)

快速步骤：

```sh
cd /path/to/immortalwrt
mkdir -p package/mtk/applications
git clone https://github.com/ChenMercy/openwrt-luci-app-Airpifanctrl.git \
  package/mtk/applications/luci-app-Airpifanctrl

printf '%s\n' \
  'CONFIG_PACKAGE_kmod-hwmon-pwmfan=y' \
  'CONFIG_PACKAGE_luci-app-Airpifanctrl=y' >> .config
make defconfig

export FORCE_UNSAFE_CONFIGURE=1
make -j$(nproc) V=s
```

## 运行检查

```sh
/etc/init.d/Airpifanctrl enable
/etc/init.d/Airpifanctrl restart
cat /etc/fanvall
cat /sys/devices/platform/pwm-fan/hwmon/hwmon2/pwm1
pgrep -a fancts.sh
```

## 致谢

原始程序作者：Manper。

## License

见[`LICENSE`](LICENSE)。
