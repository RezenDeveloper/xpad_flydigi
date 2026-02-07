> [!WARNING]
> **EXPERIMENTAL / WORK IN PROGRESS**
>
> This driver is **still under active testing**.
> It interacts directly with USB HID and vendor-specific endpoints and **may crash your system, lock up USB devices, or require a hard reboot**.
>
> Use at your own risk.

# xpad_flydigi
`xpad_flydigi` is a fork of the Linux `xpad` driver that exposes **Flydigi’s** vendor-specific output interface while keeping normal controller input fully functional.

It lets you send raw command bytes directly to **Flydigi** controllers without losing button or axis input, enabling things like real-time trigger mode switchin and in-game trigger mods.

This makes it suitable for game mods that need control over Flydigi-specific features that are not supported by the upstream xpad driver.

## Installing

```bash
sudo git clone https://github.com/RezenDeveloper/xpad_flydigi /usr/src/xpad_flydigi-0.1
sudo dkms install -m xpad_flydigi -v 0.1
```

### Prevent the default **xpad** driver from loading
Since both `xpad` and `xpad_flydigi` cannot attach to the same USB device simultaneously, you need to block `xpad` from binding to your Flydigi controller.

```bash
sudo tee /etc/modprobe.d/blacklist-xpad.conf <<'EOF'
blacklist xpad
EOF
```

### Apply the new rules
Update the initramfs to apply the blacklist:

```bash
# mkinitcpio (for Arch-based systems) or equivalent initramfs tool
sudo mkinitcpio -P
sudo systemctl restart systemd-modules-load.service
```

`xpad_flydigi` retains the core functionality of `xpad`, so other devices that normally use `xpad` will continue to work.

## Updating
```
cd /usr/src/xpad_flydigi-0.1
sudo git fetch
sudo git checkout origin/master
sudo dkms remove -m xpad_flydigi -v 0.1 --all
sudo dkms install -m xpad_flydigi -v 0.1
```

## Removing
```
sudo dkms remove -m xpad_flydigi -v 0.1 --all
sudo rm -rf /usr/src/xpad_flydigi-0.1
```

## Usage

After installing and connecting your controller, `xpad_flydigi` will create new control files inside the USB device directory.

You can locate them with:
```bash
sudo find /sys/devices -name "flydigi_*" -type f
```

### **flydigi_trigger**

This file is used to configure trigger behavior on **Flydigi** controllers.
It mirrors the behavior of the **Flydigi Space Station** application by sending the same command structure and respecting the same value ranges.

Values sent outside the documented ranges are automatically clamped to safe limits.

### **Example**
```bash
echo "0 1 10 30 0 0 0" | sudo tee /sys/bus/usb/devices/1-4:1.0/flydigi_trigger > /dev/null
```

this file expects a 7-value payload using this format
```bash
<side> <mode> <p1> <p2> <p3> <p4> <p5>
```

### **Command format**
The file expects **7 integer values**, separated by spaces:

- **side (0-1)**
    - 0 = left trigger.
    - 1 = right trigger.
- **mode (0-5)**
    - Trigger behavior mode:

|**value**|**mode**|
|---|---|
|0|Default|
|1|Race Mode|
|2|Recoil Mode|
|3|Sniper Mode|
|4|Lock Mode|
|5|Vibration Mode|

### **Mode Parameters**

Each mode uses a different subset of parameters.

Unused parameters are ignored and automatically set to 0.

#### **Default**
No parameters.
All values are ignored and reset to default behavior.

#### **Race**
|**Parameter**|**Description**|**Range**|
|---|---|---|
|p1|Initial trigger position|0–192|
|p2|Trigger pressure|1–255|

#### **Recoil**
|**Parameter**|**Description**|**Range**|
|---|---|---|
|p1|Vibration start position|0–192|
|p2|Initial recoil strength|1-255|
|p3|Recoil intensity once start position is reached|1-255|
|p4|Recoil frequency|1-255|
|p5|Only report trigger input after start position (`0` = false, `1` = true)|0-1|

#### **Sniper**
|**Parameter**|**Description**|**Range**|
|---|---|---|
|p1|Initial trigger position|0-192|
|p2|Trigger length|1-255|
|p3|Trigger pressure|1-255|
|p4|Only report trigger input after start position (`0` = false, `1` = true)|0-1|

#### **Lock**
|**Parameter**|**Description**|**Range**|
|---|---|---|
|p1|Lock start position|0–192|

#### **Vibration**
|**Parameter**|**Description**|**Range**|
|---|---|---|
|p1|Vibration intensity|0-200|
|p2|Vibration threshold (below this value no vibration occurs)|1-255|
|p3|Trigger travel range for sustained vibration feedback|1-200|
|p4|Vibration frequency|1-200|

### **flydigi_raw**
This file sends raw 15-byte packets directly to the controller, bypassing all validation, filtering, and safety checks.

> [!CAUTION]
> This interface can cause undefined behavior, firmware lockups, or permanent device damage.
> Only use this if you fully understand the Flydigi protocol.

### **Example**
```bash
printf '\xa5\x30\x06\x01\x02\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00' | sudo tee /sys/bus/usb/devices/1-4:1.0/flydigi_raw > /dev/null
```

Check `/reference` folder to see packets references from **Flydigi Space Station**

## Disclaimer

THIS SOFTWARE IS PROVIDED AS IS, WITHOUT WARRANTY OF ANY KIND.
THE AUTHOR IS NOT RESPONSIBLE FOR ANY DAMAGE TO HARDWARE, FIRMWARE, OR ACCESSORIES CAUSED BY USING THIS DRIVER.
