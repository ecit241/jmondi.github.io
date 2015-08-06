# Enable VIVI V4L2 Test driver on Android

Vivi is the V4L2 virtual test device [see here](http://linuxtv.org/wiki/index.php/VIVI), sometimes useful to test your video/media applications and services.
While it is widespread and commonly found pre-built in your Linux distribution, on Android system enabling VIVI requires a bit of effort, due to its dependencies at build and run-time.

## Problems with VIVI

On kernel 3.14 VIVI is located at 

```
drivers/media/platform/vivi.c
```

As the *Kbuild* file specifies, VIVI depends on *Console Framebuffer (CONFIG_FRAMEBUFFER_CONSOLE)*.
Turns out this is a problem for Android, since enabling console framebuffer prevent Android from booting at all.
It seems this also depends on the platform involved, since the implementation of Android framebuffer conflicts with the console framebuffer one, but only on some platforms.

As you can read, a guy [here](https://groups.google.com/forum/#!topic/android-platform/3Q9VHqefUtw) has been able to have console framebuffer built in and Android running on a Qualcomm platform.

By the way, since I didn't want to investigate on this, and my Jetson board simply hang on *"Starting kernel..."* I decided to hack around this issues and find out why VIVI depends on fb console and for what, and eventually try to remove this dependency.

## Build VIVI without framebuffer dependency

Inspecting symbols, and trying to load vivi.ko on a kernel with no built-in console framebuffer tells us that the only missing symbol is *find_font*, which is a function provided by console fb, that helps retrieve the array of characters you usually see on one of your virtual tty.

Vivi looks for the following font:
```
       const struct font_desc *font = find_font("VGA8x16");
```

Now that we know that, we can simply directly point vivi to the structure that holds the characters descriptions, but we have to tweak a bit the build system in order to have the kernel object for the *font_8x16* built when we build vivi.

First, remove vivi dependency on fb console, so that you can enable it when configuring kernel without enabling fb first

```
diff --git a/drivers/media/platform/Kconfig b/drivers/media/platform/Kconfig
index 9b84303..bb9b04a 100644
--- a/drivers/media/platform/Kconfig
+++ b/drivers/media/platform/Kconfig
@@ -222,7 +222,7 @@ if V4L_TEST_DRIVERS
 config VIDEO_VIVI
        tristate "Virtual Video Driver"
        depends on VIDEO_DEV && VIDEO_V4L2 && !SPARC32 && !SPARC64
-       depends on FRAMEBUFFER_CONSOLE || STI_CONSOLE
+       #depends on FRAMEBUFFER_CONSOLE || STI_CONSOLE
        select FONT_8x16
        select VIDEOBUF2_VMALLOC
        default n
```

Copy the C module that implements the font we want in the same directory where VIVI resides (I'm sure there are better ways to do this, this is the fastest one for sure)

```
cp drivers/video/console/font_8x16.c  drivers/media/platform/
```

And add this to the Makefile

```
diff --git a/drivers/media/platform/Makefile b/drivers/media/platform/Makefile
index 72d1750..2ed3fb5 100644
--- a/drivers/media/platform/Makefile
+++ b/drivers/media/platform/Makefile
@@ -20,7 +20,7 @@ obj-$(CONFIG_VIDEO_OMAP2)             += omap2cam.o
 obj-$(CONFIG_VIDEO_OMAP3)      += omap3isp/
 
 obj-$(CONFIG_VIDEO_VIU) += fsl-viu.o
-obj-$(CONFIG_VIDEO_VIVI) += vivi.o
+obj-$(CONFIG_VIDEO_VIVI) += vivi.o font_8x16.o
 
 obj-$(CONFIG_VIDEO_MEM2MEM_TESTDEV) += mem2mem_testdev.o
```

vivi.c needs some small adjustments as well.. 

```
diff --git a/drivers/media/platform/vivi.c b/drivers/media/platform/vivi.c
index c71a698..58d02bb 100644
--- a/drivers/media/platform/vivi.c
+++ b/drivers/media/platform/vivi.c
@@ -34,6 +34,8 @@
 #include <media/v4l2-event.h>
 #include <media/v4l2-common.h>
 
+#include <linux/font.h>
+extern const struct font_desc font_vga_8x16;
 #define VIVI_MODULE_NAME "vivi"
 
 /* Maximum allowed frame rate
@@ -1490,7 +1492,7 @@ free_dev:
  */
 static int __init vivi_init(void)
 {
-       const struct font_desc *font = find_font("VGA8x16");
+       const struct font_desc *font = &font_vga_8x16;
        int ret = 0, i;
 
        if (font == NULL) {
```

Now configure VIVI to be build as a module with menuconfig (or whatever you use) and then build it.
You will now have both vivi.ko and font_8x16.ko.

Push them to your target, load font then vivi and you should have your virtual device ready for you tests.

```
root@jetson:/data/local/tmp # insmod ./font_8x16.ko
root@jetson:/data/local/tmp # insmod ./vivi.ko
root@jetson:/data/local/tmp # dmesg
.....
<6>[  104.609799] vivi-000: V4L2 device registered as video0
<6>[  104.615137] Video Technology Magazine Virtual Video Capture Board ver 0.8.1 successfully loaded.
```

Vivi has now created a /dev/video0 entry you can interface with.
Enjoy!


