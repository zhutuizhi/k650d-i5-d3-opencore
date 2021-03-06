- 20200606 添加AR9565的驱动方法放到miscellaneous，网上找的k650d原理图，和N156HHE-GA1产品手册也放在miscellaneous中。

- N156HHE-GA1这块屏确实和k650d不太兼容，出现闪屏时，重插屏线，能恢复正常，然后在win下使用10天左右（不进macOS）后会又出现不定时闪下黑线的问题，如果进macOS后再进win，闪的间隔时间大幅变短，也变的更厉害了，看来macOS对N156HHE-GA1这块屏也有影响，这就非常难受了。对照了屏线接口定义，确实是完全对的上，但看屏库资料N156HHE-GA1是eDP1.3，原来的屏是eDP1.2，不知道是不是这个问题。也猜测可能是屏线问题，但兼容144hz的屏线，某宝150-180一条，瞬间不想折腾了。
打算装回原来的屏，滚回老系统。这里也不打算再更新了。
- 20200511   
使能PowerTimeOutKernelPanic，修复睡眠唤醒内核崩溃（原因AppleHDAHDMI_DPDriver::setPowerState）。
- 20200510    
10.15.4下ApplePS2SmartTouchPad驱动有时会不加载，导致键盘不可用，用VoodooPS2Controller代替了，触控板无法驱动。

# 2020.5.7更新
- 更换屏幕为N156HHE-GA1，解决10.13以上轻微闪屏问题。已注入该屏修改过的EDID（屏不同EDID不同，应对应修改）。20200518 N156HHE-GA1这块屏似乎和k650d不是很兼容，使用了一段时间后发现，无论是在win下还是mac下都会发生不定时闪一下屏的情况。尴尬。
- 系统升级为10.15.4，opencore 升级0.5.8. 
- AppleALC用自己修改的layout-id 18。
- wifi换用bcm4322，替换10.14的IO80211Family。

# k650d-i5-d3-opencore

分享一个opencore的配置，仅供参考，系统macOS 10.12.6。装的是双硬盘4系统，win10+macos+deepin+freebsd,通过Misc-Entries添加各系统引导（无法直接用，建议Misc-Security-ScanPolicy选0），Misc-Security-ScanPolicy选0时，会扫描出一些无用项，比如扫描出来的windows无法引导的。

## 电脑配置

| 硬件    | 详细信息                           |
| -------- | ---------------------------------- |
| 处理器   | Intel Core i5-4210M @ 2.60GHz 双核 |
| 显卡     | Intel HD Graphics 4600 +950M（屏蔽）   |
| 显示器   | 三星 SDC4852 （Samsung LTN156HL07401）   |
| 声卡     | VIA VT1802P          |
| 以太网卡  | Realtek 8411B     |
| Wi-Fi   | AR9565+BT4.0    |



## 概述

- whatevergreen驱动HD4600，屏蔽950M，ig-platform-id为0x04160000，系统10.13及以上，需要修改EDID解决开机花屏，但会有轻微闪屏现象，目测是SDC4852显示器的缘故，同机型京东方的屏不会。所以停留在10.12.6。若使用0x0A260006，切换分辨率或睡眠会黑屏，但这个ID最大亮度更高，当然10.13后这个ID无用了。
- AppleALC驱动声卡，使用id为65，但自己修改了configdata，节点路径，可惜实际效果与原版无甚差别。内置麦克风录取声音略小，线路输入输入音量调最大能录到大声音，同时杂音很大。
- ApplePS2SmartTouchPad驱动触控板，驱动来自网络，但好像是debug版，-v有大量log。
- wi-fi ar9565用修改过的AirPortAtheros40.kext直接替换IO80211Family下的驱动，所以不体现在EFI下，自带蓝牙需切windows后再切macos才能正常使用，睡眠唤醒后wifi和蓝牙都不正常，似乎因为蓝牙会秒醒，无线网卡用BCM4322不会秒醒。10.14及以上应整个替换IO80211Family. [AirPortAtheros40](https://www.insanelymac.com/forum/topic/312045-atheros-wireless-driver-os-x-101112-for-unsupported-cards/?do=findComment&comment=2509900)。
- SMCBatteryManager驱动电池，由于dsdt电池部分没有超过8位的，所以电池驱动可以直接使用，若用APCIBatterManager，电量100%时有时显示不正常，有时会报没电。所以SMCBatteryManager更好。不放电池驱动时，可以在偏好设置-键盘-快捷键设置亮度调节快捷键为F8,F9. 改完后再放电池驱动，快捷键仍然有效。
- 已解锁CFG Lock，参考了 [云屋小站](https://www.misonsky.cn/115.html) , 但kernel-Quirks-AppleXcpmCfgLock仍然打了Yes。


## SSDT简述

- SSDT-PLUG.aml：XCPM，CPU电源管理。效果等同Clover的ACPI-SSDT-Generate-PluginType；
- SSDT-PNLF.aml：笔记本亮度调节，效果等同clover的ACPI-DSDT-Fixes-AddPNLF+Devices-SetIntelBacklight；
- SSDT-LPC.aml： 加载AppleLPC电源管理，效果等同clover的ACPI-DSDT-Fixes-FakeLPC；
- SSDT-HPET_RTC_TIMR-fix.aml：声卡IRQ修复，不加的话，AppleALC（开-alcdbg）会一直卡AppleALC       alc: @ failed to find IOHDACodecVendorID, retrying xxx（无限增加的次数）
- SSDT-SBUS.aml：效果等同clover的ACPI-DSDT-Fixes-FixSBUS


## opencore使用感受

配置起来好麻烦，一度kernel panic。使用起来好像和clover没什么区别。win下主板识别成苹果。mac下关于本机没有clover版本什么的。相对来说，clover新安装系统调试更合适些，毕竟很多操作可以在引导界面修改。而opencore目前做不到。


## 鸣谢

- [黑果小兵](https://github.com/daliansky/  https://blog.daliansky.net/)
- [xjn](https://blog.xjn819.com/?author=1)
- [RehabMan](https://github.com/RehabMan)
- [Acidanthera](https://github.com/acidanthera)
- [daggeryu](https://github.com/daggeryu)
- [ivothgle](https://github.com/ivothgle/)
- [Mieze](https://github.com/Mieze/RTL8111_driver_for_OS_X)
- [chunnann](https://www.insanelymac.com/forum/topic/312045-atheros-wireless-driver-os-x-101112-for-unsupported-cards/?do=findComment&comment=2509900)
- [Mison](https://www.misonsky.cn/115.html)
