This guide outlines the complete technical workflow for converting a raw disk image into a VMware-compatible OVA using only Command Line Interface (CLI) tools. This process was specifically tested using **CirrOS 0.6.3** on a **Gentoo Linux** workstation and a **VMware ESXi 7.x/8.x** host.

## ---

**1\. Preparation (Local Machine)**

Before moving to the server, convert your raw image into a VMware-compatible virtual disk (VMDK).

* **Tool:** qemu-img (available in app-emulation/qemu on Gentoo).  
* **Command:**  
  Bash  
  qemu-img convert \-f raw \-O vmdk \-o adapter\_type=lsilogic,subformat=monolithicSparse cirros-rootfs.img cirros.vmdk

* **Upload:** Use scp to move cirros.vmdk to your ESXi datastore.

## ---

**2\. Disk Native Conversion (ESXi CLI)**

ESXi requires a specific descriptor format that qemu-img does not always provide perfectly. You must "clone" the disk into a native thin-provisioned format.

1. **Navigate to your datastore:**  
   Bash  
   cd /vmfs/volumes/datastore1/  
   mkdir cirros\_build && cd cirros\_build

2. **Clone the disk:**  
   Bash  
   vmkfstools \-i /path/to/uploaded/cirros.vmdk \-d thin cirros-final.vmdk

   *Note: If this fails, ensure you are working on a VMFS partition and not a VFAT boot partition.*

## ---

**3\. VM Registration (ESXi CLI)**

Create the "hardware" wrapper for your disk.

1. **Create the configuration file (cirros.vmx):**  
   Plaintext  
   config.version \= "8"  
   virtualHW.version \= "13"  
   memsize \= "256"  
   numvcpus \= "1"  
   guestOS \= "other3xlinux-64"  
   ethernet0.present \= "TRUE"  
   ethernet0.virtualDev \= "e1000"  
   ethernet0.networkName \= "VM Network"  
   ethernet0.addressType \= "generated"  
   scsi0.present \= "TRUE"  
   scsi0.virtualDev \= "lsilogic"  
   scsi0:0.present \= "TRUE"  
   scsi0:0.fileName \= "cirros-final.vmdk"

2. **Register the VM with the hypervisor:**  
   Bash  
   vim-cmd solo/registervm /vmfs/volumes/datastore1/cirros\_build/cirros.vmx

3. **Power On (Verify):**  
   Bash  
   \# Get ID, then power on  
   vim-cmd vmsvc/getallvms  
   vim-cmd vmsvc/power.on \<VM\_ID\>

## ---

**4\. OVA Export (Local Machine)**

The final step uses the VMware OVF Tool to pull the VM from the host and package it as a compressed .ova file.

* **Tool:** ovftool (Standalone binary from VMware/Broadcom).  
* **Command:**  
  Bash  
  ./ovftool \--noSSLVerify \\  
    vi://root@\<ESXI\_IP\>/CirrOS-Export-VM \\  
    \~/cirros-0.6.3-build.ova

## ---

**5\. Result Verification**

A successful export results in a single .ova file. You can inspect its contents using tar:

Bash

tar \-tvf \~/cirros-0.6.3-build.ova

**Expected files:**

* .ovf: XML hardware descriptor.  
* .mf: SHA checksum manifest.  
* .vmdk: Stream-optimized (compressed) disk image.

### ---

**Troubleshooting Summary**

* **Power On Failure:** Check vmware.log. If it complains about GuestOS, ensure guestOS \= "other3xlinux-64" is set in the .vmx.  
* **Disk Errors:** Ensure you are using vmkfstools on the ESXi host to finalize the disk before registration.  
* **Space Issues:** Always work inside /vmfs/volumes/datastore\_name/ to avoid filling up the ESXi RAM-disk.

Would you like me to convert this into a Markdown file you can download or save directly to your Gentoo system?
