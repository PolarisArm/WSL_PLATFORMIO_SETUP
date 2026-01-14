# WSL PlatformIO Setup Guide  
*I had a very hard time installing all the support for PlatformIO + VS Code + USB. For future me — this is the step-by-step.*

---

## ✅ Steps

1. **Install WSL**  
   Open PowerShell as Administrator and run:
   ```powershell
   wsl --install
   ```
   This installs WSL2 and Ubuntu by default.

2. **Set up Linux user**  
   After installation, reboot Windows.  
   On first boot, WSL will prompt you to create a **username and password** for your Linux user.

3. **Launch WSL**  
   After reboot, open WSL via:
   - Terminal (`wsl`)
   - Start Menu → **Ubuntu**

4. **Install required tools**
   - **On Windows**: Install [`usbipd-win`](https://github.com/dorssel/usbipd-win) (required for USB passthrough to WSL2):
     ```powershell
     winget install Microsoft.USBIPD
     ```
   - **In WSL (Ubuntu)**: Install USB utilities:
     ```bash
     sudo apt update && sudo apt install usbutils
     ```

5. **Use `usbipd` to manage USB devices**  
   All `usbipd` commands must be run in **PowerShell as Administrator on Windows** (not inside WSL).

   ### 🔹 List connected USB devices
   ```powershell
   usbipd list
   ```
   Example output:
   ```
   Connected:
   BUSID  VID:PID    DEVICE                                                        STATE
   1-10   09da:2268  USB Input Device                                              Not shared
   1-12   10c4:ea60  Silicon Labs CP210x USB to UART Bridge (COM19)                Not shared
   1-13   8087:0aa7  Intel(R) Wireless Bluetooth(R)                                Not shared
   ```

   ### 🔹 Bind the device (one-time setup)
   Binding installs a generic USB driver so `usbipd` can take control.
   ```powershell
   usbipd bind --busid 1-12
   ```

   ### 🔹 Attach to WSL
   Makes the device available inside WSL2:
   ```powershell
   usbipd wsl attach --busid 1-12
   ```
   Output:
   ```
   usbipd: info: Using WSL distribution 'Ubuntu' to attach; the device will be available in all WSL 2 distributions.
   usbipd: info: Loading vhci_hcd module.
   usbipd: info: Detected networking mode 'nat'.
   usbipd: info: Using IP address 172.19.64.1 to reach the host.
   ```

   After attaching, `usbipd list` shows:
   ```
   1-12   10c4:ea60  Silicon Labs CP210x USB to UART Bridge (COM19)   Attached
   ```

   > 💡 The device will appear in WSL as `/dev/ttyACM0` (or similar). Check with:
   > ```bash
   > ls -l /dev/ttyACM*
   > dmesg | tail
   > ```

   ### 🔹 Detach from WSL (release back to Windows)
   ```powershell
   usbipd wsl detach --busid 1-12
   ```
   State becomes: `Shared`

   ### 🔹 Unbind (optional – restores original Windows driver)
   ```powershell
   usbipd unbind --busid 1-12
   ```
   State returns to: `Not shared`
    
   ## Problem PlatformIO Unable to Locate Project (WSL2 Path Issue)
    
   After creating the project, PlatformIO **failed to recognize it**, even though the folder was created.
    
   ### 🔍 Root Cause
    - PlatformIO runs inside **WSL2 (Linux)** but sometimes uses **Windows-style backslashes (`\`)** in internal paths.
    - This creates malformed directory names like:  
    `~/PlatformIO/Projects/Projects\test`  
   which Linux cannot interpret correctly.
    
   ### 🛠️ Fix: Patch PlatformIO Home (Temporary Workaround)
    
    1. In WSL2, navigate to:
     ```bash
     cd ~/.platformio/packages/contrib-piohome/
     ```
    2. Find the minified JavaScript file (e.g., `main.XXXXX.min.js`).
    3. Open it in a text editor (e.g., `nano` or `code`):
     ```bash
     nano main.*.min.js
     ```
    4. Press **Ctrl+F** and search for:
     ```
     "\\":"/"
     ```
    5. Replace it with:
     ```
     "/":"/"
     ```
    6. Save and close the file.
    
   > 💡 This forces PlatformIO to use **forward slashes** consistently in Linux paths.
    
    7. **Reload VS Code** (Ctrl+Shift+P → “Developer: Reload Window”).
    8. Recreate your project — it should now be recognized properly.
    
    
