---
layout: post
title: "Building the Pixel 8 kernel with MTE, KASAN and KCOV"
date: 2026-04-28
categories: mobile-security
---

For my fuzzing environment I use a stack of Pixel8 devices which run under KASAN + KCOV. Recently I had to update my devices to the latest kernel version which broke my build. Google's documentation is great to have, but not very specific when it comes to the details and you mostly end up with a bootloop. This post covers compiling the Pixel 8 kernel and booting it on the current latest Pixel 8 version.

<!-- more -->

The Pixel 8 supports `MTE`. That means we can enable `KASAN_HW_TAGS`, drastically reducing the overhead brought by `KASAN_GENERIC` or `KASAN_SW_TAGS`. However mind that `KASAN_VMALLOC` will be disabled, and the last time I checked was only supported on `KASAN_GENERIC`. For my fuzzing work, I need `MTE` and `KCOV` to be enabled, and `KASLR` disabled simply because it makes comparing crash reports easier and removes some overhead. I also disable SELinux for my own convenience, which may not make sense depending on which context you're fuzzing from. 

`KASAN_HW_TAGS` itself is upstream work by [Andrey Konovalov (xairy)](https://xairy.io/) and Vincenzo Frascino — see the [introducing commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6a63a63ff1ac2959706dba218d5e17f9ec721c0c) on git.kernel.org. If you're interested in debugging the Pixel kernel via GDB with GEF enabled, you can refer to Andrey's [article](https://xairy.io/articles/pixel-kgdb).


## Downloading the relevant factory image

Version mismatch between the factory image and your custom kernel will boot-loop with no useful diagnostics, so flash a good baseline first.

Go to [https://developers.google.com/android/images](https://developers.google.com/android/images), grab the Pixel 8 (shiba) factory image for `Android 16 (CP1A.260405.005)`, extract, run the bundled `flash-all.sh`. Enable developer options and OEM unlocking through the phone afterwards.

```
https://dl.google.com/dl/android/aosp/shiba-cp1a.260405.005-factory-XXXXXXXX.zip
```


## Downloading and compiling the kernel

```sh
repo init -u https://android.googlesource.com/kernel/manifest -b android-gs-shusky-6.1-android16
repo sync -c --no-tags
```

(`uname -r` will report `6.1.124-android14-11...` once you boot. That's the GKI ABI version, not the platform branch.)


Create `private/google-modules/soc/gs/build.config.slider.khwasan` with the configs we want; this fragment runs after `defconfig` and overrides via `scripts/config`.

```sh
append_cmd POST_DEFCONFIG_CMDS update_slider_khwasan_config

function update_slider_khwasan_config() {
  ${KERNEL_DIR}/scripts/config --file ${OUT_DIR}/.config \
    -e CONFIG_KASAN \
    -e CONFIG_KASAN_HW_TAGS \
    -d CONFIG_KASAN_SW_TAGS \
    -e CONFIG_KASAN_INLINE \
    -e CONFIG_KASAN_PANIC_ON_WARN \
    -e CONFIG_KCOV \
    -e CONFIG_PANIC_ON_WARN_DEFAULT_ENABLE \
    -d CONFIG_RANDOMIZE_BASE \
    --set-val CONFIG_FRAME_WARN 0 \
    -d CONFIG_SHADOW_CALL_STACK \
    -e CONFIG_SECURITY_SELINUX_BOOTPARAM
  (cd ${OUT_DIR} && \
   make O=${OUT_DIR} "${TOOL_ARGS[@]}" ${MAKE_ARGS} olddefconfig)
}
```

Export it in `private/google-modules/soc/gs/BUILD.bazel`:

```python
exports_files(["build.config.slider.khwasan"])
```

Append `nokaslr` to `chosen.bootargs` in `private/devices/google/zuma/dts/zuma.dtsi`.

Edit `build_shusky.sh` and change the last line to:

```sh
exec tools/bazel run \
    ${parameters} \
    --config=stamp \
    --lto=none \
    --config=shusky \
    --gki_build_config_fragment=//private/google-modules/soc/gs:build.config.slider.khwasan \
    --sandbox_debug \
    --kasan \
    --//build/kernel/kleaf:use_prebuilt_gki=false \
    //private/devices/google/shusky:zuma_shusky_dist "$@"
```

We set `--//build/kernel/kleaf:use_prebuilt_gki=false` here because the `--use_prebuilt_gki=false` alias set on the command line fails to override the same value set in `device.bazelrc`. In that case the GKI keeps coming from the prebuilt, throwing an error.

Compile:

```sh
./build_shusky.sh
```

## Patching SELinux to stay permissive

`androidboot.selinux=permissive` and `enforcing=0` on the cmdline are not enough on a `user` build. Android `init` writes `1` to `/sys/fs/selinux/enforce` after policy load, effectively setting it back to `enforcing`.

If you're lazy like me, you can just patch `aosp/security/selinux/selinuxfs.c:sel_write_enforce`:

```c
new_value = !!new_value;

if (new_value)
    new_value = 0;

old_value = enforcing_enabled(state);
```

Note: A better way would be to add this functionality to the `UAF test driver` I'll talk about below, so we can set/unset enforcing whenever we need. That being said, I'm pretty sure there are more legitimate ways to achieve this.

## UAF test driver

To test if `KASAN` works and actually produces a crash report, I've added a driver which we can use to trigger a UAF.

Drop `private/google-modules/misc/uaftest/{uaftest.c,Makefile,Kconfig,BUILD.bazel}` and add `"//private/google-modules/misc/uaftest"` to `private/devices/google/zuma/BUILD.bazel:kernel_ext_modules.srcs`.

The driver registers both `/dev/uaftest` and `/proc/uaftest`.

`/dev/uaftest` is on mode 0666 in source, but Android ueventd downgrades it to 0600. For `/proc/uaftest` procfs respects the mode bits and ueventd doesn't mess with them. While we're at it let's add a "give root" function so that we don't have to deal with magisk.

`private/google-modules/misc/uaftest/uaftest.c`:

```c
// SPDX-License-Identifier: GPL-2.0-only

#include <linux/cred.h>
#include <linux/fs.h>
#include <linux/init.h>
#include <linux/ioctl.h>
#include <linux/kernel.h>
#include <linux/miscdevice.h>
#include <linux/module.h>
#include <linux/proc_fs.h>
#include <linux/slab.h>
#include <linux/uaccess.h>

#define UAFTEST_MAGIC 'U'
#define UAFTEST_IOC_UAF_READ    _IO(UAFTEST_MAGIC, 1)
#define UAFTEST_IOC_UAF_WRITE   _IO(UAFTEST_MAGIC, 2)
#define UAFTEST_IOC_DOUBLE_FREE _IO(UAFTEST_MAGIC, 3)
#define UAFTEST_IOC_OOB_READ    _IO(UAFTEST_MAGIC, 4)
#define UAFTEST_IOC_GIVE_ROOT   _IO(UAFTEST_MAGIC, 100)

#define UAFTEST_BUF_SZ 128

static noinline u8 uaftest_uaf_read(void)
{
	u8 *p = kmalloc(UAFTEST_BUF_SZ, GFP_KERNEL);
	u8 v;

	if (!p)
		return 0;
	memset(p, 0xa5, UAFTEST_BUF_SZ);
	kfree(p);
	/* deliberate UAF read; KASAN/MTE should fire here */
	v = READ_ONCE(p[16]);
	return v;
}

static noinline void uaftest_uaf_write(void)
{
	u8 *p = kmalloc(UAFTEST_BUF_SZ, GFP_KERNEL);

	if (!p)
		return;
	kfree(p);
	WRITE_ONCE(p[32], 0x41);
}

static noinline void uaftest_double_free(void)
{
	u8 *p = kmalloc(UAFTEST_BUF_SZ, GFP_KERNEL);

	if (!p)
		return;
	kfree(p);
	kfree(p);
}

static noinline u8 uaftest_oob_read(void)
{
	u8 *p = kmalloc(UAFTEST_BUF_SZ, GFP_KERNEL);
	u8 v;

	if (!p)
		return 0;
	v = READ_ONCE(p[UAFTEST_BUF_SZ]);   /* one past end */
	kfree(p);
	return v;
}

static int uaftest_give_root(void)
{
	struct cred *new = prepare_creds();
	if (!new)
		return -ENOMEM;
	new->uid.val  = new->gid.val  = 0;
	new->euid.val = new->egid.val = 0;
	new->suid.val = new->sgid.val = 0;
	new->fsuid.val = new->fsgid.val = 0;
	return commit_creds(new);
}

static long uaftest_ioctl(struct file *f, unsigned int cmd, unsigned long arg)
{
	switch (cmd) {
	case UAFTEST_IOC_UAF_READ:
		pr_warn("uaftest: triggering UAF read\n");
		(void)uaftest_uaf_read();
		return 0;
	case UAFTEST_IOC_UAF_WRITE:
		pr_warn("uaftest: triggering UAF write\n");
		uaftest_uaf_write();
		return 0;
	case UAFTEST_IOC_DOUBLE_FREE:
		pr_warn("uaftest: triggering double-free\n");
		uaftest_double_free();
		return 0;
	case UAFTEST_IOC_OOB_READ:
		pr_warn("uaftest: triggering OOB read\n");
		(void)uaftest_oob_read();
		return 0;
	case UAFTEST_IOC_GIVE_ROOT:
		pr_warn("uaftest: GIVE_ROOT requested by uid=%u pid=%d\n",
			from_kuid(&init_user_ns, current_uid()),
			task_pid_nr(current));
		return uaftest_give_root();
	default:
		return -ENOTTY;
	}
}

static const struct file_operations uaftest_fops = {
	.owner          = THIS_MODULE,
	.unlocked_ioctl = uaftest_ioctl,
	.compat_ioctl   = uaftest_ioctl,
	.llseek         = noop_llseek,
};

static struct miscdevice uaftest_dev = {
	.minor = MISC_DYNAMIC_MINOR,
	.name  = "uaftest",
	.fops  = &uaftest_fops,
	.mode  = 0666,
};

static const struct proc_ops uaftest_proc_ops = {
	.proc_ioctl        = uaftest_ioctl,
	.proc_compat_ioctl = uaftest_ioctl,
	.proc_open         = simple_open,
	.proc_lseek        = noop_llseek,
};

static struct proc_dir_entry *uaftest_proc;

static int __init uaftest_init(void)
{
	int rc;

	pr_warn("uaftest: ENGINEERING TEST MODULE — do not ship\n");
	rc = misc_register(&uaftest_dev);
	if (rc)
		return rc;
	/* ueventd downgrades /dev/uaftest to 0600. /proc/uaftest keeps
	 * the mode we set here, giving the shell user a way in. */
	uaftest_proc = proc_create("uaftest", 0666, NULL, &uaftest_proc_ops);
	if (!uaftest_proc) {
		misc_deregister(&uaftest_dev);
		return -ENOMEM;
	}
	return 0;
}

static void __exit uaftest_exit(void)
{
	if (uaftest_proc)
		proc_remove(uaftest_proc);
	misc_deregister(&uaftest_dev);
}

module_init(uaftest_init);
module_exit(uaftest_exit);

MODULE_DESCRIPTION("Test driver to check if KASAN is working");
MODULE_LICENSE("GPL v2");
```

And the userspace trigger, `private/google-modules/misc/uaftest/uaftest_trigger.c`:

```c
// SPDX-License-Identifier: GPL-2.0-only

#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>

#define UAFTEST_MAGIC 'U'
#define UAFTEST_IOC_UAF_READ    _IO(UAFTEST_MAGIC, 1)
#define UAFTEST_IOC_UAF_WRITE   _IO(UAFTEST_MAGIC, 2)
#define UAFTEST_IOC_DOUBLE_FREE _IO(UAFTEST_MAGIC, 3)
#define UAFTEST_IOC_OOB_READ    _IO(UAFTEST_MAGIC, 4)
#define UAFTEST_IOC_GIVE_ROOT   _IO(UAFTEST_MAGIC, 100)

static const struct {
	const char *name;
	unsigned long cmd;
	const char *desc;
} CMDS[] = {
	{ "uaf-read",    UAFTEST_IOC_UAF_READ,    "kmalloc -> kfree -> read (UAF read)" },
	{ "uaf-write",   UAFTEST_IOC_UAF_WRITE,   "kmalloc -> kfree -> write (UAF write)" },
	{ "double-free", UAFTEST_IOC_DOUBLE_FREE, "kmalloc -> kfree -> kfree" },
	{ "oob-read",    UAFTEST_IOC_OOB_READ,    "read one byte past kmalloc end" },
	{ "give-root",   UAFTEST_IOC_GIVE_ROOT,   "set current task creds to uid 0" },
};

static void usage(const char *prog)
{
	size_t i;
	fprintf(stderr, "usage: %s <command>\n  commands:\n", prog);
	for (i = 0; i < sizeof(CMDS)/sizeof(CMDS[0]); i++)
		fprintf(stderr, "    %-12s  %s\n", CMDS[i].name, CMDS[i].desc);
}

int main(int argc, char **argv)
{
	int fd, rc;
	size_t i;
	unsigned long cmd = 0;

	if (argc != 2) {
		usage(argv[0]);
		return 2;
	}
	for (i = 0; i < sizeof(CMDS)/sizeof(CMDS[0]); i++) {
		if (!strcmp(argv[1], CMDS[i].name)) {
			cmd = CMDS[i].cmd;
			break;
		}
	}
	if (!cmd) {
		usage(argv[0]);
		return 2;
	}

	/* /dev/uaftest gets 0600 root by ueventd; /proc/uaftest is 0666. */
	fd = open("/proc/uaftest", O_RDWR);
	if (fd < 0)
		fd = open("/dev/uaftest", O_RDWR);
	if (fd < 0) {
		fprintf(stderr, "open /proc/uaftest or /dev/uaftest: %s\n", strerror(errno));
		return 1;
	}

	if (cmd == UAFTEST_IOC_GIVE_ROOT) {
		rc = ioctl(fd, cmd, 0);
		if (rc < 0) {
			fprintf(stderr, "ioctl: %s\n", strerror(errno));
			return 1;
		}
		printf("uid=%u euid=%u — exec a shell to keep it: exec /system/bin/sh\n",
		       getuid(), geteuid());
		execl("/system/bin/sh", "sh", (char *)NULL);
		perror("execl");
		return 1;
	}

	rc = ioctl(fd, cmd, 0);
	if (rc < 0) {
		fprintf(stderr, "ioctl: %s\n", strerror(errno));
		return 1;
	}
	printf("ok — check `dmesg` for the KASAN report\n");
	return 0;
}
```

Cross-compile the trigger with the NDK in the kernel tree:

```sh
prebuilts/ndk-r23/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android31-clang \
    -static -O2 -o uaftest_trigger \
    private/google-modules/misc/uaftest/uaftest_trigger.c
```


## Setting boot cmdline parameters on Pixel devices

`/proc/cmdline` after first boot has `kasan=off` and `arm64.nomte` appended after our dts cmdline, which kills runtime KASAN and MTE. `kasan=off` is in the `vendor_cmdline` field of `vendor_boot.img`'s header; `arm64.nomte` is appended by the bootloader.

For `arm64.nomte` there's a bootloader knob:

```sh
fastboot oem mte on
```

For `kasan=off` you have to edit `vendor_boot.img`. My kernel build didn't produce one so I pulled it from the matching factory image:

```sh
unzip image-shiba-cp1a.260405.005.zip vendor_boot.img init_boot.img vbmeta.img

python3 tools/mkbootimg/unpack_bootimg.py --boot_img vendor_boot.img --out vb
```

The `vendor command line args` value is logged to stdout by `unpack_bootimg.py`. Strip `kasan=off` and `rodata=on`, append `enforcing=0 kasan=on`, repack.

```sh
NEW_CMDLINE='fips140.load_sequential=1 exynos_drm.load_sequential=1 \
g2d.load_sequential=1 samsung_iommu_v9.load_sequential=1 swiotlb=noforce \
disable_dma32=on earlycon=exynos4210,0x10870000 console=ttySAC0,115200 \
androidboot.console=ttySAC0 printk.devkmsg=on cma_sysfs.experimental=Y \
rcupdate.rcu_expedited=1 rcu_nocbs=all rcutree.enable_rcu_lazy swiotlb=1024 \
cgroup.memory=nokmem sysctl.kernel.sched_pelt_multiplier=4 \
at24.write_timeout=100 log_buf_len=1024K android_arch_task_struct_size=512 \
bootconfig enforcing=0 kasan=on'

python3 tools/mkbootimg/mkbootimg.py \
    --header_version 4 --pagesize 0x800 \
    --vendor_cmdline "$NEW_CMDLINE" \
    --base 0x0 \
    --kernel_offset 0x10008000 --ramdisk_offset 0x11000000 \
    --tags_offset 0x10000100 --dtb_offset 0x11f00000 \
    --ramdisk_type platform --ramdisk_name '' \
        --vendor_ramdisk_fragment vb/vendor_ramdisk00 \
    --ramdisk_type none --ramdisk_name '16K' \
        --vendor_ramdisk_fragment vb/vendor_ramdisk01 \
    --vendor_bootconfig vb/bootconfig \
    --vendor_boot vendor_boot_new.img
```


## Console/serial

P.S. If you have a SuzyQ cable or a USB cereal and would like to dump bootloader and kernel logs to tty, do it here — append your serial console params to `NEW_CMDLINE` above before repacking, or set them via `fastboot oem uart` after flashing:

```sh
fastboot oem uart enable
fastboot oem uart config 3000000
```


## Flashing

```sh
fastboot -w --disable-verification
fastboot oem disable-verification
fastboot flash boot              out/shusky/dist/boot.img
fastboot flash dtbo              out/shusky/dist/dtbo.img
fastboot flash vendor_kernel_boot out/shusky/dist/vendor_kernel_boot.img
fastboot flash vendor_boot       vendor_boot_new.img
fastboot flash vbmeta            vbmeta.img

fastboot oem uart enable
fastboot oem uart config 3000000
fastboot oem mte on

fastboot reboot fastboot
# wait until `fastboot getvar is-userspace` says yes

fastboot flash vendor_dlkm out/shusky/dist/vendor_dlkm.img
fastboot flash system_dlkm out/shusky/dist/system_dlkm.img
fastboot reboot
```

`vendor_dlkm` and `system_dlkm` are logical partitions inside `super` and can only be resized from `fastbootd`, hence the bounce.


## Verifying it works

```sh
adb shell getenforce
adb shell cat /proc/uaftest
adb push uaftest_trigger /data/local/tmp/
adb shell chmod +x /data/local/tmp/uaftest_trigger
adb shell /data/local/tmp/uaftest_trigger uaf-read
```

dmesg report (read it after `give-root`):

```
[  229.126588] uaftest: triggering UAF read
[  229.127765] BUG: KASAN: invalid-access in uaftest_uaf_read+0x6c [uaftest]
[  229.128456] Read at addr f9ffff88e6653610 by task uaftest_trigger/13318
[  229.128485] Pointer tag: [f9], memory tag: [fe]
[  229.132481] Hardware name: ZUMA SHIBA MP based on ZUMA (DT)
…
```

If you're seeing something like the above, that means KASAN and MTE are doing their job.

## Root via the driver

`adb root` won't work on a `user` build (`ro.debuggable=0`), and production Android does not ship `su` or `sudo` in `$PATH` (userdebug builds do). Luckily, our driver's `GIVE_ROOT` ioctl gives us root inside an existing shell:

```sh
adb shell /data/local/tmp/uaftest_trigger give-root
# uid=0(root) gid=0(root)
```

I created a `su` script for convenience:

```sh
#!/system/bin/sh
exec /data/local/tmp/uaftest_trigger give-root
```

```sh
adb push su /data/local/tmp/su
adb shell chmod +x /data/local/tmp/su
adb shell /data/local/tmp/su -c id
```

`/data/local/tmp` isn't on `$PATH`, so either always type the full path or `export PATH=/data/local/tmp:$PATH` per session.


Note: `adbd` itself runs as `shell` because the system image has `ro.debuggable=0`. `adb push` to read-only partitions still fails as a non root user. If this bothers you use Magisk to patch `init_boot.img` from the factory image and flash it.


