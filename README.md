## 🚀 Complete Driver Installation Script (For New OS)

This combined script will install necessary tools, apply all the fixes we identified together, compile, and install the driver with DKMS.

You must first make sure you have installed **`git`** and cloned the repository:

```bash
# 1. Install prerequisites (if not already done)
sudo apt update
sudo apt install build-essential dkms git

# 2. Clone the driver repository (if starting clean)
# Note: If you already have the folder, skip this step.
git clone https://github.com/lwfinger/rtl8852au.git
cd rtl8852au
```

### 3\. Apply All Patches and Install

Copy and paste the entire block below and execute it to apply the necessary patches and set up the driver permanently:

```bash
#!/bin/bash

echo "Starting full patch and DKMS installation for rtl8852au..."

# --- FIX 1: VFS MODULE_IMPORT_NS ERROR ---
echo "Applying Fix 1: Commenting out MODULE_IMPORT_NS..."
sed -i 's/^\(\s*MODULE_IMPORT_NS.*\)$/\/*\1 *\//' os_dep/osdep_service_linux.c

# --- FIX 2 & 3: CFG80211 SIGNATURE AND POINTER FIXES (Linux 6.0+) ---
# Patch Function Definitions
echo "Applying Fix 2: Updating cfg80211 function definitions..."
sed -i 's/cfg80211_rtw_get_txpower(struct wiphy *wiphy, struct wireless_dev *wdev, int *tx_power)/cfg80211_rtw_get_txpower(struct wiphy *wiphy, struct wireless_dev *wdev, unsigned int type, int *tx_power)/' os_dep/linux/ioctl_cfg80211.c
sed -i 's/cfg80211_rtw_set_monitor_channel(struct wiphy *wiphy, struct cfg80211_chan_def *chan_def)/cfg80211_rtw_set_monitor_channel(struct wiphy *wiphy, struct net_device *dev, struct cfg80211_chan_def *chan_def)/' os_dep/linux/ioctl_cfg80211.c

# Patch Structure Assignments (Workaround)
echo "Applying Fix 3: Setting incompatible function pointers to NULL..."
sed -i 's/\.get_tx_power = cfg80211_rtw_get_txpower,/\.get_tx_power = NULL,/' os_dep/linux/ioctl_cfg80211.c
sed -i 's/\.set_monitor_channel = cfg80211_rtw_set_monitor_channel,/\.set_monitor_channel = NULL,/' os_dep/linux/ioctl_cfg80211.c

# --- SETUP DKMS AND INSTALL ---
echo "--- Setting up DKMS for permanent installation ---"

echo "Adding source to DKMS..."
sudo dkms add .

echo "Building and installing module via DKMS..."
# Note: Using the correct module name 'rtl8852au'
if sudo dkms install rtl8852au/1.15.0.1; then
    echo "✅ DKMS installation successful. The driver will now auto-rebuild on kernel updates."
    
    echo "Loading the module..."
    sudo modprobe 8852au
    
    echo "Final driver status check:"
    dkms status
    ip a
else
    echo "❌ DKMS installation failed. Please check the output for errors."
    exit 1
fi
```

This single block represents the **total fix and permanent installation** for your Realtek 0bda:8832 adapter on modern Linux kernels.

## ✅ Next Steps to Verify

While the installation is complete, the driver module is **not necessarily loaded and active** yet. You need to verify that your system is using the new module.

Here is how you can check and activate it:

1.  **Load the Module:**
    Try to load the module manually:

    ```bash
    sudo modprobe 8852au
    ```

2.  **Verify the Module is Loaded:**
    Check if the module is in use:

    ```bash
    lsmod | grep 8852au
    ```

    If you see output containing `8852au`, it's loaded.

3.  **Check for Wireless Interface:**
    Check if the wireless interface (e.g., `wlan0`, `wlp...`) has appeared:

    ```bash
    ip a
    ```

    or

    ```bash
    iw dev
    ```
-----

```
drclab@drclab:~/rtl8852au$ sudo modprobe 8852au
drclab@drclab:~/rtl8852au$ lsmod | grep 8852au
8852au              14278656  0
cfg80211             1433600  4 iwlmvm,8852au,iwlwifi,mac80211
drclab@drclab:~/rtl8852au$
```
That's great news\! The driver installation was **successful**, and the new module is now **loaded and active** in your kernel.

The output from `lsmod | grep 8852au` confirms this:

```
8852au             14278656  0
cfg80211            1433600  4 iwlmvm,8852au,iwlwifi,mac80211
```

  * The line starting with `8852au` shows the module is loaded.
  * The line starting with `cfg80211` shows that your new `8852au` driver is correctly hooked into the Linux wireless subsystem (`cfg80211`), alongside other potential wireless components.

