# How to make vendor_boot recoveries (WIP)

## On Windows
 - open a command window and run the command below
```bash
wsl --install
```
 - after this it will prompt you to set a username and password for your linux virtual machine
 - when you set your username and password it will load into WSL.

## Install git WSL and Linux
 - run the following command
```bash
apt install git-all
```
## Extracting the ramdisks of the stock vendor_boot (simplified)
- Clone this repo [extract-via-magiskboot](https://github.com/cd-Crypton/extract-via-magiskboot) Thanks to @cd-crypton for making this script, and special mention to @nilz3000 for making the modified version of magiskboot that supports vendor_boot.img.
- copy your stock vendor_boot.img into the extract-via-magiskboot folder
- run the following command from the roof folder of cloned repo with the vendor_boot.img inside
```bash
bash unpack.sh vendor_boot.img
```
## The Ramdisks
 - in the folder once extracted you will find a number of folders, in the BASE_IMG folder you will find your dtb, this is all you need in a vendor_boot recovery, theres no kernel or dtbo.
 - RAMDISK is your vendor ramdisk usually which should contain your first_stage_ramdisk and your vendor_boot modules.
 - RECOVERY is your recovery ramdisk, if you have made a recovery.img before, this is the same as your normal recovery and you will find the same contents.
 - there may be a 3rd ramdisk, this will be a DLKM ramdisk, this mainly houses DLKM modules (NOT FOUND ON ALL VENDOR_BOOT).
## The shortcut (Repurpose an AOSP tree) skip if you can manually writeup your makefiles.
 - have a dump of the device firmware handy, most dumps dont include this extracted vendor_boot content, but alot of vendor_boot's can't be run through twrpdtgen, so the workaround is to use the AOSP tree from your dump, and rework the files, i will go into detail on how.
 - once you have your aosp makefiles look through them and change every instance of the word "lineage" to "twrp"
 - you will find the following line in your "lineage_codename.mk" file:
```bash
# Inherit some common Lineage stuff.
$(call inherit-product, vendor/lineage/config/common_full_phone.mk)
```
- Swap this line out to the following
```bash
# Inherit some common twrp stuff.
$(call inherit-product, vendor/twrp/config/common.mk)
```
## Cleanup your aosp BoardConfig.mk
 - Remove the following and the files assosiated with the lines
```bash
TARGET_FORCE_PREBUILT_KERNEL := true
```
```bash
ifeq ($(TARGET_FORCE_PREBUILT_KERNEL),true)
```
```bash
TARGET_PREBUILT_KERNEL := $(DEVICE_PATH)/prebuilts/kernel
```
```bash
BOARD_INCLUDE_DTB_IN_BOOTIMG := 
BOARD_PREBUILT_DTBOIMAGE := $(DEVICE_PATH)/prebuilts/dtbo.img
BOARD_KERNEL_SEPARATED_DTBO := 
```
```bash
TARGET_PRODUCT_PROP += $(DEVICE_PATH)/product.prop
TARGET_SYSTEM_EXT_PROP += $(DEVICE_PATH)/system_ext.prop
TARGET_SYSTEM_DLKM_PROP += $(DEVICE_PATH)/system_dlkm.prop
TARGET_ODM_PROP += $(DEVICE_PATH)/odm.prop
TARGET_ODM_DLKM_PROP += $(DEVICE_PATH)/odm_dlkm.prop
TARGET_VENDOR_DLKM_PROP += $(DEVICE_PATH)/vendor_dlkm.prop
```
```bash
# VINTF
DEVICE_MANIFEST_FILE += $(DEVICE_PATH)/manifest.xml

# Inherit the proprietary files
include vendor/oem/codename/BoardConfigVendor.mk
```
 - as for the other prop files, you use them just like you would in a standard recovery
 - create the following directory
```bash
recovery/root
```
 - move the contents of the rootdir/etc into the recovery/root folder
 - delete the fstab from recovery/root
 - in the RECOVERY folder from your extracted ramdisks copy the following directory
```bash
system/etc/recovery.fstab
```
- to the root directory of your modified aosp tree
- delete the rootdir folder
- rename the prebuilts folder to prebuilt and in boardconfig change the dtb path to prebuilt too.
## vendor_boot flags
- Edit the following line in BoardConfig.mk
```bash
BOARD_BOOTIMG_HEADER_VERSION := 4
```
 - change it too the following
```bash
BOARD_BOOT_HEADER_VERSION := 4
```
## vendor_boot flags (BoardConfig.mk)
 - You will need to add these flags to your BoardConfig.mk
```bash
TARGET_NO_KERNEL := true
```
```bash
BOARD_USES_GENERIC_KERNEL_IMAGE := true
```
```bash
BOARD_MOVE_GSI_AVB_KEYS_TO_VENDOR_BOOT := true
BOARD_MOVE_RECOVERY_RESOURCES_TO_VENDOR_BOOT := true
BOARD_INCLUDE_RECOVERY_RAMDISK_IN_VENDOR_BOOT := true
```
 - also add these to device.mk (partitions may be different depending on device)
```bash
# A/B
$(call inherit-product, $(SRC_TARGET_DIR)/product/virtual_ab_ota/launch_with_vendor_ramdisk.mk)
AB_OTA_UPDATER := true

AB_OTA_PARTITIONS += \
    dtbo \
    boot \
    vendor_dlkm \
    init_boot \
    system \
    product \
    system_ext \
    vbmeta \
    system_dlkm \
    vendor \
    vbmeta_system \
    odm \
    vendor_boot

AB_OTA_POSTINSTALL_CONFIG += \
    RUN_POSTINSTALL_system=true \
    POSTINSTALL_PATH_system=system/bin/otapreopt_script \
    FILESYSTEM_TYPE_system=erofs \
    POSTINSTALL_OPTIONAL_system=true
```
 - from the folder named "RAMDISK" in your extracted vendor_boot.img copy the first_stage_ramdisk folder into prebuilt on the recovery tree and add the following to device.mk
```bash
$(LOCAL_PATH)/prebuilt/first_stage_ramdisk/fstab.vendor:$(TARGET_COPY_OUT_VENDOR_RAMDISK)/first_stage_ramdisk/fstab.vendor
