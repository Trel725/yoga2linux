# yoga2linux
Linux for Lenovo Yoga Tab 2

# Working hardware
1. BQ27xxx battery contrller. Needed to display battery charge. Added to the DSDT, should work out of the box or need loading of bq27xxx_battery.
2. BQ24296 charge controller. Needed to switch USB OTG port between device and host modes. Added to DSDT as BQ24196, which is not 100 % compatible (some register differences according to datasheets), nevertheless switching of device/host modes is same in both chips. Should work out of the box or need loading of bq24190_charger module. After module is loaded, following command should switch USB to host mode:  
`echo 2 > /sys/class/power_supply/bq24190-charger/f_chg_config`
3. Synaptics I2C touchscreen. Added to DSDT but won't work unless kernel is patched. Due to the Baytrail GPIO interrupt architecture some of the pins (including touchscreen one) can not be used as IRQ sources (see following patch https://www.fclose.com/linux-kernels/718144/pinctrl-baytrail-do-not-add-all-gpios-to-irq-domain-linux-4-10/).  
Somehow during initialization BIT_DIRECT_IRQ_EN is set for the touchscreen GPIO and thus it is excluded from the system IRQ domain.
```C
if (value & BYT_DIRECT_IRQ_EN) {
			clear_bit(i, gc->irq.valid_mask);
dev_dbg(dev, "excluding GPIO %d from IRQ domain\n", i);
```
To overcome this I've used a very dirty ugly hack to prevent touchscreen GPIO from excluding (available in pinctrl.patch)
```C
if (value & BYT_DIRECT_IRQ_EN) {

	if(i!=12){
		clear_bit(i, gc->irq.valid_mask);
		dev_dbg(dev, "excluding GPIO %d from IRQ domain\n", i);
	}
	else{
		byt_gpio_clear_triggering(vg, i);
		dev_dbg(dev, "keeping GPIO %d\n", i);
	}
  ```
  If it is possible to achieve this in some more elegant way, please let me know.
  
  4. Wireless module need firmware, shttps://wiki.debian.org/InstallingDebianOn/Asus/T100TAimilar as desribed here: https://wiki.debian.org/InstallingDebianOn/Asus/T100TA
  
# Non working hardware
1. Sound. Device uses WM5102 chip, and there is a patch to support it on Baytrail devices: https://patchwork.kernel.org/patch/9764493/, nevertheless it seems that patch was not included in the mainstream kernel. Probably possible.
2. Screen brightness. According to the debug messages from Android, display uses some mysterious Chineese chip without support in kernel or even information about. Probably impossible.
3. Sensors. This question was not investigated, so unknown.

# Loading of modified DSDT
According to the Arch Wiki (https://wiki.archlinux.org/index.php/DSDT), section Using a CPIO archive.