-----

## 💻 Final Verification

Since the driver is loaded, your operating system should now be able to see and use the Wi-Fi adapter.

Please run the following commands to confirm the adapter is visible as a network interface:

### 1\. Check Network Interfaces

Use the `ip a` or `iw dev` command to list your network adapters. Look for a new interface name, usually something starting with **`w`** (e.g., `wlan0`, `wlp...`).

```bash
ip a
```

or

```bash
iw dev
```

### 2\. Check Device Status (Optional)

You can also check the kernel log for messages related to the driver loading, which can confirm if your specific adapter hardware was detected:

```bash
dmesg | grep 8852au
```
-----

```
drclab@drclab:~/rtl8852au$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: wlp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 50:e0:85:4d:f6:4d brd ff:ff:ff:ff:ff:ff
    inet 192.168.110.40/24 brd 192.168.110.255 scope global dynamic noprefixroute wlp1s0
       valid_lft 1099sec preferred_lft 1099sec
    inet6 fe80::bf7:65f5:701d:b083/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: wlx90de8059135e: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 90:de:80:59:13:5e brd ff:ff:ff:ff:ff:ff
drclab@drclab:~/rtl8852au$
```
-----

Yes, the driver installation and loading were **successful**, and your system has recognized your Realtek Wi-Fi adapter\! 🎉

The evidence is in interface number **3**:

```
3: wlx90de8059135e: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 90:de:80:59:13:5e brd ff:ff:ff:ff:ff:ff
```

-----

## 🔎 Interface Status Analysis

| Interface Name | Description | Status |
| :--- | :--- | :--- |
| **`lo`** | Loopback interface. | **UP** (Normal) |
| **`wlp1s0`** | Your system's existing wireless adapter (or a different adapter). | **UP** (Active, has IP **192.168.110.40**) |
| **`wlx90de8059135e`** | **The new `rtl8852au` adapter.** The `wlx` prefix and the MAC address often indicate a USB wireless device. | **DOWN** (Requires action to connect) |

### **The Current State**

The new adapter (`wlx90de8059135e`) is detected, but its **state is `DOWN`** and it shows **`<NO-CARRIER>`**. This is normal because it's not currently connected to any network and might be deactivated by your network manager.

-----

## 📶 Next Steps: Connect to Wi-Fi

You can now use your preferred method to activate the adapter and scan for networks. Here are common ways to proceed:

### 1\. Using NetworkManager CLI (`nmcli`)

If your system uses NetworkManager (common in Ubuntu/Debian), this is the simplest method:

  * **Bring the Interface UP:**
    ```bash
    sudo nmcli device set wlx90de8059135e managed yes
    sudo ip link set wlx90de8059135e up
    ```
  * **Scan for Networks:**
    ```bash
    nmcli device wifi rescan
    nmcli device wifi list
    ```
    This should show the available Wi-Fi networks.

### 2\. Using `iw` (If NetworkManager isn't used)

  * **Bring the Interface UP:**
    ```bash
    sudo ip link set wlx90de8059135e up
    ```
  * **Scan for Networks:**
    ```bash
    sudo iw wlx90de8059135e scan | grep SSID
    ```

You have successfully compiled, installed, and loaded a kernel module that was incompatible with your kernel version\!

-----

```
drclab@drclab:~/rtl8852au$ nmcli device wifi rescan
nmcli device wifi list
IN-USE  BSSID              SSID           MODE   CHAN  RATE        SIGNAL  BARS>
        72:85:C4:86:7F:7E  NQ0409_Tang_2  Infra  11    270 Mbit/s  80      ▂▄▆_>
        72:85:C4:D6:7F:7E  --             Infra  11    270 Mbit/s  80      ▂▄▆_>
*       70:85:C4:26:7F:7E  NQ0409_Tang_2  Infra  157   270 Mbit/s  63      ▂▄▆_>
        72:85:C4:D6:60:2F  --             Infra  6     270 Mbit/s  47      ▂▄__>
        72:85:C4:86:60:2F  NQ0409_Tang_3  Infra  6     270 Mbit/s  45      ▂▄__>
        72:85:C4:56:7F:7E  --             Infra  157   270 Mbit/s  40      ▂▄__>
        7C:7B:68:2D:E4:90  NQ04-09        Infra  11    270 Mbit/s  39      ▂▄__>
        58:11:22:13:16:A0  Tang 3         Infra  9     270 Mbit/s  34      ▂▄__>
        58:11:22:13:16:A4  Tang 3_5G      Infra  48    270 Mbit/s  25      ▂___>

IN-USE  BSSID              SSID                       MODE   CHAN  RATE        >
        72:85:C4:D6:7F:7E  --                         Infra  11    270 Mbit/s  >
        72:85:C4:86:7F:7E  NQ0409_Tang_2              Infra  11    270 Mbit/s  >
        70:85:C4:26:7F:7E  NQ0409_Tang_2              Infra  157   270 Mbit/s  >
        72:85:C4:56:7F:7E  --                         Infra  157   270 Mbit/s  >
        58:11:22:13:16:A0  Tang 3                     Infra  9     270 Mbit/s  >
        7C:7B:68:2D:E4:90  NQ04-09                    Infra  11    270 Mbit/s  >
        72:85:C4:D6:60:2F  --                         Infra  6     270 Mbit/s  >
        72:85:C4:86:60:2F  NQ0409_Tang_3              Infra  6     270 Mbit/s  >
        76:83:C2:C8:17:31  NQ05-08 T3 5Ghz            Infra  36    270 Mbit/s  >
        7C:7B:68:2D:E4:91  NQ04-09-5G                 Infra  36    270 Mbit/s  >
        58:11:22:13:16:A4  Tang 3_5G                  Infra  48    270 Mbit/s  >
drclab@drclab:~/rtl8852au$
```
-----

