# Building scx-lavd for Android Devices

This guide provides comprehensive instructions for building scx-lavd (and other sched_ext schedulers) for Android devices and integrating them into an Android kernel.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Understanding the Requirements](#understanding-the-requirements)
- [Setting Up the Build Environment](#setting-up-the-build-environment)
- [Cross-Compiling for Android](#cross-compiling-for-android)
- [Kernel Configuration](#kernel-configuration)
- [Building the Android Kernel with sched_ext](#building-the-android-kernel-with-sched_ext)
- [Deploying to Android Device](#deploying-to-android-device)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Host System Requirements

- Linux host system (Ubuntu 20.04+ or similar recommended)
- At least 16GB RAM (32GB recommended for kernel builds)
- At least 100GB free disk space
- Root/sudo access for some operations

### Required Tools and Dependencies

Install the following on your host system:

```bash
# On Ubuntu/Debian
sudo apt update
sudo apt install -y \
    build-essential \
    git \
    curl \
    cmake \
    clang-17 \
    llvm-17 \
    pkg-config \
    libelf-dev \
    libssl-dev \
    libz-dev \
    libzstd-dev \
    libbpf-dev \
    bpftool \
    python3 \
    python3-pip \
    flex \
    bison \
    bc \
    pahole \
    protobuf-compiler \
    libseccomp-dev

# Install Rust toolchain (1.82 or newer)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
rustup default stable
rustup target add aarch64-linux-android
```

### Android-Specific Requirements

- Android NDK (r26 or newer recommended)
- Android kernel source code (with sched_ext support)
- ADB (Android Debug Bridge) for deployment

```bash
# Download and extract Android NDK
cd ~
wget https://dl.google.com/android/repository/android-ndk-r26d-linux.zip
unzip android-ndk-r26d-linux.zip
export ANDROID_NDK_HOME=~/android-ndk-r26d
export PATH=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin:$PATH
```

---

## Understanding the Requirements

### What is sched_ext?

`sched_ext` is a Linux kernel feature that enables implementing kernel thread schedulers in BPF and dynamically loading them. It's available in Linux kernel 6.12 and later.

### What is scx-lavd?

`scx_lavd` is a Latency-criticality Aware Virtual Deadline (LAVD) scheduler that's particularly well-suited for interactive workloads like gaming and UI applications - making it ideal for Android devices.

### Key Components

1. **BPF Code**: The scheduling logic written in eBPF
2. **Userspace Binary**: The Rust application that loads and manages the BPF scheduler
3. **Kernel Support**: A Linux kernel with `CONFIG_SCHED_CLASS_EXT=y`

---

## Setting Up the Build Environment

### 1. Clone the scx Repository

```bash
cd ~
git clone https://github.com/sched-ext/scx.git
cd scx
export SCX_ROOT=$(pwd)
```

### 2. Set Up Android NDK Environment

Create a standalone toolchain configuration:

```bash
# Export Android NDK environment variables
export ANDROID_NDK_HOME=~/android-ndk-r26d
export ANDROID_API=33  # Android 13, adjust as needed
export ANDROID_TARGET=aarch64-linux-android

# Add NDK toolchain to PATH
export PATH=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin:$PATH

# Set up cross-compilation tools
export CC=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/${ANDROID_TARGET}${ANDROID_API}-clang
export CXX=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/${ANDROID_TARGET}${ANDROID_API}-clang++
export AR=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-ar
export RANLIB=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-ranlib
export STRIP=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-strip
```

### 3. Configure Cargo for Cross-Compilation

Create or edit `~/.cargo/config.toml`:

```toml
[target.aarch64-linux-android]
linker = "aarch64-linux-android33-clang"
ar = "llvm-ar"
rustflags = [
    "-C", "link-arg=-fuse-ld=lld",
]

[env]
CARGO_TARGET_AARCH64_LINUX_ANDROID_LINKER = "aarch64-linux-android33-clang"
```

**Note**: If you're using a different Android API level (set in `$ANDROID_API`), update the `33` in both `linker` paths to match your API level. For example, for Android 11 (API 30):
```toml
linker = "aarch64-linux-android30-clang"
```
and
```toml
CARGO_TARGET_AARCH64_LINUX_ANDROID_LINKER = "aarch64-linux-android30-clang"
```

---

## Cross-Compiling for Android

### Building libbpf for Android

Since Android doesn't ship with libbpf, you'll need to cross-compile it:

```bash
cd $SCX_ROOT
mkdir -p android-build
cd android-build

# Clone and build libbpf
git clone https://github.com/libbpf/libbpf.git
cd libbpf/src

# Cross-compile libbpf
make \
    CC=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android${ANDROID_API}-clang \
    AR=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-ar \
    DESTDIR=$SCX_ROOT/android-build/sysroot \
    prefix=/usr \
    install

export PKG_CONFIG_PATH=$SCX_ROOT/android-build/sysroot/usr/lib64/pkgconfig:$PKG_CONFIG_PATH
export PKG_CONFIG_SYSROOT_DIR=$SCX_ROOT/android-build/sysroot
```

### Building scx-lavd for Android

```bash
cd $SCX_ROOT

# Set up environment for BPF compilation
export BPF_CLANG=clang-17
export BPF_CFLAGS="-g -O2 -Wall -Wno-compare-distinct-pointer-types -D__TARGET_ARCH_arm64 -mcpu=v3 -mlittle-endian"

# Note: If your Android kernel has older BPF support (pre-5.13), you may need to use -mcpu=v2 instead of v3
# export BPF_CFLAGS="-g -O2 -Wall -Wno-compare-distinct-pointer-types -D__TARGET_ARCH_arm64 -mcpu=v2 -mlittle-endian"

# Build scx_lavd for Android
cargo build \
    --release \
    --target aarch64-linux-android \
    --package scx_lavd

# The resulting binary will be at:
# target/aarch64-linux-android/release/scx_lavd
```

### Alternative: Building C Schedulers

If you want to build C-based schedulers:

```bash
cd $SCX_ROOT

# Build with cross-compilation settings
make all \
    CC=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android${ANDROID_API}-clang \
    BPF_CLANG=clang-17 \
    ARCH=arm64 \
    TARGET_ARCH=arm64

# Binaries will be in: build/scheds/c/
```

---

## Kernel Configuration

### Required Kernel Options

Your Android kernel must have the following options enabled. Add these to your kernel's `defconfig` or `.config` file:

```kconfig
# Core BPF support (mandatory)
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT=y
CONFIG_DEBUG_INFO_BTF=y
CONFIG_BPF_JIT_ALWAYS_ON=y
CONFIG_BPF_JIT_DEFAULT_ON=y

# sched_ext support (mandatory)
CONFIG_SCHED_CLASS_EXT=y

# Required by some schedulers
CONFIG_KALLSYMS_ALL=y

# Required on ARM64
# CONFIG_DEBUG_INFO_REDUCED is not set

# LAVD-specific: futex tracking for lock holder preemption avoidance
# LAVD tracks futex operations to provide additional time slices to futex holders,
# improving system-wide progress by avoiding lock holder preemption.
# This requires either ftrace (via FUNCTION_TRACER) or tracepoint support.
CONFIG_FUNCTION_TRACER=y

# Recommended for development/testing
CONFIG_SCHED_DEBUG=y
CONFIG_SCHED_AUTOGROUP=y
CONFIG_SCHED_CORE=y
CONFIG_SCHED_MC=y

# Preemption configuration
CONFIG_PREEMPT=y
CONFIG_PREEMPT_COUNT=y
CONFIG_PREEMPTION=y
CONFIG_PREEMPT_DYNAMIC=y
CONFIG_PREEMPT_RCU=y

# BPF events and tracing
CONFIG_BPF_EVENTS=y
CONFIG_FTRACE_SYSCALLS=y
CONFIG_HAVE_DYNAMIC_FTRACE=y
CONFIG_DYNAMIC_FTRACE=y
CONFIG_HAVE_KPROBES=y
CONFIG_KPROBES=y
CONFIG_KPROBE_EVENTS=y
CONFIG_ARCH_SUPPORTS_UPROBES=y
CONFIG_UPROBES=y
CONFIG_UPROBE_EVENTS=y
CONFIG_DEBUG_FS=y
```

### Obtaining a sched_ext-Enabled Kernel

#### Option 1: Use Upstream Kernel 6.12+

```bash
# Clone mainline kernel
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
git checkout v6.12  # or later version
```

#### Option 2: Use sched_ext Development Kernel

```bash
# Clone sched_ext for-next kernel
git clone -b sched_ext/for-next https://git.kernel.org/pub/scm/linux/kernel/git/tj/sched_ext.git
cd sched_ext
```

#### Option 3: Patch Your Android Kernel

If you're using a vendor Android kernel (e.g., from Google, Qualcomm, MediaTek):

1. Ensure your base kernel is 6.12 or newer
2. Apply sched_ext patches if not already included
3. Merge necessary device tree and driver changes from your vendor kernel

---

## Building the Android Kernel with sched_ext

### 1. Configure the Kernel

```bash
cd /path/to/your/android-kernel
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-android-

# Start with your device's defconfig
make ARCH=arm64 <your_device>_defconfig

# Or start with a generic ARM64 config
make ARCH=arm64 defconfig

# Add the required sched_ext options
cat >> .config << EOF
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT=y
CONFIG_DEBUG_INFO_BTF=y
CONFIG_BPF_JIT_ALWAYS_ON=y
CONFIG_BPF_JIT_DEFAULT_ON=y
CONFIG_SCHED_CLASS_EXT=y
CONFIG_KALLSYMS_ALL=y
CONFIG_FUNCTION_TRACER=y
EOF

# Update configuration with dependencies
make ARCH=arm64 olddefconfig
```

### 2. Build the Kernel

```bash
# Build kernel image and modules
make ARCH=arm64 \
    CROSS_COMPILE=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android- \
    -j$(nproc)

# Build device tree blobs (if needed)
make ARCH=arm64 \
    CROSS_COMPILE=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android- \
    dtbs
```

### 3. Verify BTF Generation

BTF (BPF Type Format) is crucial for sched_ext schedulers:

```bash
# Check if vmlinux has BTF information
readelf -S vmlinux | grep BTF

# You should see sections like:
# .BTF
# .BTF.ext
```

If BTF sections are missing:

```bash
# Ensure pahole is installed (version 1.16+)
pahole --version

# Rebuild with BTF generation
make ARCH=arm64 vmlinux
```

---

## Deploying to Android Device

### 1. Flash the Custom Kernel

The method depends on your device:

#### Method A: Using Fastboot (most common)

```bash
# Boot into bootloader
adb reboot bootloader

# Flash boot image (contains kernel)
fastboot flash boot boot.img

# Or boot without flashing (for testing)
fastboot boot boot.img

# Reboot
fastboot reboot
```

#### Method B: Using TWRP/Custom Recovery

1. Package your kernel as a flashable ZIP
2. Copy to device
3. Flash via recovery

#### Method C: Using Magisk (for rooted devices)

```bash
# Extract boot.img from device
adb shell "dd if=/dev/block/by-name/boot of=/sdcard/boot.img"
adb pull /sdcard/boot.img

# Patch with Magisk and your custom kernel
# Flash the patched image
fastboot flash boot magisk_patched.img
```

### 2. Push scx-lavd Binary to Device

```bash
# Ensure device is connected
adb devices

# Remount system as read-write (requires root)
adb root
adb remount

# Push the binary
adb push target/aarch64-linux-android/release/scx_lavd /system/bin/
adb shell chmod +x /system/bin/scx_lavd

# Alternatively, push to /data/local/tmp for testing
adb push target/aarch64-linux-android/release/scx_lavd /data/local/tmp/
adb shell chmod +x /data/local/tmp/scx_lavd
```

### 3. Verify Kernel Support

```bash
# Check if sched_ext is enabled
adb shell "cat /sys/kernel/sched_ext/state"
# Should output: disabled (meaning sched_ext is available but not active)

# Check kernel version
adb shell uname -r
# Should be 6.12 or higher
```

### 4. Run scx-lavd

```bash
# Start an interactive shell
adb shell

# Run scx_lavd (requires root)
su
/system/bin/scx_lavd

# Or with parameters
/system/bin/scx_lavd --stats 5

# To monitor statistics, use --monitor in a separate terminal
# /system/bin/scx_lavd --monitor 5
```

### 5. Verify Scheduler is Running

```bash
# In another terminal/shell
adb shell "cat /sys/kernel/sched_ext/state"
# Should output: enabled

adb shell 'cat /sys/kernel/sched_ext/*/ops'
# Should output: lavd
# (The wildcard * matches the scheduler ID directory)
```

### 6. Making it Persistent (Optional)

To run scx-lavd automatically on boot:

#### Using init.rc

Create `/system/etc/init/scx_lavd.rc`:

```rc
service scx_lavd /system/bin/scx_lavd
    class main
    user root
    group root
    oneshot
    disabled

on property:sys.boot_completed=1
    start scx_lavd
```

#### Using Magisk Module

Create a Magisk module structure:

```
magisk-scx/
├── META-INF/
│   └── com/
│       └── google/
│           └── android/
│               ├── update-binary
│               └── updater-script
├── system/
│   └── bin/
│       └── scx_lavd
├── service.sh
└── module.prop
```

In `service.sh`:

```bash
#!/system/bin/sh
# Wait for boot to complete
while [ "$(getprop sys.boot_completed)" != "1" ]; do
    sleep 1
done

# Start scx_lavd
/system/bin/scx_lavd &
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. "BPF syscall not available"

**Problem**: Kernel doesn't have BPF support enabled.

**Solution**: Rebuild kernel with `CONFIG_BPF_SYSCALL=y`.

```bash
# Verify BPF is available
adb shell "ls -l /proc/sys/kernel/bpf"
```

#### 2. "sched_ext not supported"

**Problem**: Kernel doesn't have sched_ext enabled.

**Solution**: 
- Ensure kernel version is 6.12+
- Rebuild with `CONFIG_SCHED_CLASS_EXT=y`

```bash
# Check kernel config
adb shell "zcat /proc/config.gz | grep SCHED_CLASS_EXT"
# or
adb shell "cat /proc/config.gz | gunzip | grep SCHED_CLASS_EXT"
```

#### 3. "BTF not found" or "vmlinux BTF not found"

**Problem**: Kernel compiled without BTF debug information.

**Solution**:
- Install pahole (version 1.16+)
- Rebuild kernel with `CONFIG_DEBUG_INFO_BTF=y`
- Ensure BTF is generated: `readelf -S vmlinux | grep BTF`

#### 4. Binary won't execute: "cannot execute binary file"

**Problem**: Wrong architecture or missing dynamic libraries.

**Solution**:

```bash
# Check binary architecture
file target/aarch64-linux-android/release/scx_lavd
# Should show: "ARM aarch64"

# Check dynamic library dependencies
adb shell "readelf -d /system/bin/scx_lavd"

# If missing libraries, push them or build statically:
cargo build --release --target aarch64-linux-android --package scx_lavd \
    --config "target.aarch64-linux-android.rustflags=['-C', 'target-feature=+crt-static']"
```

#### 5. "Permission denied" when running scheduler

**Problem**: Insufficient permissions.

**Solution**:
- Ensure you're running as root: `adb root`
- Check SELinux status: `adb shell getenforce`
- If enforcing, temporarily set permissive **for testing only**:
  ```bash
  adb shell setenforce 0
  ```
  
  **⚠️ WARNING**: Setting SELinux to permissive mode is a **significant security risk** and should **NEVER** be used in production environments. This is only for testing purposes. For production deployments, you must create proper SELinux policies for the scx_lavd binary and BPF programs. Consult the [Android SELinux documentation](https://source.android.com/docs/security/features/selinux) for policy creation guidelines.

#### 6. Scheduler loads but system becomes unstable

**Problem**: Scheduler configuration may not be optimal for mobile workload.

**Solution**:
- Start with conservative settings
- Use scx_lavd's autopilot mode
- Monitor performance with `scx_lavd --monitor 5` (displays live statistics)
- Or check statistics periodically with `scx_lavd --stats 5`
- Fall back to default scheduler using one of these methods:
  1. Kill the scx_lavd process: `adb shell killall scx_lavd`
  2. Use sysrq to reset scheduler: `adb shell "echo S > /proc/sysrq-trigger"`
  
**Note**: `sysrq-S` is the sched_ext-specific SysRq key that switches the system back to the default scheduler. This is different from the standard Linux SysRq emergency sync command.

#### 7. Cross-compilation linking errors

**Problem**: Can't find libbpf or other dependencies.

**Solution**:

```bash
# Verify libbpf was built for correct architecture
file $SCX_ROOT/android-build/sysroot/usr/lib64/libbpf.a
# Should show ARM aarch64

# Set PKG_CONFIG environment correctly
export PKG_CONFIG_SYSROOT_DIR=$SCX_ROOT/android-build/sysroot
export PKG_CONFIG_PATH=$SCX_ROOT/android-build/sysroot/usr/lib64/pkgconfig

# Try static linking
cargo rustc --release --target aarch64-linux-android --package scx_lavd -- \
    -C target-feature=+crt-static
```

### Debugging Tips

#### Enable Debug Logging

```bash
# Run with tracing enabled
RUST_LOG=debug /system/bin/scx_lavd

# Or redirect to file
RUST_LOG=debug /system/bin/scx_lavd 2>/data/local/tmp/scx_lavd.log
```

#### Check Kernel Logs

```bash
# Monitor kernel logs
adb logcat -b kernel

# Or use dmesg
adb shell dmesg | grep -i "sched\|bpf"
```

#### Use BPF Tools

```bash
# Check loaded BPF programs
adb shell bpftool prog list

# Check BPF maps
adb shell bpftool map list

# Dump scheduler struct_ops
adb shell bpftool struct_ops list
```

### Performance Monitoring

```bash
# Monitor scheduler statistics (live updates)
adb shell "/system/bin/scx_lavd --monitor 5"

# Check periodic statistics from the scheduler instance
adb shell "/system/bin/scx_lavd --stats 5"

# Check CPU utilization
adb shell top

# Monitor scheduler events
adb shell "cat /sys/kernel/debug/tracing/events/sched/enable"
adb shell "echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable"
adb shell "cat /sys/kernel/debug/tracing/trace_pipe"
```

---

## Additional Resources

- **sched_ext Documentation**: https://github.com/sched-ext/scx
- **Linux Kernel sched_ext Docs**: https://docs.kernel.org/scheduler/sched-ext.html
- **Android Kernel Development**: https://source.android.com/docs/core/architecture/kernel
- **BPF Documentation**: https://docs.kernel.org/bpf/
- **scx_lavd Details**: See [scheds/rust/scx_lavd/README.md](scheds/rust/scx_lavd/README.md)

## Community Support

- **GitHub Issues**: https://github.com/sched-ext/scx/issues
- **Discord**: https://discord.gg/b2J8DrWa7t
- **Mailing List**: sched-ext@lists.linux.dev

---

## Notes

- **Warranty**: Building and flashing custom kernels can brick your device. Proceed at your own risk.
- **Backups**: Always backup your device before flashing custom kernels.
- **Testing**: Test on a non-production device first.
- **SELinux**: You may need to adjust SELinux policies for production use.
- **Battery**: Monitor battery usage, as some scheduler configurations may affect power consumption.
- **Compatibility**: Not all Android devices support custom kernels. Check XDA forums for your device.

## License

This documentation follows the same license as the scx project: GPL-2.0-only
