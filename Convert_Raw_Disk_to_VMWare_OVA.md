---

# CLI Guide: Creating a VMware OVA from a Raw Image

This document provides the end-to-end workflow for converting a **raw disk image** (like `cirros.img`) into a portable **OVA** using only CLI tools. 

## Phase 1: Local Preparation (Gentoo Workstation)
Convert the raw image into a VMware-compatible virtual disk (VMDK). 

1.  **Check the format:**

It's important to check the image format. If the format is not raw, but qcow2, an additional conversion is required.

```bash
qemu-img info cirros.img

image: original.img
file format: qcow2
virtual size: 112 MiB (117440512 bytes)
disk size: 49.4 MiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
    extended l2: false
Child node '/file':
    filename: cirros.img
    protocol type: file
    file length: 49.4 MiB (51838976 bytes)
    disk size: 49.4 MiB
```

The orginal.img above is qcow2 (file format: qcow2), this is not a raw image and the conversion to raw is required.

```bash
qemu-img convert -f qcow2 -O raw cirros.img cirros.raw
qemu-img info disk.raw
```

2.  **Convert Format:**
    ```bash
    qemu-img convert -f raw -O vmdk -o adapter_type=lsilogic,subformat=monolithicSparse cirros.raw cirros.vmdk
    ```
3.  **Upload to ESXi:** Use `scp` to move `cirros.vmdk` to your ESXi datastore (e.g., `/vmfs/volumes/datastore1/`).

---

## Phase 2: Native Disk Conversion (SSH to ESXi)
ESXi requires a specific descriptor format to recognize the disk. You must clone it into a native thin-provisioned format.

1.  **Navigate to Storage:**
    ```bash
    cd /vmfs/volumes/datastore1/
    mkdir cirros_build && cd cirros_build
    ```
2.  **Clone the Disk:**
    ```bash
    vmkfstools -i /upload path at ESXi/cirros.vmdk -d thin cirros-final.vmdk
    ```
    *Note: This creates two files: `cirros-final.vmdk` (descriptor) and `cirros-final-flat.vmdk` (data).*

---

## Phase 3: VM Configuration and Registration (ESXi SSH)
Create the "Hardware Wrapper" (VMX) for the disk.

1.  **Create `cirros.vmx`:**
    ```text
    config.version = "8"
    virtualHW.version = "13"
    memsize = "256"
    numvcpus = "1"
    guestOS = "other3xlinux-64"

    # To define the firmware as UEFI in a VMware .vmx file
    # firmware = "efi"
    
    # IMPORTANT: This name is used by ovftool in the locator
    displayName = "CirrOS-Export-VM"

    ethernet0.present = "TRUE"
    ethernet0.virtualDev = "e1000"
    ethernet0.networkName = "VM Network"
    ethernet0.addressType = "generated"

    scsi0.present = "TRUE"
    scsi0.virtualDev = "lsilogic"
    scsi0:0.present = "TRUE"
    scsi0:0.fileName = "cirros-final.vmdk"
    ```
2.  **Register the VM:**
    ```bash
    # Use the absolute path to the .vmx file
    vim-cmd solo/registervm /vmfs/volumes/datastore1/cirros_build/cirros.vmx
    ```

---

## Phase 4: Exporting to OVA (Gentoo Workstation)
Use the `ovftool` to connect to the ESXi host and pull the VM into a compressed package.

* **Syntax:** `vi://[user]:[pass]@[host]/[displayName]`
* **Command:**
    ```bash
    ./ovftool --noSSLVerify \
      vi://root@<ESXI_IP>/CirrOS-Export-VM \
      ~/cirros-0.6.3.ova
    ```

---

## Phase 5: Verification & Cleanup
1.  **Verify the OVA:** ```bash
    tar -tvf ~/cirros-0.6.3.ova
    ```
    (You should see the `.ovf`, `.mf`, and `.vmdk` files).
2.  **Cleanup ESXi:**
    ```bash
    # Get the VM ID first
    vim-cmd vmsvc/getallvms | grep CirrOS
    # Unregister and delete
    vim-cmd vmsvc/unregister <VM_ID>
    rm -rf /vmfs/volumes/datastore1/cirros_build
    ```

---

### Technical Summary Table
| Component | Value | Role |
| :--- | :--- | :--- |
| **Source Image** | `raw` | Bit-for-bit disk copy |
| **Guest OS ID** | `other3xlinux-64` | Compatibility string for modern ESXi |
| **Display Name** | `CirrOS-Export-VM` | The **key identifier** for the `ovftool` locator |
| **Disk Adapter** | `lsilogic` | Standard SCSI controller for Linux |