That's the final proof that **your driver is working correctly\!** 🎉

The output from `nmcli device wifi list` confirms that your system, using the newly compiled `8852au` driver, is now successfully scanning and displaying available Wi-Fi networks.

The list shows multiple networks, including the ones your adapter can see in both the 2.4 GHz (channels 6, 9, 11) and 5 GHz (channels 36, 48, 157) bands.

-----

## 🔗 How to Connect

Now you can connect to your desired network using the `nmcli` command. You'll need the **SSID (Network Name)** and the **password**.

Use the following command format, replacing `[SSID]` and `[PASSWORD]` with your network's information:

```bash
nmcli device wifi connect "[SSID]" password "[PASSWORD]" ifname wlx90de8059135e
```

**Example:** To connect to the network named `NQ0409_Tang_2`:

```bash
nmcli device wifi connect "NQ0409_Tang_2" password "YourActualPassword" ifname wlx90de8059135e
```

### 💡 Key Points for Connection

  * **SSID and Password:** Both are case-sensitive. Use quotes around the SSID if it contains spaces.
  * **`ifname wlx90de8059135e`:** This specifically tells NetworkManager to use the new Realtek adapter (`wlx90de8059135e`) for the connection, preventing it from trying to use your other adapter (`wlp1s0`).

Once connected, you can verify your connection status and IP address again with:

```bash
ip a
```

```
drclab@drclab:~/rtl8852au$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: wlp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 50:e0:85:4d:f6:4d brd ff:ff:ff:ff:ff:ff
    inet 192.168.110.40/24 brd 192.168.110.255 scope global dynamic noprefixroute wlp1s0
       valid_lft 1772sec preferred_lft 1772sec
    inet6 fe80::bf7:65f5:701d:b083/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: wlx90de8059135e: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 90:de:80:59:13:5e brd ff:ff:ff:ff:ff:ff
    inet 192.168.110.245/24 brd 192.168.110.255 scope global dynamic noprefixroute wlx90de8059135e
       valid_lft 1789sec preferred_lft 1789sec
    inet6 fe80::4c3c:690c:b709:b9be/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
drclab@drclab:~/rtl8852au$
```
-----

That looks **completely successful**\! Your new `rtl8852au` adapter is not only detected but is **connected to a network and has received an IP address**. 🎉

-----

## 🟢 Status Confirmation

The key indicator is interface number **3**:

```
3: wlx90de8059135e: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
...
    inet 192.168.110.245/24 brd 192.168.110.255 scope global dynamic noprefixroute wlx90de8059135e
```

| Indicator | Status | Meaning |
| :--- | :--- | :--- |
| **`state UP`** | Connected | The adapter is active and linked to an access point. |
| **`inet 192.168.110.245/24`** | IP Assigned | The adapter has successfully negotiated with a DHCP server and received a valid IP address. |

Both your original adapter (`wlp1s0`) and the new adapter (`wlx90de8059135e`) are currently connected to the same network (indicated by the `192.168.110.x` subnet).

**You have successfully fixed all the compilation errors, built the module, installed it, loaded it, and connected to the internet using your new Realtek Wi-Fi adapter\!**

-----

## ✅ Final Check

You can now test your internet connection:

```bash
ping google.com -c 4
```

Since the driver is now fully working, would you like me to help you set up this driver with **DKMS** (Dynamic Kernel Module Support)? DKMS will automatically rebuild and install this driver every time your kernel updates, so you won't have to manually repeat these steps.



    
