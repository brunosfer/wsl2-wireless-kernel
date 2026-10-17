<!--
title: Tutorial for building a custom WSL2 Kernel for WiFi Simulation
author: Bruno Fernandes (bruno.fernandes [at] tum [dot] de)
date: 13-05-2026
description: Required if Mininet-WiFi or MiniNDN-WiFi are to be used in WSL2.
-->

# Building a Custom WSL2 Kernel for WiFi Simulation

<!-- Keywords -->
**Keywords:** `wsl` `wifi` `emulation` `minindn-wifi` `mininet-wifi` `wmediumd`

> [!TIP]
> **Switch to:** :baby_bottle:[Beginner Version](beginner.md)

Welcome to the advanced version of this guide for recompiling the WSL2 Linux kernel to include `mac80211_hwsim` support for any kind of wireless emulation tool such as [Mininet-WiFi](https://intrig.dca.fee.unicamp.br/mininetwifi/) or [MiniNDN-WiFi](https://github.com/named-data/mini-ndn) where these changes are required to run on Windows 10/11 via [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install), as the default Microsoft kernel lacks wireless simulation modules.

> [!IMPORTANT]
> **Scope:** This guide currently covers **IBSS (ad-hoc) mode**, which gives full control over L2/L3 routing without hidden kernel mechanisms. Support for Mesh mode (802.11s/HWMP) will be added in a future version. Some mininet-wifi examples using Mesh, or AP/STA modes with encryption, may not run or may crash. WSL2 is intentionally kept lean and this guide follows that same principle.

## Index
<p>
  <a href="#dev">Development environment &amp; versioning</a><br>
  <a href="#prereq">Prerequisites</a><br>
  <a href="#stepone">1. Get the Microsoft WSL2 Kernel source code</a><br>
  <a href="#steptwo">2. Configure the Microsoft WSL2 Kernel and Build</a><br>
  <a href="#stepthree">3. Move the Kernel to Windows</a><br>
  <a href="#stepfour">4. Configure WSL to use the New Kernel</a><br>
  <a href="#stepfive">5. Restart and Verify</a><br>
</p>

<a name="dev"></a>
## Development environment & versioning
```bash
C:\Users\Author>wsl -v
WSL version: 2.6.3.0
Kernel version: 6.6.87.2-1
WSLg version: 1.0.71
MSRDC version: 1.2.6353
Direct3D version: 1.611.1-81528511
DXCore version: 10.0.26100.1-240331-1435.ge-release
Windows version: 10.0.26200.8246
```

<a name="prereq"></a>
## Prerequisites
> [!NOTE]
> Tested OS: Ubuntu 22.04 LTS

Inside your WSL2 instance install the build dependencies:

```bash
sudo apt update
sudo apt install build-essential flex bison libssl-dev libelf-dev bc dwarves libncurses-dev git wireless-regdb
```

> [!IMPORTANT]
> `wireless-regdb` must be installed **before** recompiling the kernel as these files will be embedded into the kernel binary. Verify they are present:
> ```bash
> ls -la /lib/firmware/regulatory.db /lib/firmware/regulatory.db.p7s
> ```

<a name="stepone"></a>
## 1. Get the Microsoft WSL2 Kernel source code

```bash
git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
cd WSL2-Linux-Kernel
```

> [!CAUTION]
> Checkout your specific version branch according to your kernel version.
> At the time of writing this guide, the version (branch) used was [linux-msft-wsl-6.6.87.2](https://github.com/microsoft/WSL2-Linux-Kernel.git).
> ```bash
> git checkout linux-msft-wsl-6.6.87.2
> ```

<a name="steptwo"></a>
## 2. Configure the Microsoft WSL2 Kernel and Build
Copy your current configuration:
```bash
zcat /proc/config.gz > .config
```

Open the menu with the command:
```bash
make menuconfig
```

Select the kernel options according to the table. This step will allow `mininet-wifi` to "plug in" the radios:

<table>
  <thead>
    <tr style="border-top: none; border-bottom: 2px solid #333;">
      <th align="left">Path</th>
      <th align="left">Feature</th>
      <th align="left">Setting</th>
    </tr>
  </thead>
  <tbody>
    <tr style="border: none;">
      <td><code>Networking support -> Wireless</code></td>
      <td><code>cfg80211 - wireless configuration API</code></td>
      <td><code>&lt;M&gt;</code></td>
    </tr>
    <tr style="border: none;">
      <td><code>Networking support -> Wireless</code></td>
      <td><code>Generic IEEE 802.11 Networking Stack (mac80211)</code></td>
      <td><code>&lt;M&gt;</code></td>
    </tr>
    <tr style="border: none;">
      <td><code>Device Drivers -> Network device support -> Wireless LAN</code></td>
      <td><code>Simulated radio testing tool for mac80211</code></td>
      <td><code>&lt;M&gt;</code></td>
    </tr>
    <tr style="border: none;">
      <td><code>Networking support -> Wireless</code></td>
      <td><code>cfg80211 certification onus</code></td>
      <td><code>[*]</code></td>
    </tr>
    <tr style="border: none;">
      <td><code>Networking support -> Wireless</code></td>
      <td><code>require regdb signature</code></td>
      <td><code>[*]</code></td>
    </tr>
    <tr style="border: none;">
      <td><code>Networking support -> Wireless</code></td>
      <td><code>allow regdb keys shipped with the kernel</code></td>
      <td><code>[*]</code></td>
    </tr>
    <tr style="border: none;">
      <td><code>Device Drivers -> Generic Driver Options -> Firmware loader</code></td>
      <td><code>Firmware loading facility</code></td>
      <td><code>{*}</code></td>
    </tr>
    <tr style="border: none;">
      <td><code>Device Drivers -> Generic Driver Options -> Firmware loader</code></td>
      <td><code>Build named firmware blobs into the kernel binary</code></td>
      <td><code>regulatory.db regulatory.db.p7s</code></td>
    </tr>
    <tr style="border: none;">
      <td><code>Device Drivers -> Generic Driver Options -> Firmware loader</code></td>
      <td><code>Firmware blobs root directory</code></td>
      <td><code>/lib/firmware</code></td>
    </tr>
    <tr style="border: none;">
      <td><code>Kernel hacking -> Compile-time checks and compiler options</code></td>
      <td><code>Generate BTF type information</code></td>
      <td><code>[ ]</code></td>
    </tr>
  </tbody>
</table>

Save the `.config` file.

> [!NOTE]
> The `Generate BTF type information` is set to <code>[ ]</code> to disable it. \
> `Build named firmware blobs into the kernel binary` is a text field. Press `Enter` on it and type `regulatory.db regulatory.db.p7s` (two filenames separated by a space). The build system reads them from `/lib/firmware` and embeds them into the kernel binary. \
> If you're interested in knowing more about the \[M\] option instead the \[\*\] option, or the regulatory domain settings, see the [Beginner Version](beginner.md).

#### Build and Install
```bash
# Clean previous attempts in case you have saved other options inadvertently
make clean

# Build the kernel and modules (using all CPU cores)
make -j$(nproc)
sudo make modules_install
```

:hourglass: Have some coffee or tea because this step will take some time depending on your machine.

<a name="stepthree"></a>
## 3. Move the Kernel to Windows
Identify your Windows username and copy the compiled `bzImage`:

```bash
mkdir -p /mnt/c/Users/<YourUsername>/wsl-kernel
cp arch/x86/boot/bzImage /mnt/c/Users/<YourUsername>/wsl-kernel/vmlinux_wifi
```

<a name="stepfour"></a>
## 4. Configure WSL to use the New Kernel
Create a file named `.wslconfig` in your windows user profile and add the following content:

```filename
[wsl2]
kernel=C:\\Users\\<YourUsername>\\wsl-kernel\\vmlinux_wifi
```
Save the file and exit to the Command Prompt.

<a name="stepfive"></a>
## 5. Restart and Verify

#### Close WSL
Open PowerShell and run:

```bash
wsl --shutdown
```

#### Re-open WSL.

Verify that the `mac80211_hwsim` module can be loaded:

```bash
sudo modprobe mac80211_hwsim
lsmod | grep -E '(mac80211|cfg80211|hwsim)'
iw list
iw reg get
ls -la /lib/firmware/regulatory.db /lib/firmware/regulatory.db.p7s
```

> [!NOTE]
> `iw reg get` should return a country entry (e.g. `country 00:`) rather than an error. If it fails, check that `wireless-regdb` was installed before building and that `regulatory.db regulatory.db.p7s` was entered correctly in the firmware blobs field.

If `lsmod` shows all three modules, `iw list` shows virtual interfaces, and `iw reg get` shows a regulatory domain, the setup is ready.

##### You're now able to run any kind of wireless emulation tool such as **Mininet-WiFi** or **MiniNDN-WiFi**.