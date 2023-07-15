# 折腾的细节

### Insterested things

- 词典笔使用的是RGB信号接口的屏幕，而不是在手机平板常见的MIPI接口 芯片为`ST7789V2`

  ```c
  panel {
      compatible = "simple-panel";
      ...
      rockchip,output = "rgb";
      rgb-mode = "s888";
      status = "okay";
      rockchip,cmd-type = "mcu";
      ...
  };
  ```

- 充电电流限制在500mA，可以解锁到2000mA，实测使用乐视24w充电头最高1600mA,充电时间只需1h以内

  ```c
  charger {
      ...
      max_chrg_current = <500>;
      ...
  };
  ```

  `max_chrg_current`的数值改`2000`即可（参考自[rk3326-evb-lp3-v10-linux.dts](https://github.com/rockchip-linux/kernel/blob/develop-4.4/arch/arm64/boot/dts/rockchip/rk3326-evb-lp3-v10-linux.dts)）

- cpu实际上支持1.3G-1.5G的频率，但是也被屏蔽

  ```c
  opp-1248000000 {
      ...
      status = "disabled";
  };
  
  opp-1296000000 {
      ...
      status = "disabled";
  };
  
  opp-1416000000 {
      ...
      status = "disabled";
  };
  
  opp-1512000000 {
      ...
      status = "disabled";
  };
  ```

- vpu_combo解码单元也被屏蔽...

  ```c
  vpu_combo {
      compatible = "rockchip,vpu_combo";
      ...
      status = "disabled";
  };
  ```

### Tips

- firefly提供的[linux sdk](https://www.t-firefly.com/doc/download/67.html) 直接`repo sync -l` 出来的代码过久，需要更新

  - rkbin 瑞芯微提供的非开源工具集 https://github.com/rockchip-linux/rkbin

  - uboot 引导loader ,需要使用https://gitlab.com/firefly-linux/u-boot的源

    - `px30/firefly`分支
    - 配置文件`evb-rk3326`

    使用[瑞芯微官方](https://github.com/rockchip-linux/u-boot)的会无法使用按键进Loader模式，只能手动短接Maskrom触点

  - kernel linux内核 ,kernel firefly自带的无法启动，会报arm trust firmware EL3错误，需使用瑞芯微官方源https://github.com/rockchip-linux/kernel `develop-4.4`分支

### Buildroot Fix Build Error (仍未编译出来，放弃了)

#### 编译buildroot

开启多线程编译以加速构建

```bash
source envsetup.sh rk3326_64
make BR2_JLEVEL=8 V=1 #8是CPU线程数 V=1 详细信息显示
```


- Firefly自带的buildroot 软件包buildroot 更新1.33.1后出错 

  **原因:脚本sed命令参数用错了**

  ```bash
  /usr/bin/sed -i -e "/\\<CONFIG_NOMMU\\>/d" 
  /usr/bin/sed: no input files
  ```

  Fix:`buildroot/package/pkg-utils.mk`

  ```bash
  define KCONFIG_ENABLE_OPT # (option, file)
  	$(SED) "/\\<$(1)\\>/d" $(2)
  	echo '$(1)=y' >> $(2)
  endef
  
  define KCONFIG_ENABLE_OPT # (option, file)
  	$(SED) "/\\<$(1)\\>/d" $(2)
  	echo '$(1)=y' >> $(2)
  endef
  
  define KCONFIG_SET_OPT # (option, value, file)
  	$(SED) "/\\<$(1)\\>/d" $(3)
  	echo '$(1)=$(2)' >> $(3)
  endef
  
  define KCONFIG_DISABLE_OPT # (option, file)
  	$(SED) "/\\<$(1)\\>/d" $(2)
  	echo '# $(1) is not set' >> $(2)
  endef
  ```

  修改成

  ```bash
  KCONFIG_DOT_CONFIG = $(strip \
  	$(if $(strip $(1)), $(1), \
  		$($(PKG)_BUILDDIR)/$($(PKG)_KCONFIG_DOTCONFIG) \
  	) \
  )
  
  # KCONFIG_MUNGE_DOT_CONFIG (option, newline [, file])
  define KCONFIG_MUNGE_DOT_CONFIG
  	$(SED) '/^\(# \)\?$(strip $(1))\>/d' $(call KCONFIG_DOT_CONFIG,$(3)) && \
  	echo '$(strip $(2))' >> $(call KCONFIG_DOT_CONFIG,$(3))
  endef
  
  # KCONFIG_ENABLE_OPT (option [, file])
  # If the option is already set to =m or =y, ignore.
  define KCONFIG_ENABLE_OPT
  	$(Q)if ! grep -q '^$(strip $(1))=[my]' $(call KCONFIG_DOT_CONFIG,$(2)); then \
  		$(call KCONFIG_MUNGE_DOT_CONFIG, $(1), $(1)=y, $(2)); \
  	fi
  endef
  # KCONFIG_SET_OPT (option, value [, file])
  KCONFIG_SET_OPT     = $(Q)$(call KCONFIG_MUNGE_DOT_CONFIG, $(1), $(1)=$(2), $(3))
  # KCONFIG_DISABLE_OPT  (option [, file])
  KCONFIG_DISABLE_OPT = $(Q)$(call KCONFIG_MUNGE_DOT_CONFIG, $(1), $(SHARP_SIGN) $(1) is not set, $(2))
  ```

- 需更新的软件包列表，以firefly gitlab上为参考,in Archlinux

  - busybox 1.7 -> busybox 1.33.2

  - glibc 2.29 -> 2.32

  - gstreamer1 1.14.4 -> 1.16.3 (替换成buildroot官方2020.03)

  `buildroot/configs/rockchip/video_gst.config`

  血压上来了，先关掉`BR2_PACKAGE_GST1_PLUGINS_BAD_PLUGIN_MPEG2ENC`和相关代码，先不管了

  - lmbench 直接删掉，没啥用还报错

  - qt5,qt5cinex 换2023.3 buildroot

    qtbase里面的 `syncqt.pl `手动复制到host/bin

  - perl

- 补全gpu驱动 [libmail](https://github.com/tsukumijima/libmali-rockchip) or 去gitlab更新

  **Fix: 将lib里面的文件放到`external/libmali/lib/aarch64-linux-gnu/`里面即可**

  原因:firefly度盘给的sdk(`px30_linux_release_20210304.tgz`) **文件不全**

  ```bash
  Build machine cpu family: x86_64
  Build machine cpu: x86_64
  Host machine cpu family: aarch64
  Host machine cpu: cortex-a35
  Target machine cpu family: aarch64
  Target machine cpu: cortex-a35
  
  output/rockchip_rk3326_64/build/libmali-develop/meson.build:23:0: ERROR: Problem encountered:
  ```

  
