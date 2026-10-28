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
> **Switch to:** :nerd_face:[Advanced Version](advanced.md)

This guide provides a step by step walkthrough to recompiling the WSL2 Linux kernel to include `mac80211_hwsim` support. For any kind of wireless emulation tool such as [Mininet-WiFi](https://intrig.dca.fee.unicamp.br/mininetwifi/) or [MiniNDN-WiFi](https://github.com/named-data/mini-ndn) where these changes are required to run on Windows 10/11 via [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install), as the default Microsoft kernel lacks wireless simulation modules.

> [!IMPORTANT]
> **Scope:** This guide currently covers **IBSS (ad-hoc) mode**, which gives full control over L2/L3 routing without hidden kernel mechanisms. Support for Mesh mode (802.11s/HWMP) will be added in a future version. Some mininet-wifi examples using Mesh, or AP/STA modes with encryption, may not run or may crash. WSL2 is intentionally kept lean and this guide follows that same principle.

## Index
<p>
  <a href="#env">Development environment &amp; versioning</a><br>
  <a href="#pre">Prerequisites</a><br>
  <a href="#step1">1. Get the Microsoft WSL2 Kernel source code</a><br>
  <a href="#step2">2. Configure the Microsoft WSL2 Kernel</a><br>
  &nbsp;&nbsp;&nbsp;&nbsp; <a href="#wireless">Wireless Stack Options</a><br>
  &nbsp;&nbsp;&nbsp;&nbsp; <a href="#regdb">Regulatory Domain Options</a><br>
  &nbsp;&nbsp;&nbsp;&nbsp; <a href="#firmware">Firmware Loader Options</a><br>
  &nbsp;&nbsp;&nbsp;&nbsp; <a href="#btf">Disable BTF (Preventing the 'Invalid Argument' error)</a><br>
  &nbsp;&nbsp;&nbsp;&nbsp; <a href="#check">Final Verification Checklist Before You Build</a><br>
  &nbsp;&nbsp;&nbsp;&nbsp; <a href="#save">Save and Exit</a><br>
  <a href="#step3">3. Build and Install</a><br>
  <a href="#step4">4. Move the Kernel to Windows</a><br>
  <a href="#step5">5. Configure WSL to use the New Kernel</a><br>
  <a href="#step6">6. Restart and Verify</a><br>
  <a href="#trouble">Quick troubleshooting guide</a><br>
</p>

<a name="env"></a>
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

<a name="pre"></a>
## Prerequisites
> [!NOTE]
> Throughout this guide when reading "tested OS" or simply "OS" it refers to the tested OS: Ubuntu 22.04 LTS

Inside your WSL2 instance install the build dependencies:

```bash
sudo apt update
sudo apt install build-essential flex bison libssl-dev libelf-dev bc dwarves libncurses-dev git cpio
```

Additionally, install the wireless regulatory database:

```bash
sudo apt install wireless-regdb
```

> [!IMPORTANT]
> This package must be installed **before** recompiling the kernel. The files `regulatory.db` and `regulatory.db.p7s` will be baked directly into the kernel binary in a later step (see [Firmware Loader Options](#firmware)). If those files are missing at build time the kernel will be compiled without a regulatory database and wireless channels will not be available.

Verify the files are present before proceeding:

```bash
ls -la /lib/firmware/regulatory.db /lib/firmware/regulatory.db.p7s
```

> [!NOTE]
> Expected output:
> ```
> -rw-r--r-- 1 root root 6328 Jan 15 02:20 /lib/firmware/regulatory.db
> -rw-r--r-- 1 root root 1085 Jan 15 02:20 /lib/firmware/regulatory.db.p7s
> ```

<a name="step1"></a>
## 1. Get the Microsoft WSL2 Kernel source code
Inside your WSL2 instance (tested OS) check your current kernel version with the command `uname -r`. \
Clone the official [Microsoft WSL2 Kernel repository](https://github.com/microsoft/WSL2-Linux-Kernel.git):
> [!NOTE]
> Ensure you are in a directory with enough space (at least 3GB are required).

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

<a name="step2"></a>
## 2. Configure the Microsoft WSL2 Kernel
Copy your current configuration:
```bash
zcat /proc/config.gz > .config
```

Open the menu with the command:
```bash
make menuconfig
```

<a name="wireless"></a>
#### Wireless Stack Options
Navigate through the menus and set the following options to `<M>` (Module) *(press 'M' on the keyboard)*:

`Networking support -> Wireless`

`cfg80211 - wireless configuration API` -> Set to `<M>`

`Generic IEEE 802.11 Networking Stack (mac80211)` -> Set to `<M>`

Select `< Exit >` and keep exiting until you are back at the main menu.

`Device Drivers -> Network device support -> Wireless LAN`

`Simulated radio testing tool for mac80211` -> Set to `<M>`

Select `< Exit >` and keep exiting until you are back at the main menu.

<a name="regdb"></a>
#### Regulatory Domain Options
Navigate back to `Networking support -> Wireless`. Enable the regulatory domain certification options *(press 'Y' on the keyboard to set `[*]`)*:

`cfg80211 certification onus` -> Set to `[*]`

Once enabled, two sub-options become visible directly below it in the menu. Enable both:

&nbsp;&nbsp;&nbsp;&nbsp;`require regdb signature` -> Set to `[*]`

&nbsp;&nbsp;&nbsp;&nbsp;`allow regdb keys shipped with the kernel` -> Set to `[*]`

> [!NOTE]
> The **regulatory database** (`regulatory.db`) defines which Wi-Fi frequencies and power levels are legally permitted in each country. Without it, `cfg80211` falls back to a very restrictive set of channels that will block IBSS (ad-hoc) communication entirely.
>
> Enabling `require regdb signature` means the kernel will only accept a **cryptographically signed** regulatory database preventing tampering. Enabling `allow regdb keys shipped with the kernel` means the trusted public key is embedded directly into the kernel binary, so the kernel can verify the database at boot without relying on anything from the host filesystem at runtime.

Select `< Exit >` and keep exiting until you are back at the main menu.

<a name="firmware"></a>
#### Firmware Loader Options
Navigate to:

`Device Drivers -> Generic Driver Options -> Firmware loader`

`Firmware loading facility` -> Ensure it is set to `{*}` *(built-in, not a module)*

> [!NOTE]
> The curly braces `{*}` indicate the feature is **compiled directly into the kernel** rather than loaded as an external module. This is required because the regulatory database must be available the moment the kernel boots, before any module loading can occur.

Once inside the `Firmware loader` submenu, configure the following two options:

`Build named firmware blobs into the kernel binary` -> Enter the field and type: `regulatory.db regulatory.db.p7s`

> [!NOTE]
> This is a text input field. Press `Enter` on this option to open an input box and type the two filenames **separated by a single space**: `regulatory.db regulatory.db.p7s`. The kernel build system will read both files from `/lib/firmware` on your WSL2 machine and embed them directly inside the compiled `bzImage`. This is exactly why `wireless-regdb` must be installed before this build step, as these files must exist at build time.

`Firmware blobs root directory` -> Ensure it is set to `/lib/firmware`

Select `< Exit >` and keep exiting until you are back at the main menu.

<a name="btf"></a>
#### Disable BTF (Preventing the 'Invalid Argument' error)
> [!NOTE]
> To prevent kernel module signature mismatches (common in WSL2) it's better to disable BTF
> In many Linux environments, you just "Turn off BTF". By disabling the generation of BTF information, we are removing the "ID badge" preventing 'Invalid argument' (symbol mismatch) error.
> When we run `modprobe` later, the kernel will see a module without BTF data and, because we have checked "Allow loading modules with non-matching BTF", it will allow it to run and load the driver.

`Kernel hacking -> Compile-time checks and compiler options`

`Generate BTF type information` -> Set to <code>[ ]</code> // *Deselect (Off)*

:memo: **NOTE** Turning this off will make sub-options disappear; this is expected.

> [!IMPORTANT]
> This configuration selects hardware simulator as Modules \[M\], not built-in \[\*\].
> The reason we select it as Modules \[M\] has to do with how *[mininet-wifi](https://github.com/intrig-unicamp/mininet-wifi)* was built. If we build the driver directly into the kernel (\[\*\]), it will be always active. However, *[mininet-wifi](https://github.com/intrig-unicamp/mininet-wifi)* is scripted to explicitly run the command `modprobe mac80211_hwsim`. Since `modprobe` looks for external module files `.ko` and doesn't find one (because it's inside the kernel), it throws a FATAL error.
>
> Another reason is because *[mininet-wifi](https://github.com/intrig-unicamp/mininet-wifi)* often tries to pass arguments to that module (like the number of virtual radios) which only works if it's a module. If *[mininet-wifi](https://github.com/intrig-unicamp/mininet-wifi)* requires creating a specific number of radios (e.g., it expects 4 radios but built-in only gives you 1 radio), there for in this situation the Modules \[M\] option is the most modular and advantageous.
>
> To avoid this "trap" for using *[mininet-wifi](https://github.com/intrig-unicamp/mininet-wifi)*, we need to ensure that we configuration select hardware simulator as Modules \[M\] and then install those modules into your [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) filesystem.

<a name="check"></a>
#### Final Verification Checklist Before You Build
Before you select `<Save>` on the `menuconfig`, please double check if they are according to the table. This is the step that will allow `mininet-wifi` to "plug in" the radios:

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

If all the options are selected as confirmed in the table above, don't just exit the `menuconfig` we still have to save the `.config` file.

<a name="save"></a>
#### Save and Exit
Use the Right Arrow key to select `<Save>` at the bottom and press `Enter`. \
It will ask for a `filename`. Just leave it as `.config` and press `Enter`. \
Select `< Exit >` and keep exiting until you are back at the command prompt.

<a name="step3"></a>
## 3. Build and Install
```bash
# Clean previous attempts in case you have saved other options inadvertently
make clean

# Build the kernel and modules (using all CPU cores)
make -j$(nproc)
sudo make modules_install
```

:hourglass: Have some coffee or tea because this step will take some time depending on your machine.

<a name="step4"></a>
## 4. Move the Kernel to Windows
> [!NOTE]
> The field `<YourUsername>` should be replaced by your Windows username.

Identify your Windows username and copy the compiled `bzImage`:

```bash
mkdir -p /mnt/c/Users/<YourUsername>/wsl-kernel
cp arch/x86/boot/bzImage /mnt/c/Users/<YourUsername>/wsl-kernel/vmlinux_wifi
```

<a name="step5"></a>
## 5. Configure WSL to use the New Kernel
In Windows, go to `%USERPROFILE%` (Usually in path: `C:\Users<YourUsername>`).

> [!NOTE]
> To create the file `.wslconfig` it's easier if you have hidden files and folders visible.
> There are many tutorials for that, but in short, open `File Explorer`, click the `View` menu at the top, select `Show`, and check Hidden items.

Create (notepad) or edit (Command Prompt) a file named `.wslconfig`.

Add the following content to `.wslconfig`:

```filename
[wsl2]
kernel=C:\\Users\\<YourUsername>\\wsl-kernel\\vmlinux_wifi
```
Save the file and exit to the Command Prompt.

<a name="step6"></a>
## 6. Restart and Verify

#### Close WSL
Open PowerShell and run:

```bash
wsl --shutdown
```

#### Re-open WSL.

Verify that the `mac80211_hwsim` module can be loaded:

```bash
sudo modprobe mac80211_hwsim
```

Verify all three wireless modules are active:

```bash
lsmod | grep -E '(mac80211|cfg80211|hwsim)'
```

Verify that virtual radio interfaces were created:

```bash
iw list
```

Verify that the regulatory database is loaded and a domain is active:

```bash
iw reg get
```

> [!NOTE]
> `iw reg get` should return a country entry such as `country 00:` rather than an error. If it returns nothing or an error, check that `wireless-regdb` was installed **before** building and that `regulatory.db regulatory.db.p7s` was entered correctly in the firmware blobs field during `menuconfig`.

Verify the regulatory database files are present on the filesystem:

```bash
ls -la /lib/firmware/regulatory.db /lib/firmware/regulatory.db.p7s
```

> [!NOTE]
> Expected output:
> ```
> -rw-r--r-- 1 root root 6328 Jan 15 02:20 /lib/firmware/regulatory.db
> -rw-r--r-- 1 root root 1085 Jan 15 02:20 /lib/firmware/regulatory.db.p7s
> ```

If `lsmod` shows all three modules, `iw list` shows virtual interfaces, and `iw reg get` shows a regulatory domain, the setup is ready.

##### You're now able to run any kind of wireless emulation tool such as **Mininet-WiFi** or **MiniNDN-WiFi**.


<br><br>
<a name="trouble"></a>
## Quick troubleshooting guide
<br>

```terminal
Error: modprobe: FATAL: Module mac80211_hwsim not found
```
> Ensure you ran `sudo make modules_install` inside the kernel source directory. If the kernel version has a + suffix (e.g., 6.6.87+), ensure the folder in `/lib/modules/` matches exactly what `uname -r` reported.

<br>

```terminal
Error: modprobe: ERROR: could not insert 'mac80211_hwsim': Invalid argument
```
> This is usually caused by BTF mismatch. Ensure you disabled Generate BTF type information in `menuconfig` and performed a `make clean` before recompiling.
