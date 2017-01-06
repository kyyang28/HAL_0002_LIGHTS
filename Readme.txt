HAL : 	lights_hal.c

把新文件上传的Ubuntu Android 5.0.2目录，具体位置如下：
	
	hardware/libhardware/modules/lights/lights_hal.c
	hardware/libhardware/modules/lights/Android.mk

Android.mk内容如下：

	LOCAL_PATH := $(call my-dir)

	include $(CLEAR_VARS)

	LOCAL_MODULE := lights.tiny4412

	# HAL module implementation stored in
	# hw/<VIBRATOR_HARDWARE_MODULE_ID>.default.so
	LOCAL_MODULE_RELATIVE_PATH := hw
	LOCAL_C_INCLUDES := hardware/libhardware
	LOCAL_SRC_FILES := lights_hal.c
	LOCAL_SHARED_LIBRARIES := liblog
	LOCAL_MODULE_TAGS := eng

	include $(BUILD_SHARED_LIBRARY)

修改 ：
vendor/friendly-arm/tiny4412/device-tiny4412.mk文件

ifeq ($(BOARD_USES_PWMLIGHTS),false)
PRODUCT_COPY_FILES += \
   $(VENDOR_PATH)/proprietary/lights.tiny4412.so:system/lib/hw/lights.tiny4412.so
endif

改为

ifeq ($(BOARD_USES_PWMLIGHTS),false)
#PRODUCT_COPY_FILES += \
#   $(VENDOR_PATH)/proprietary/lights.tiny4412.so:system/lib/hw/lights.tiny4412.so
endif


编译 :

. setenv

-B表示强制编译
mmm hardware/libhardware/modules/lights -B

/** 
 *	比较proprietary/lights.tiny4412.so和system/lib/hw/lights.tiny4412.so是否一样
 *	用这个命令确保我们提供的lights_hal.c已经编进了system.img文件中
 */
[OPTIONAL] diff vendor/friendly-arm/tiny4412/proprietary/lights.tiny4412.so out/target/product/tiny4412/system/lib/hw/lights.tiny4412.so

make snod

./gen-img.sh


使用 logcat lights:V *:S 查找 lights是否在system.img中


错误：
130|shell@tiny4412:/ $ logcat lights:V *:S
--------- beginning of main
--------- beginning of system
E/lights  ( 1979): write_string failed to open /sys/class/leds/led1/trigger
E/lights  ( 1979): write_int failed to open /sys/class/leds/led1/delay_on

/sys/class/leds/led1/brightness
/sys/class/leds/led1/trigger
/sys/class/leds/led1/delay_on
/sys/class/leds/led1/delay_off
都有权限不够的问题，需要修改内核的led-class.c和ledtrig-timer.c


ledtrig-timer.c的84行
	
	static DEVICE_ATTR(delay_on, 0644, led_delay_on_show, led_delay_on_store);
	static DEVICE_ATTR(delay_off, 0644, led_delay_off_show, led_delay_off_store);

	改为

	static DEVICE_ATTR(delay_on, 0666, led_delay_on_show, led_delay_on_store);
	static DEVICE_ATTR(delay_off, 0666, led_delay_off_show, led_delay_off_store);

led-class.c的76行

	__ATTR(brightness, 0644, led_brightness_show, led_brightness_store),

	改为

	__ATTR(brightness, 0666, led_brightness_show, led_brightness_store),

led-class.c的79行

	__ATTR(trigger, 0644, led_trigger_show, led_trigger_store),

	改为

	__ATTR(trigger, 0666, led_trigger_show, led_trigger_store),

重新编译内核

	make zImage

烧写新的zImage到tiny4412开发板

