# Samsung Galaxy M30s – Magisk Kernel

Custom kernel for the **Samsung Galaxy M30s (Exynos 9611)** with **Magisk initramfs baked-in**.  
Provides **untethered Magisk root** for system-as-root (no-ramdisk) devices, so you don’t have to rely on recovery-patched boots anymore.

---

## Installation
1. Download the flashable zip from [Releases](../../releases).  
2. Flash with **TWRP** (you may need to flash magisk zip before to provide binaries for Anykernel3 do it only when you face no ramdisk error).  
3. Boot system → Magisk app will detect root .  
4. (Optional) Enable **Zygisk** in Magisk settings if you need process injection or modules like Shamiko.  

---

## Notes
- This kernel includes **Magisk by default**.  
- **KernelSU** is supported but set to n in defconfig it is not recommended to enable it in this build as it's there for previous builds purposes.  
- Compatible only with **Galaxy M30s (SM-M307)**. Do **not** flash on other devices.  
- XDA Forum: https://xdaforums.com/t/kernel-magisk-untethered-magisk-boot-image-for-galaxy-m30s.4762170/
---

## Building
```bash
cd m30s-magisk-kernel
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-android-
make magisk_defconfig
make -j$(nproc)
```
---

## Credits
- [Magisk by topjohnwu](https://github.com/topjohnwu/Magisk)  
- [AnyKernel3 by osm0sis](https://github.com/osm0sis/AnyKernel3)  

---

## Disclaimer
I am **not responsible** for bricked devices, dead SD cards, or thermonuclear war.  
You are choosing to make these modifications proceed at your own risk.  
