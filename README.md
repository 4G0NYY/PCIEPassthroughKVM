# **Arch Linux: KVM GPU Passthrough + Hypervisor Obscuring**
### **Prerequisites:**
✅ **Hardware:**  
- **Primary GPU** (for host, e.g., Intel iGPU or another GPU).  
- **Secondary GPU** (RTX 3070 Ti) for passthrough.  
- **CPU** with **VT-d/AMD-Vi** (Intel) or **AMD-Vi/SVM** (AMD).  
- **Motherboard** with **IOMMU** enabled in BIOS.  
- **Two monitors** (or a way to switch inputs).  

✅ **Software:**  
- **Arch Linux** (fully updated).  
- **Windows 11 ISO** (for the VM).  
- **UEFI boot** (not legacy).  

---

## **Step 1: Enable IOMMU & Verify GPU Isolation**
### **1.1 Enable IOMMU in Kernel Parameters**
Edit `/etc/default/grub` and modify the `GRUB_CMDLINE_LINUX` line:  
```bash
GRUB_CMDLINE_LINUX="... intel_iommu=on iommu=pt vfio-pci.ids=10de:2487,10de:228b ..."
```
- Replace `10de:2487,10de:228b` with your **GPU & Audio Controller IDs** (find them with `lspci -nn | grep NVIDIA`).  
- For **AMD CPUs**, use `amd_iommu=on`.  

Update GRUB:  
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### **1.2 Verify IOMMU Groups**
Run:  
```bash
sudo dmesg | grep -i iommu
```
Check if IOMMU is enabled. Then:  
```bash
#!/bin/bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done
```
Ensure your **GPU and its audio controller** are in **separate groups**. If not, you may need the **ACS override patch**.

## ACS Override Patch

### **Why This Happens**  
Some motherboards **don’t isolate PCIe devices properly** due to cost-cutting or firmware limitations. This is common on **consumer-grade boards** (vs. server/workstation ones).  

### **Solutions**  
Here’s how to fix it:  

---

### **Option 1: ACS Override Patch (Most Effective)**  
**What it does**: Forces PCIe device isolation (even if the motherboard doesn’t support it natively).  
**⚠️ Warning**: This *can* reduce security slightly (potential DMA attacks), but for a home setup, it’s generally safe.  

#### **Steps:**  
1. **Install a kernel with ACS patch** (or patch it yourself):  
   - Pre-patched kernels (Arch):  
     ```bash
     yay -S linux-vfio
     ```
   - Or manually patch (advanced users).  

2. **Enable ACS override in kernel parameters**:  
   Edit `/etc/default/grub` again and add:  
   ```bash
   GRUB_CMDLINE_LINUX="... pcie_acs_override=downstream,multifunction ..."
   ```
   - `downstream`: Isolates devices below the root port.  
   - `multifunction`: Splits multi-function devices (like GPU + audio).  

3. **Update GRUB & reboot**:  
   ```bash
   sudo grub-mkconfig -o /boot/grub/grub.cfg
   sudo reboot
   ```

4. **Verify new IOMMU groups**:  
   Run the IOMMU group script again:  
   ```bash
   #!/bin/bash
   for d in /sys/kernel/iommu_groups/*/devices/*; do
       n=${d#*/iommu_groups/*}; n=${n%%/*}
       printf 'IOMMU Group %s ' "$n"
       lspci -nns "${d##*/}"
   done
   ```
   - Your GPU and audio should now be in **separate groups**.  

---

### **Option 2: Use a Different PCIe Slot**  
- Some motherboards **isolate slots better** (e.g., the first x16 slot shares lanes with the CPU, while others go through the chipset).  
- Try moving the GPU to another slot and check IOMMU groups again.  

---

### **Option 3: Pass the Entire Group (Last Resort)**  
If ACS patch doesn’t work and you can’t change slots:  
- Pass the **entire IOMMU group** to the VM (including unwanted devices).  
- Then **disable/unbind the unwanted devices** inside the VM.  
- Not ideal, but works in some cases.  

---

