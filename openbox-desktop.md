# Configure a minimal Openbox desktop for Gentoo

## 1. Install Simple Desktop Display Manager (SDDM)
```Bash
sudo emerge -av x11-misc/sddm
```
### 1.1 When using openrc

OpenRC, unlike systemd, doesn't have a built-in session and seat manager. This functionality is required by modern display managers like SDDM. The standard solution in Gentoo is to use elogind. The PAM error you're seeing is a classic symptom of elogind not being installed, not running, or not being correctly integrated.

#### **Solution Steps:**

1. **Install elogind:** If you haven't already, install it.  
   ```Bash  
   sudo emerge \--ask \--verbose elogind
   ```

2. **Enable the elogind USE Flag:** Many packages need to be compiled with support for elogind to integrate with it correctly. Ensure the elogind USE flag is set globally in your /etc/portage/make.conf.  
   Ini, TOML  
   # Example /etc/portage/make.conf  

   USE\="... elogind \-systemd ..."

   After adding it, you'll need to update your system so the changes are applied to all relevant packages.  
   ```Bash  
   sudo emerge -avuDN @world
   ```

4. **Ensure the Service is Running:** elogind needs to be running as a service.  
   * Add it to the default runlevel to start on boot:  
     ```Bash  
     sudo rc-update add elogind default
     ```

   * Start it now:  
     ```Bash  
     sudo rc-service elogind start
     ```

   * You can check its status with:  
     ```Bash  
     rc-status
     ```
---
