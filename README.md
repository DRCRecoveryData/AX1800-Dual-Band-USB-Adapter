# 🛠️ Realtek RTL8852AU (0bda:8832) WiFi Driver with Modern Kernel Patches

This repository contains the `rtl8852au` driver source, patched to successfully **compile and install on recent Linux kernel versions (6.0+)**.

The provided script handles all necessary steps: installing prerequisites, applying the kernel patches, and setting up the driver for permanent use with **DKMS (Dynamic Kernel Module Support)**.

## 🚀 One-Step Installation (Recommended)

This single command block will handle all prerequisites, patching, compilation, and permanent installation via DKMS.

---

### **1. Clone the Repository and Execute**

If you are starting on a fresh system, copy and paste the entire block below into your terminal and run it.

```bash
# Install prerequisites (build-essentials, dkms, git)
sudo apt update
sudo apt install -y build-essential dkms git

# Clone the repository and navigate into the folder
git clone [https://github.com/lwfinger/rtl8852au.git](https://github.com/lwfinger/rtl8852au.git)
cd rtl8852au

#!/bin/bash

echo "Starting full patch and DKMS installation for rtl8852au..."

# --- FIX 1: VFS MODULE_IMPORT_NS ERROR ---
echo "Applying Fix 1: Commenting out MODULE_IMPORT_NS (for older kernel support)..."
sed -i 's/^\(\s*MODULE_IMPORT_NS.*\)$/\/*\1 *\//' os_dep/osdep_service_linux.c

# --- FIX 2 & 3: CFG80211 SIGNATURE AND POINTER FIXES (Linux 6.0+) ---
echo "Applying Fix 2 & 3: Updating cfg80211 function definitions and setting incompatible pointers to NULL..."

# Patch Function Definitions
sed -i 's/cfg80211_rtw_get_txpower(struct wiphy *wiphy, struct wireless_dev *wdev, int *tx_power)/cfg80211_rtw_get_txpower(struct wiphy *wiphy, struct wireless_dev *wdev, unsigned int type, int *tx_power)/' os_dep/linux/ioctl_cfg80211.c
sed -i 's/cfg80211_rtw_set_monitor_channel(struct wiphy *wiphy, struct cfg80211_chan_def *chan_def)/cfg80211_rtw_set_monitor_channel(struct wiphy *wiphy, struct net_device *dev, struct cfg80211_chan_def *chan_def)/' os_dep/linux/ioctl_cfg80211.c

# Patch Structure Assignments (Set incompatible functions to NULL)
sed -i 's/\.get_tx_power = cfg80211_rtw_get_txpower,/\.get_tx_power = NULL,/' os_dep/linux/ioctl_cfg80211.c
sed -i 's/\.set_monitor_channel = cfg80211_rtw_set_monitor_channel,/\.set_monitor_channel = NULL,/' os_dep/linux/ioctl_cfg80211.c

# --- SETUP DKMS AND INSTALL ---
echo "--- Setting up DKMS for permanent installation ---"
sudo dkms add .
sudo dkms install rtl8852au/1.15.0.1

if [ $? -eq 0 ]; then
    echo "✅ DKMS installation successful. The driver will auto-rebuild on kernel updates."
    echo "Loading the module..."
    sudo modprobe 8852au
    
    echo "Final driver status check:"
    dkms status
    echo ""
    echo "Your new wireless interface (likely wlx...) should now be available."
else
    echo "❌ DKMS installation failed. Please check the output for errors."
    exit 1
fi
````

-----

## ✅ Verification & Next Steps

After running the script, the driver module should be loaded, and your Realtek adapter should be available as a new network interface (e.g., `wlan0`, or a persistent name like `wlx90de8059135e`).

### **1. Verify Module Loading**

Confirm the `8852au` module is loaded into the kernel:

```bash
lsmod | grep 8852au
```

*Expected Output: A line showing the `8852au` module.*

### **2. Check for the Wireless Interface**

Check for a new network interface, often named starting with `wlx...`:

```bash
ip a
```

*Expected Output: A new interface (e.g., `wlx90de8059135e`) with status `UP` or `DOWN`.*

### **3. Connect to Wi-Fi**

If your system uses **NetworkManager** (common in Ubuntu, Debian, Fedora):

  * **Rescan and List Networks:**

    ```bash
    nmcli device wifi rescan
    nmcli device wifi list
    ```

  * **Connect to your desired network** (replace the values and `ifname` with your actual interface name if different from `wlx90de8059135e`):

    ```bash
    nmcli device wifi connect "YourNetworkSSID" password "YourActualPassword" ifname wlx90de8059135e
    ```

  * **Final Connection Status Check:**

    ```bash
    ip a
    ```

    *Look for the new interface to have an `inet` (IP) address assigned.*

-----

## 🗑️ Uninstalling the Driver

If you need to remove the driver and stop DKMS from managing it:

```bash
# 1. Uninstall the module from DKMS
sudo dkms remove rtl8852au/1.15.0.1 --all

# 2. Unload the module from the kernel (if loaded)
sudo modprobe -r 8852au

# 3. Clean the build directory (optional)
cd rtl8852au
make clean
```
