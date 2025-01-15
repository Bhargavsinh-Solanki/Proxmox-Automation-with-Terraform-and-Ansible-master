# PCIe Passthrough

## Determining PCI Card Address

```bash
lspci
```

## Configuration

### IOMMU

1. **Enable IOMMU in the Kernel Command Line:**

    ```bash
    sudo nano /etc/default/grub
    
    # Edit the following in the grub file
    intel_iommu=on iommu=pt
    
    # Save and exit the file
    ```

2. **Enable Required Kernel Modules:**

    ```bash
    sudo nano /etc/modules
    
    # Add the following to the modules file
    vfio
    vfio_iommu_type1
    vfio_pci
    ```

3. **Refresh Initramfs:**

    After modifying modules, refresh the initramfs using:

    ```bash
    sudo update-initramfs -u -k all
    ```

4. **Verify Loaded Modules:**

    Use the following command to verify if the modules are loaded:

    ```bash
    lsmod | grep vfio
    ```

    The output should include the modules listed above.

## Verifying IOMMU Parameters

### Verify IOMMU is Enabled

1. Reboot the system and run:

    ```bash
    dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
    ```

    You should see a line like "DMAR: IOMMU enabled." If no output appears, there is an issue.

2. **Check Interrupt Remapping Support:**

    ```bash
    dmesg | grep 'remapping'
    ```

    If you see lines such as:
    
    - `AMD-Vi: Interrupt remapping enabled`
    - `DMAR-IR: Enabled IRQ remapping in x2apic mode`

    Then remapping is supported.

## Host Configuration

If Proxmox VE does not automatically make the PCI(e) device unavailable for the host, you can:

1. **Pass Device IDs to VFIO Modules:**

    Add the following to a `.conf` file in `/etc/modprobe.d/`:

    ```bash
    options vfio-pci ids=1234:5678,4321:8765
    ```

    Replace `1234:5678` and `4321:8765` with the vendor and device IDs obtained by:

    ```bash
    lspci -nn
    ```

2. **Blacklist Host Drivers:**

    To ensure the device is free for passthrough:

    ```bash
    lspci -k
    ```

    For example, running:

    ```bash
    lspci -k | grep -A 3 "VGA"
    ```

    Might output:

    ```
    01:00.0 VGA compatible controller: NVIDIA Corporation GP108 [GeForce GT 1030] (rev a1)
        Subsystem: Micro-Star International Co., Ltd. [MSI] GP108 [GeForce GT 1030]
        Kernel driver in use: <some-module>
        Kernel modules: <some-module>
    ```

    Blacklist the drivers by creating a `.conf` file in `/etc/modprobe.d/`:

    ```bash
    echo "blacklist <some-module>" >> /etc/modprobe.d/blacklist.conf
    ```

3. **Update Initramfs and Reboot:**

    Refresh the initramfs and reboot the system:

    ```bash
    sudo update-initramfs -u -k all
    sudo reboot
    ```

## Verify Configuration

To check if the configuration was successful, run:

```bash
lspci -nnk
```

Check the device entry. If it shows:

```bash
Kernel driver in use: vfio-pci
```

or the *in use* line is missing, the device is ready for passthrough.

## BIOS Settings

Ensure **Vt-d** is enabled in the BIOS settings. This step is critical and often overlooked.