## **Step 2: Configure vfio-pci Early Binding**
### **2.1 Identify GPU & Audio Controller IDs**
```bash
lspci -nn | grep NVIDIA
```
Example output:  
```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA104 [GeForce RTX 3070 Ti] [10de:2487]  
01:00.1 Audio device [0403]: NVIDIA Corporation GA104 High Definition Audio Controller [10de:228b]  
```

### **2.2 Add vfio-pci to mkinitcpio**
Edit `/etc/mkinitcpio.conf` and add `vfio_pci`, `vfio`, `vfio_iommu_type1` to `MODULES`:  
```bash
MODULES=(vfio_pci vfio vfio_iommu_type1)
```
Then regenerate initramfs:  
```bash
sudo mkinitcpio -P
```

### **2.3 Blacklist NVIDIA Drivers (Optional)**
If the host shouldn’t use the GPU:  
```bash
echo "blacklist nouveau" | sudo tee /etc/modprobe.d/blacklist-nvidia.conf
echo "blacklist nvidia" | sudo tee -a /etc/modprobe.d/blacklist-nvidia.conf
```

---

## **Step 3: Install & Configure KVM/QEMU**
### **3.1 Install Required Packages**
```bash
sudo pacman -S qemu libvirt virt-manager dnsmasq edk2-ovmf
```

### **3.2 Enable & Start Services**
```bash
sudo systemctl enable --now libvirtd
```

### **3.3 Configure Libvirt**
Edit `/etc/libvirt/qemu.conf`:  
```bash
user = "your_username"
group = "your_username"
security_driver = "none"
```
Uncomment & set:  
```bash
nvram = [
   "/usr/share/edk2-ovmf/x64/OVMF_CODE.fd:/usr/share/edk2-ovmf/x64/OVMF_VARS.fd"
]
```

Restart libvirt:  
```bash
sudo systemctl restart libvirtd
```

---

## **Step 4: Create the VM (virt-manager)**
### **4.1 Create a New VM**
1. Open `virt-manager`.  
2. Click **"Create a new virtual machine"**.  
3. Select **"Local install media"** (Windows 11 ISO).  
4. Choose **"Customize before install"**.  

### **4.2 Configure VM Settings**
- **Chipset**: `Q35`  
- **Firmware**: `UEFI` (OVMF)  
- **CPU**: **"Copy host CPU"** + enable **"topology"** (match cores).  
- **Enable Nested Virtualization** (for WSL2/Hyper-V inside VM):  
  ```bash
  sudo modprobe -r kvm_intel  # If Intel
  sudo modprobe kvm_intel nested=1
  echo "options kvm_intel nested=1" | sudo tee /etc/modprobe.d/kvm.conf
  ```

### **4.3 Add GPU Passthrough**
1. **Add Hardware** → **PCI Host Device** → Select **GPU & Audio Controller**.  
2. **Enable "All features"** for hypervisor hiding.  

### **4.4 Hide the Hypervisor (Critical for NVIDIA)**
Edit the VM’s XML (`virsh edit vmname`):  
```xml
<features>
  <hyperv>
    <vendor_id state='on' value='randomid'/>
    <kvm>
      <hidden state='on'/>
    </kvm>
  </hyperv>
  <ioapic driver='kvm'/>
</features>
<cpu mode='host-passthrough' check='none'/>
```
This prevents NVIDIA drivers from detecting a VM.

---

## **Step 5: Final Steps**
### **5.1 Install Windows 11**
- Boot the VM, install Windows.  
- Install **NVIDIA drivers** inside VM.  

### **5.2 (Optional) Looking Glass for Low-Latency Display**
```bash
yay -S looking-glass
```
Configure in VM:  
- Add `-device ivshmem-plain,memdev=looking-glass` to QEMU args.  

### **5.3 (Optional) Auto-Start VM**
```bash
sudo virsh autostart vmname
```

---

# **Troubleshooting**
- **Error 43 (NVIDIA)**: Ensure hypervisor hiding is enabled.  
- **IOMMU Group Issues**: Use **ACS override patch** if needed.  
- **Audio Issues**: Pass through HDMI audio controller separately.  

---

### **Final Result:**  
✅ **GPU Passthrough** (RTX 3070 Ti fully utilized).  
✅ **Nested Virtualization** (for WSL2/Hyper-V inside VM).  
✅ **Undetectable VM** (games/DRM work).  