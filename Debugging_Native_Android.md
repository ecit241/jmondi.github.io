# Debugging Native Android application

##Intro 
Android applications debugging is the everyday bread and butter of every Android applications developer.
The Android SDK offers a lot of tools, such as a step-by-step debugger, integrated in both the new *Android Studio* IDE as well as in the previous default development platform, *Eclipse*.
Once the SDK is set-up, and the target device gets correctly identified by the IDE, most of the required setup work is done, you are ready to step in your application code!

A lot of other great tools are useful to trace execution and identify performances issues in your Android applications, tools already built-in in Android such as Systrace, Dumpsys and others. There's plenty of documentation about this tools around, got look for it!

For middleware and platform developers, jumping all day between Java, JNI and C/C++, there's no such a built-in support, and setup of tools and debug utilities has to be performed mostly by hand, tailored to the specific use case.
Sometimes, due to the complexity and the overall effort required for setting things up, it's tempting to take shortcuts and use prints and instrument code with side effects to just follow the execution flow, but this is a time demanding operation as well, which requires lot of compile-load-reboot cycles, which we know are not exactly *cheap* in Android.

This brief tutorial targets a specific use case: *debugging C/C++ libraries with gdb* which, per se, it is not traditionally a complex task embedded world; what is tricky here, in the context of Android middleware devlopment, is that most of the time the code we want to debug is deployed as a library, which is then loaded and run by a Java system component we cannot interrupt and step into easily. Therefore, some tricks are required!

## Target setup
As said debugging will be performed with *gdb* running on the host side, with the Android's built-in *gdbserver* running on target; 

We firstly have to connect gdbserver to the service we are interested to debug. Every native library is (usually) loaded by an upper layer system service, you should identify which service loads your object and find the PID of its process image.

Since we are going to step into the camera HAL, we are interested in knowing the *media service* PID.

```
root@android:/ # ps -epfgrep media
USER     PID   PPID  VSIZE  RSS     WCHAN    PC         NAME 
media     131   1     0      0     ffffffff 00000000 Z mediaserver
```

Once we know the service PID, we can ask gdbserver to attach to its process

```
root@android:/ # gdbserver --attach localhost:2345 131
Attached; pid = 131 Listening on port 2345 Remote debugging from host 127.0.0.1
```

gdbserver will now start listening on the *target's* TCP port 2345.

## Host setup
Once gdbserver has been successfully attached to the system service we are interested in, we should, on the host side, ask *adb* to forward the target port on the host system, where we can connect to.
If your target has a network interface and an IP address assigned, you can connect directly to it, but, most of the times, adb over USB is the default choice for most developers.

We'll forward the target's port on the same port on the host side:

```
jmondi@w540:~$ adb forward tcp:2345 tcp:2345
```
Easy!

**NOTE**  
*gdb uses init file, such as ~/.gdbinit, with a set of command it executes at startup. Use an empty file the first time and make sure nothing is executed on startup.*

Now we can start gdb, or some frontend like ddd.
Once the debugger has started, we have to load symbols for the library we want to debug, and we have to load it at the correct memory address, so that the debugger knows how to map them to the running target.

The trick here is how to retrieve the address where the library is loaded at.
To get to know it, we need to inspect the memory map of the process that hosts our library, the same we have attacched gdb to on the target side.

On Linux systems, the memory mapping is exposed to userspace by means of the */proc/* virtual filesystem. Knowing the PID of the service we are interested in, we can dump it, and look for the mnemonic name of the library we want to debug

```
jmondi@w540:~$ adb shell cat /proc/131/maps | grep camera.omap
40c54000-40cd5000 r-xp 00000000 103:00 538  /system/lib/hw/camera.omap4.so 
40cd6000-40cda000 r--p 00081000 103:00 538  /system/lib/hw/camera.omap4.so
40cda000-40cdb000 rw-p 00085000 103:00 538  /system/lib/hw/camera.omap4.so
```

What we see here is the process memory map, with the address where the library has been loaded. The memory map shows different permission for each ELF sector, have a look at [this](http://stackoverflow.com/questions/14361248/whats-the-difference-of-section-and-segment-in-elf-file-format) to know more about sections and sectors in ELF format and, have a look at [this other Stackoverflow thread](http://stackoverflow.com/questions/1401359/understanding-linux-proc-id-maps) for the */proc/maps* output format!

What we are interested in, by the way, is the lowest memory address, where the library has been placed: in our example it is **0x40c54000**.

This is not enough by the way, since we need to know exactly where the **.text** section of our object is, or more precisely what is its displacement from the start of the library object.
We can use the *objdump* tool to analyze the .so and find this out:

```
jmondi@w540:~$ arm-eabi-objdump -h out/target/product/PRODUCT/symbols/system/lib/hw/camera.omap4.so  | grep text
00043d0c  00024630  00024630  00024630  2**3
```

The second value in the output (**_0x00024630_**) is the displacement, thus we can sum the two value and have the exact memory location where the executable part of the library has been loaded.

***  40c54000+00024630 = 40C78630 ***

Let's get back to our debugger console, and proceed in loading symbols; the service symbols first, then the library's ones at the correct memory location.

```
(gdb) file out/target/product/dl40pb/system/bin/service
(gdb) add-symbol-file camera.omap4.so 0x40C78630 
add symbol table from file "camera.omap4.so" at .text_addr = 0x40c78630
```

Now connect to the remote gdbserver interface, forwarded on a local port by adb, and install some breakpoint in the code

```
(gdb) target remote localhost:2345
*gdb complains here*
(gdb) b CameraHal.cpp:1765
*gdb complains here*
```

Now, do whatever drives execution to the point where you set the breakpoint, and... it *should work* and you should be able to step in the shared object code, as you would do for Java applications!


# Links

Have also a look at [this](http://linux-mobile-hacker.blogspot.co.uk/2008/02/debug-shared-library-with-gdbserver.html) where most of the same is done on Linux!

