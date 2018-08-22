---
layout:     post
title:      "non-A/B Recovery OTA 升级流程"
subtitle:   "non-A/B OTA work flow,user guide"
date:       2018-8-21 22:00:00
author:     "Bydas"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - recovery ota
    - Android
---
# non-A/B OTA 升级流程

## 一、OTA 升级三大步

**1.1 服务器 -> 客户端**

- FOTA (升级差分包)或者 UL (整包)，通过在服务器配置机器的辨识号码（如 IMEI 或者 ISN）与本地机器的 DM Client (fota server apk) 进行 C/S 交互，将服务器的差分包下载到本地指定路径。

**1.2 本地机器升级**

- DMC 应用程序将升级包的路径参数写入到指定路径保留，重启机器进入 recovery 模式，通过 recovery 启动流程调用相关升级参数选择对应的工作流程。

**1.3 升级完成后的功能测试**

- 当机器升级完成后，需要做各个模块的功能性测试，由于本文主要介绍升级流程，主要针对升级过程的 debug 做介绍。

![image](http://ww1.sinaimg.cn/large/005xWQEggy1fu9b1zwgl6j30f10fkmxm.jpg)

## 二、Android 系统三种启动模式

![image](http://ww1.sinaimg.cn/large/005xWQEggy1fu3k1i5rt9j30l40afq3b.jpg)

1. bootloader : fastboot 模式处于此阶段，作为启动Android系统之前的引导程序，可通过读取 MISC 分区（BCB Bootloader Control Block）获得来至 Main system 和 Recovery 的消息。

2. main system : bootloader 读取的 MISC 分区为空时，以正常模式启动，是用 boot.img 启动的系统，Android 的正常工作模式。更新时，在这种模式中我们的上层操作就是使用 OTA 或则从 SD 卡中升级 update.zip 包。在重启进入 Recovery 模式之前，会向 BCB 中写入命令，以便在重启后告诉 bootloader 进入 Recovery 模式。

3. recovery : 系统进入 Recovery 模式后会装载 Recovery 分区，该分区包含 recovery.img（同 boot.img 相同，包含了标准的内核和根文件系统）。进入该模式后主要是运行 Recovery 服务（/sbin/recovery）来做相应的操作（重启、升级 update.zip、擦除 cache 分区等）。

### 2.1 三种模式的通讯方式
#### 2.1.1 通过CACHE分区中的文件

Recovery 通过 /cache/recovery/ 目录下的文件与 main system 通信,常使用的有如下
![image](http://ww1.sinaimg.cn/large/005xWQEggy1fu3p44s2qrj30od0c83zm.jpg)

从 main system -> recovery 传入命令介绍，可参考 [recovery.cpp](http://androidxref.com/8.0.0_r4/xref/bootable/recovery/recovery.cpp#148)：
- --send_intent=anystring   // 在 Recovery 结束时在 finish_recovery 函数中将定义的 intent 字符串作为参数传进来，并写入到 /cache/recovery/intent中
- --update_package=root:path   // Main system 将这条命令写入时,代表系统需要升级，在进入 Recovery 模式后，将该文件中的命令读取并写入 BCB 中，然后进行相应的更新 update.zip 包的操作
- --wipe_data // 擦除用户数据。擦除 data 分区时必须要擦除 cache 分区
- --wipe_cache   // wipe cache(but not user data),then reboot
- --set_encrypted_filesystem=on/off   -enables  // diasables encrypted fs


#### 2.1.2 通过BCB（Bootloader Control Block）

BCB 是 bootloader 与 Recovery 的通信接口，也是 Bootloader 与 Main system 之间的通信接口。存储在 flash 中的 MISC 分区，大小为 2-KiB，其本身就是一个结构体，实现参考 [bootloader_message.h](http://androidxref.com/8.0.0_r4/xref/bootable/recovery/bootloader_message/include/bootloader_message/bootloader_message.h#63)，具体成员以及各成员含义如下（Android O）：
```
struct bootloader_message {
    char command[32];
    // The command field is updated by linux when it wants to
    // reboot into recovery or to update radio or bootloader firmware.
    // It is also updated by the bootloader when firmware update
    // is complete (to boot into recovery for any final cleanup)
    
    char status[32];
    // The status field was used by the bootloader after the completion
    // of an "update-radio" or "update-hboot" command, which has been
    // deprecated since Froyo.
    
    char recovery[768];
    // The recovery field is only written by linux and used
    // for the system to send a message to recovery or the other way around.
    // The 'recovery' field used to be 1024 bytes.  It has only ever
    // been used to store the recovery command line, so 768 bytes
    // should be plenty.  We carve off the last 256 bytes to store the
    // stage string (for multistage packages) and possible future expansion.
    
    char stage[32];
    // The stage field is written by packages which restart themselves
    // multiple times, so that the UI can reflect which invocation of the
    // package it is.  If the value is of the format "#/#" (eg, "1/3"),
    // the UI will add a simple indicator of that status.

    char reserved[1184];
    // The 'reserved' field used to be 224 bytes when it was initially
    // carved off from the 1024-byte recovery field. Bump it up to
    // 1184-byte so that the entire bootloader_message struct rounds up
    // to 2048-byte.
};
```
在系统重启之前，我们可以看到，Main System 定会向 BCB 中的 command 域写入 boot-recovery，用来告知 Bootloader 重启后进入 recovery 模式，向 /cache/recovery/command 中写入 Recovery 将要进行的操作命令。

### 2.2 三种模式的相互联系

![image](http://ww1.sinaimg.cn/large/005xWQEggy1fu9b7ffzh9j30iq0cigm0.jpg)

- 流程 1：bootloader 读取 BCB 中的 command 成员变量 ，若为 boot-recovery 启动 recovery,若 command 为空则启动 main system。

- 流程 2-1：main system 可往 BCB 中的 command 字段写入 boot-recovery 或者往 recovery 字段 写入 –update_package= PATH, 注意 recovery 字段存放的是 recovry 模块的启动参数，一般包括升级包路径。其存储结构如下：第一行存放字符串“recovery”,第二行存放路径信息“–update_package=update.zip PATH”等。 因此，参数之间是以“\n”分割的。

- 流程 2-2：先读取 BCB 然后读取 /cache/recovery/command，然后将后者重新写回 BCB，当 升级失败可重新读取 BCB 中的 command。在 finish_recovery 函数中，Recovery 又会清空 BCB 的 command 域和 recovery 域，这样确保重启后不再进入 Recovery 模式。

- 流程 3-1：main system 往 /cache/recovery/command 中写入信息，也可从 /cache/recovery/intent 中读取 recovery 传给 main system 的 intent 字符串。

- 流程 3-2：recovery 从 cache/recovery/command 中获取操作命令，并存放 recovery 过程的相关信息和 log。

- 按键进入：从 reboot 到 Recovery 服务 从 Bootloader 开始如果检测到组合键按下，就以 Recovery 模式开始启动启动镜像是 recovery.img。这个镜像同 boot.img 类似，也包含了标准的内核和根文件系统。其后就与正常的启动系统类似，也是启动内核，然后启动文件系统。在进入文件系统后会执行 init，init 的配置文件是 recovery 下的 [init.rc](http://androidxref.com/8.0.0_r4/xref/bootable/recovery/etc/init.rc)，启动 recovery（/sbin/recovery）服务。

- 服务重启异常进入：服务在4分钟内重启次数超过4次，则重启手机进入recovery模式，参考 [service.cpp](http://androidxref.com/8.0.0_r4/xref/system/core/init/service.cpp#Reap)

  ```
      // If we crash > 4 times in 4 minutes, reboot into recovery.
      boot_clock::time_point now = boot_clock::now();
      if ((flags_ & SVC_CRITICAL) && !(flags_ & SVC_RESTART)) {
          if (now < time_crashed_ + 4min) {
              if (++crash_count_ > 4) {
                  LOG(ERROR) << "critical process '" << name_ << "' exited 4 times in 4 minutes";
                  panic();
              }
          } else {
              time_crashed_ = now;
              crash_count_ = 1;
          }
      }
  ```
## 三、OTA 升级服务流程
### 3.1 应用层升级流程

![image](http://ww1.sinaimg.cn/large/005xWQEggy1fu9bd388slj30gx0j6js4.jpg)
- 应用层主要通过 DM client apk 完成与服务器的交互工作，将升级包下载到本地，并触发升级更新操作提示。
- 当本地下载升级包完成后，会调用 RecoverySystem.java 下的 [installPackage](http://androidxref.com/8.0.0_r4/xref/frameworks/base/core/java/android/os/RecoverySystem.java#523) 方法进行安装过程操作：

#### 3.1.1 installPackage 实现流程

1. 如果将升级包下载到 data 分区，判断升级包是否有做过 uncrypt'd 处理，如果升级过程失败，可做 uncrypt 的 avc 检查。
2. 将传入的文件名替换为 filename = "@/cache/recovery/block.map"，升级开始需要匹配该文件才能正常升级。
3. 将 command ："--update_package=" 和 "--locale="，写入到 BCB。
4. 最后，以 boot-recovery 重启进入 recovery 进行升级操作。 

#### 3.1.2 使用 adb 指令调试安装升级包

- 将升级包 push 进 data 分区，进入 recovery 模式：
1. To initiate recovery, run the following adb commands:

   ```
   adb root 
   adb push update.zip /data/update.zip
   adb shell mkdir /cache/recovery 
   ```


2. Create a file called command as follows:

   ```
   echo "--update_package=/data/update.zip" > command
   ```


3. Run the following commands:

   ```
   adb shell sync
   adb reboot recovery
   ```
   The device reboots, updates all images, and reboots again. The logs for the last 
   recovery are located under /cache/recovery.

### 3.2 recovery 阶段升级服务流程

recovery 服务流程可从 recovery.cpp 中的 [main()](http://androidxref.com/8.0.0_r4/xref/bootable/recovery/recovery.cpp#1352) 入口进行分析：

#### 3.2.1 Recovery 的两大类服务：

##### 3.2.1.1 FACTORY RESET，恢复出厂设置

1. user selects "factory reset"

2. main system writes "--wipe_data" to /cache/recovery/command

3. main system reboots into recovery

4. get_args() writes BCB with "boot-recovery" and "--wipe_data"

    -- after this, rebooting will restart the erase --

5. erase_volume() reformats /data

6. erase_volume() reformats /cache

7. finish_recovery() erases BCB

    -- after this, rebooting will restart the main system --

8. main() calls reboot() to boot main system

##### 3.2.1.2 OTA INSTALL，即 update.zip 包升级

![image](http://ww1.sinaimg.cn/large/005xWQEggy1fuacry2rqkj30im0pk764.jpg)

1. main system downloads OTA package to /cache/some-filename.zip
2. main system writes "--update_package=/cache/some-filename.zip"
3. main system reboots into recovery
4. get_args() writes BCB with "boot-recovery" and "--update_package=..."

   -- after this, rebooting will attempt to reinstall the update 
5. install_package() attempts to install the update

   -- NOTE: the package install must itself be restartable from any point
6. finish_recovery() erases BCB

   -- after this, rebooting will (try to) restart the main system 
7. if install failed
   i. prompt_and_wait() shows an error icon and waits for the user
   ii. the user reboots (pulling the battery, etc) into the main system

#### 3.2.2 install_package() 流程

具体实现代码可参考 [install_package](http://androidxref.com/8.0.0_r4/xref/bootable/recovery/install.cpp#630) 方法实现，其流程图如下：

![image](http://ww1.sinaimg.cn/large/005xWQEggy1fubm898cj7j30ix0pkmyl.jpg)

1. pipe()：创建管道，用于下面的子进程和父进程之间的通信。
2. [update_binary_command](http://androidxref.com/8.0.0_r4/xref/bootable/recovery/install.cpp#269) 方法用文件读写操作实现将 "META-INF/com/google/android/update-binary" 备份到 "/tmp/update_binary", 将 binary 作为参数传入 execv(chr_args[0], const_cast<char**>(chr_args)) ，创建 binary 子进程。
3. fork()：创建子进程。其中的子进程主要负责执行 binary（execv(binary,args)，即执行我们的安装命令脚本），父进程负责接受子进程发送的命令去更新ui显示（显示当前的进度）。子父进程间通信依靠管道。
4. 在创建子进程后，父进程有两个作用。
    1. 一是通过管道接受子进程发送的命令来更新UI显示。
    2. 二是等待子进程退出并返回 INSTALL  SUCCESS。其中子进程在解析执行安装脚本的同时所发送的命令主要有以下几种：
-     progress  <frac> <secs>：根据第二个参数secs（秒）来设置进度条
-     set_progress  <frac>：直接设置进度条，frac取值在0.0到0.1之间
-     ui_print <string>：在屏幕上显示字符串，即打印更新过程

##### 3.2.2.1 update-binary 进程具体实现

execv(binary,args)，其执行的 update-binary 具体实现源码可参考 [updater.cpp](http://androidxref.com/8.0.0_r4/xref/bootable/recovery/updater/updater.cpp#59) 。

1. 函数参数以及版本的检查：当前 updater binary API 所支持的版本号有 1，2，3 三个。

2. 获取管道并打开：在执行此程序的过程中向该管道写入命令，用于通知其父进程根据命令去更新 UI 显示。

3. 读取 updater-script 脚本：从 update.zip 包中将 updater-script 脚本读到一块动态内存中，供后面执行。

   ```
     ZipString script_name(SCRIPT_NAME);
     ZipEntry script_entry;
     int find_err = FindEntry(za, script_name, &script_entry);
   ```


4. Configure edify's functions：注册脚本中的语句处理函数，即识别脚本中命令的函数。主要有以下几类:

   ```
     RegisterBuiltins();   
     //注册程序中控制流程的语句，如 ifelse、assert、abort、stdout 等
     
     RegisterInstallFunctions(); 
     //实际安装过程中安装所需的功能函数，比如 mount、format、set_progress、set_perm 等
     
     RegisterBlockImageFunctions();
     //与设备相关的额外添加項，在源码中并没有任何实现
     
     RegisterDeviceExtensions();
     //结束注册
   ```


5. Parse the script：调用 yy* 库函数解析脚本，并将解析后的内容存放到一个 Expr 类型的 python 类中。主要函数是 yy_scan_string() 和 yyparse()。
6. Evaluate the parsed script：核心函数是 Evaluate（），它会调用其他的 callback 函数，而这些 callback 函数又会去调用 Evaluate 去解析不同的脚本片段，从而实现一个简单的脚本解释器。
7. 错误信息提示：最后就是根据 Evaluate（）执行后的返回值，给出一些打印信息。

##### 3.2.2.2  updater-script 脚本与 Edify 语法

update-script 脚本格式是 edify，具体可参考 Android 官网对 [Edify](https://source.android.com/devices/tech/ota/nonab/inside_packages) 语法的介绍。

###### 3.2.2.2.1 update-script 脚本常用语法：

1. abort([msg])：使用可选的 msg 立即中止脚本执行。如果用户开启了文本显示功能，msg 将出现在恢复日志和屏幕上。
2. assert(condition)：如果 condition 参数的计算结果为 False，则停止脚本执行，否则继续执行脚本。
3. apply_patch_space(bytes)：用于差分升级脚本中，如果至少有 bytes 个字节的暂存空间可用于打二进制补丁程序，则返回 True。
4. show_progress(frac,sec)：frac 表示进度完成的数值，sec 表示整个过程的总秒数。主要用与显示 UI 上的进度条。
5. package_extract_file(srcfile_path,desfile_paht)：srcfile_path，要提取的文件，desfile_path，提取文件的目标位置。示例：package_extract_file(“boot.img”,”/tmp/boot.img”)将升级包中的 boot.img 文件拷贝到内存文件系统的 /tmp 下。
6. getprop(key)：通过指定 key 的值来获取对应的属性信息。示例：getprop(“ro.product.device”) 获取 ro.product.device 的属性值。

###### 3.2.2.2.2 qualcomm 整包 updater-script 脚本执行流程分析：

  ```
  (!less_than_int(1526288656, getprop("ro.build.date.utc"))) || abort("E3003: Can't install this package (Mon May 14 17:04:16 CST 2018) over newer build (" + getprop("ro.build.date") + ").");
  getprop("ro.product.device") == "%s" || '
             'abort("E%d: This package is for \\"%s\\" devices; '
             'this is a \\"" + getprop("ro.product.device") + "\\".");')
  ui_print("Target: "); // 可获取 ro.build.fingerprint 属性来判断
  show_progress(0.750000, 0);
  ui_print("Patching system image unconditionally...");
  block_image_update("/dev/block/bootdevice/by-name/system", package_extract_file("system.transfer.list"), "system.new.dat", "system.patch.dat") ||
    abort("E1001: Failed to update system image.");
  show_progress(0.050000, 5);
  package_extract_file("boot.img", "/dev/block/bootdevice/by-name/boot");
  show_progress(0.200000, 10);

  # ---- radio update tasks ----

  ui_print("Patching firmware images...");
  ifelse(msm.boot_update("main"), (
  package_extract_file("firmware-update/cmnlib64.mbn", "/dev/block/bootdevice/by-name/cmnlib64");
  package_extract_file("firmware-update/splash.img", "/dev/block/bootdevice/by-name/splash");
  package_extract_file("firmware-update/cmnlib.mbn", "/dev/block/bootdevice/by-name/cmnlib");
  package_extract_file("firmware-update/rpm.mbn", "/dev/block/bootdevice/by-name/rpm");
  package_extract_file("firmware-update/tz.mbn", "/dev/block/bootdevice/by-name/tz");
  package_extract_file("firmware-update/mdtp.img", "/dev/block/bootdevice/by-name/mdtp");
  package_extract_file("firmware-update/lksecapp.mbn", "/dev/block/bootdevice/by-name/lksecapp");
  package_extract_file("firmware-update/sbl1.mbn", "/dev/block/bootdevice/by-name/sbl1");
  package_extract_file("firmware-update/devcfg.mbn", "/dev/block/bootdevice/by-name/devcfg");
  package_extract_file("firmware-update/keymaster.mbn", "/dev/block/bootdevice/by-name/keymaster");
  ), "");

  msm.boot_update("finalize");
  package_extract_file("firmware-update/adspso.bin", "/dev/block/bootdevice/by-name/dsp");
  set_progress(1.000000);
  ```
1. 比较时间戳：如果升级包较旧则终止脚本的执行，具体实现参考 edify_generator.py 下调用的 [AssertOlderBuild](http://androidxref.com/8.0.0_r4/xref/build/tools/releasetools/edify_generator.py#137) 方法。

2. 匹配设备信息：如果和当前的设备信息不一致，则停止脚本的执行，流程实现参考 ota_from_target_files.py 下调用的  [AppendAssertions](http://androidxref.com/8.0.0_r4/xref/build/make/tools/releasetools/ota_from_target_files.py#AppendAssertions) 方法：

   ```
   def WriteFullOTAPackage(input_zip, output_zip):
     # TODO: how to determine this?  We don't know what version it will
     # be installed on top of. For now, we expect the API just won't
     # change very often. Similarly for fstab, it might have changed
     # in the target build.
     script = edify_generator.EdifyGenerator(3, OPTIONS.info_dict)

     recovery_mount_options = OPTIONS.info_dict.get("recovery_mount_options")
     oem_props = OPTIONS.info_dict.get("oem_fingerprint_properties")
     oem_dicts = None
     if oem_props:
       oem_dicts = _LoadOemDicts(script, recovery_mount_options)

     target_fp = CalculateFingerprint(oem_props, oem_dicts and oem_dicts[0],
                                      OPTIONS.info_dict)
     metadata = {
         "post-build": target_fp,
         "pre-device": GetOemProperty("ro.product.device", oem_props,
                                      oem_dicts and oem_dicts[0],
                                      OPTIONS.info_dict),
         "post-timestamp": GetBuildProp("ro.build.date.utc", OPTIONS.info_dict),
     }

     device_specific = common.DeviceSpecificParams(
         input_zip=input_zip,
         input_version=OPTIONS.info_dict["recovery_api_version"],
         output_zip=output_zip,
         script=script,
         input_tmp=OPTIONS.input_tmp,
         metadata=metadata,
         info_dict=OPTIONS.info_dict)

     assert HasRecoveryPatch(input_zip)

     metadata["ota-type"] = "BLOCK"

     #升级前对升级包时间和设备辨识进行判断
     ts = GetBuildProp("ro.build.date.utc", OPTIONS.info_dict)
     ts_text = GetBuildProp("ro.build.date", OPTIONS.info_dict)
     script.AssertOlderBuild(ts, ts_text)

     AppendAssertions(script, OPTIONS.info_dict, oem_dicts)
     device_specific.FullOTA_Assertions()
   ```


3. 显示进度条：如果以上两步匹配则开始显示升级进度条。
4. 将升级包中的文件写入到对应的分区，先升级 system，再升级 boot。
5. 升级 modem 分区，将 modem 分区放在 /device/qcom/<target>/RADIO 下，将需要升级的分区写入 radio 下的 filesmap 进行匹配。

#### 3.2.3 finish_recovery() 流程
[finish_recovery()](http://androidxref.com/8.0.0_r4/xref/bootable/recovery/recovery.cpp#finish_recovery) 函数功能介绍：
- clear the recovery command and prepare to boot a (hopefully working) system.
- copy our log file to cache as well (for the system to read). 
- This function is idempotent: call it as many times as you like.

![image](http://ww1.sinaimg.cn/large/005xWQEggy1fuihk3iwfgj308x0geglw.jpg)

## 四、non-A/B OTA 升级流程总结

1. OTA 主要工作分为 **在线下载升级包、本地升级、升级完成后的功能验证 **三部分。
2. 介绍 Android 系统启动的三种模式  **fastboot、recovery、normal boot** 。
3. 深入三种模式是如何进行通信来传递 message command，着重介绍 **BCB** 和 **/cache/recovery 下的文件信息** 两种方式。
4. 先介绍应用层是如何将在线下载的升级包参数传递到底层，再详细分析底层 recovery 阶段是如何完成升级过程，最后完成升级是如何处理传递进来的的 message command 的。





















