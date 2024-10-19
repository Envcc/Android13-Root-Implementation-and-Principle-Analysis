Android 13 is a relatively new version of Android (this patch was modified half a year ago). It brings many new features and improvements, but also adds some security restrictions. It is more troublesome to get root permissions for the system than in previous versions. If you want to have full control on Android 13, you need to modify some system files and kernel code to bypass SELinux, Verity, and other protection mechanisms. This article will introduce a method of implementing root functionality on Android 13, as well as the files and code that need to be modified.
If your device does not support GMS , you can do whatever you want. But be careful, if your product involves banking education, their APP products may not work!

This method is for debugging purposes only.

Prerequisites
To make a machine, the following conditions are required:
• Android 13 source code and devices
• Looks more handsome
• Smarter brain
Step 1: Modify system files
Some system files need to be modified to allow the adbd process to run under the root user and turn off Verity checks. We need to modify the following files:


1.frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
This file is responsible for creating application processes and setting their permissions and capabilities. You need to comment out DropCapabilitiesBoundingSetthe code in the function to prevent it from deleting any capabilities of the adbd process. This file should be commented out in previous versions.
@@ -658,7 +658,7 @@ static void EnableKeepCapabilities(fail_fn_t fail_fn) {
 }
 
 static void DropCapabilitiesBoundingSet(fail_fn_t fail_fn) {
-  for (int i = 0; prctl(PR_CAPBSET_READ, i, 0, 0, 0) >= 0; i++) {;
+  /*for (int i = 0; prctl(PR_CAPBSET_READ, i, 0, 0, 0) >= 0; i++) {;
     if (prctl(PR_CAPBSET_DROP, i, 0, 0, 0) == -1) {
       if (errno == EINVAL) {
         ALOGE("prctl(PR_CAPBSET_DROP) failed with EINVAL. Please verify "
@@ -667,7 +667,7 @@ static void DropCapabilitiesBoundingSet(fail_fn_t fail_fn) {
         fail_fn(CREATE_ERROR("prctl(PR_CAPBSET_DROP, %d) failed: %s", i, strerror(errno)));
       }
     }
-  }
+  }*/
 }
2.packages/modules/adb/Android.bp
This file defines the compilation options and dependencies of the adbd module. It needs to be added -DALLOW_ADBD_ROOT=1to cflags to enable the root mode of the adbd process, and added remountto required to allow the adbd process to remount the system partition. This file seems to be from Android 12 and above, adb has changed to this directory
@@ -50,6 +50,7 @@ cc_defaults {
         "-Wvla",
         "-DADB_HOST=1",         // overridden by adbd_defaults
         "-DANDROID_BASE_UNIQUE_FD_DISABLE_IMPLICIT_CONVERSION=1",
+   "-DALLOW_ADBD_ROOT=1",
     ],
     cpp_std: "experimental",
 
@@ -111,8 +112,15 @@ cc_defaults {
 cc_defaults {
     name: "adbd_defaults",
     defaults: ["adb_defaults"],
+    cflags: [
+    "-UADB_HOST",
+    "-DADB_HOST=0",
+    "-UALLOW_ADBD_ROOT",
+    "-DALLOW_ADBD_ROOT=1",
+    "-DALLOW_ADBD_DISABLE_VERITY",
+    "-DALLOW_ADBD_NO_AUTH",
+],
 
-    cflags: ["-UADB_HOST", "-DADB_HOST=0"],
 }
 
 cc_defaults {
@@ -605,7 +613,7 @@ cc_library {
         "libcrypto",
         "liblog",
     ],
-
+    required: [ "remount",],
     target: {
         android: {
             srcs: [
3.packages/modules/adb/daemon/main.cpp
This file is the main entry point for the adbd process. We need to modify should_drop_privilegesthe function so that it always returns false to prevent it from lowering the privileges of the adbd process. This file is usually commented out in previous versions.
@@ -64,6 +64,7 @@
 static const char* root_seclabel = nullptr;
 
 static bool should_drop_privileges() {
+    return false;
     // The properties that affect `adb root` and `adb unroot` are ro.secure and
     // ro.debuggable. In this context the names don't make the expected behavior
     // particularly obvious.
4.system/core/fs_mgr/Android.bp
This file defines the compilation options and dependencies of the fs_mgr module. The fs_mgr module is responsible for managing the file system on the device. We need to modify -DALLOW_ADBD_DISABLE_VERITY=0it -DALLOW_ADBD_DISABLE_VERITY=1to allow the adbd process to turn off Verity checks. This file seems to be from Android 12 and above, and adb has changed to this directory
@@ -237,7 +237,8 @@ cc_binary {
         "fs_mgr_remount.cpp",
     ],
     cppflags: [
-        "-DALLOW_ADBD_DISABLE_VERITY=0",
+   "-UALLOW_ADBD_DISABLE_VERITY",
+        "-DALLOW_ADBD_DISABLE_VERITY=1",
     ],
     product_variables: {
         debuggable: {
5.system/core/init/Android.bp
This file defines the compilation options and dependencies of the init module. The init module is the first process that runs when the device starts, and is responsible for initializing system services and properties. We need to modify the following options: ( It doesn't really matter, I just changed them )
• -DALLOW_FIRST_STAGE_CONSOLE=1: Allow the init process to open console output in the first stage
• -DALLOW_LOCAL_PROP_OVERRIDE=1: Allow the init process to overwrite local properties
• -DALLOW_PERMISSIVE_SELINUX=1: Allow the init process to set SELinux to permissive mode
• -DREBOOT_BOOTLOADER_ON_PANIC=1: Allow the init process to reboot into bootloader mode when a kernel panic occurs
• -DWORLD_WRITABLE_KMSG=1: Allow the init process to set the kmsg file to be writable
• -DDUMP_ON_UMOUNT_FAILURE=1: Allow the init process to generate a memory dump when unmounting a partition fails
• -DSHUTDOWN_ZERO_TIMEOUT=1: Allow the init process to execute immediately when receiving a shutdown command
@@ -113,13 +113,13 @@ libinit_cc_defaults {
         "-Wno-unused-parameter",
         "-Werror",
         "-Wthread-safety",
-        "-DALLOW_FIRST_STAGE_CONSOLE=0",
-        "-DALLOW_LOCAL_PROP_OVERRIDE=0",
-        "-DALLOW_PERMISSIVE_SELINUX=0",
-        "-DREBOOT_BOOTLOADER_ON_PANIC=0",
-        "-DWORLD_WRITABLE_KMSG=0",
-        "-DDUMP_ON_UMOUNT_FAILURE=0",
-        "-DSHUTDOWN_ZERO_TIMEOUT=0",
+        "-DALLOW_FIRST_STAGE_CONSOLE=1",
+        "-DALLOW_LOCAL_PROP_OVERRIDE=1",
+        "-DALLOW_PERMISSIVE_SELINUX=1",
+        "-DREBOOT_BOOTLOADER_ON_PANIC=1",
+        "-DWORLD_WRITABLE_KMSG=1",
+        "-DDUMP_ON_UMOUNT_FAILURE=1",
+        "-DSHUTDOWN_ZERO_TIMEOUT=1",
         "-DINIT_FULL_SOURCES",
         "-DINSTALL_DEBUG_POLICY_TO_SYSTEM_EXT=0",
     ],
Step 2: Modify the kernel code
Next, we need to modify some kernel code to allow the adbd process to modify the system's capability set, as well as to turn off SELinux enforcement. We need to modify the following files:
1.kernel-5.10/security/commoncap.c
This file implements some common capability manipulation functions. We need to comment out cap_prctl_dropthe code in the function to prevent it from checking whether the adbd process has the CAP_SETPCAP capability and whether a valid capability parameter is passed.
@@ -1163,11 +1163,11 @@ static int cap_prctl_drop(unsigned long cap)
 {
    struct cred *new;
 
-   if (!ns_capable(current_user_ns(), CAP_SETPCAP))
+/* if (!ns_capable(current_user_ns(), CAP_SETPCAP))
        return -EPERM;
    if (!cap_valid(cap))
        return -EINVAL;
-
+*/
    new = prepare_creds();
    if (!new)
        return -ENOMEM;
2.system/core/init/selinux.cpp
This file implements some SELinux related functions. We need to modify IsEnforcingthe function so that it always returns false to prevent it from checking if the system properties or kernel parameters have SELinux enforcement set.
@@ -102,6 +102,7 @@ namespace {
 
 enum EnforcingStatus { SELINUX_PERMISSIVE, SELINUX_ENFORCING };
 
+/*
 EnforcingStatus StatusFromProperty() {
     EnforcingStatus status = SELINUX_ENFORCING;
 
@@ -120,13 +121,15 @@ EnforcingStatus StatusFromProperty() {
     }
 
     return status;
-}
+}*/
 
 bool IsEnforcing() {
-    if (ALLOW_PERMISSIVE_SELINUX) {
+    //add root
+    return false;
+    /*if (ALLOW_PERMISSIVE_SELINUX) {
         return StatusFromProperty() == SELINUX_ENFORCING;
     }
-    return true;
+    return true;*/
 }
Summarize
Through the above modifications, we can achieve root function on Android 13. However, these modifications may reduce the security of the device, please do so only after clearly understanding the risks.
I hope this blog post is helpful to you. If you have any questions or suggestions, please leave a message to discuss. Thank you!
