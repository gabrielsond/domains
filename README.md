# domains
A collection of (working/tested) domain/VM XML configuration files focused on integrated GPU dedicated passthrough ([GVT-D](https://github.com/intel/gvt-linux/wiki) "Legacy" mode on Intel 11700k).

## Background
The 11700k includes an integrated graphics device (`[8086:4c8a] 00:02.0 VGA compatible controller: Intel Corporation RocketLake-S GT1 [UHD Graphics 750]`) which supports GVT-D but has lacked clear examples/tutorials of how to passthrough the GPU to a VM. After many attempts and failures, here are domain config with GVT-D working using [OVMF](https://github.com/tianocore/tianocore.github.io/wiki/OVMF) with the GPU in apparent ["Legacy" mode](https://github.com/qemu/qemu/blob/master/docs/igd-assign.txt).

## Hardware Configuration
- Intel Core i7 11700k
- ASUS Prime Z590-A
- Trident Z Neo DDR4-3600 CL16-19-19-39 1.35V 16GB x 2
- ASUS DUAL RTX 3070 OC 8GB
- WD Black SN750 NVMe SSD 1TB (Unraid Cache)
- WD Black SN850 NVMe SSD 1TB (Resevered for VM Passthrough)
- Fresco FL1100 4x USB 3.0 PCI Express Card PCI-e USB3.0 Host Controller Adapter (Resevered for VM passthrough; bus='0x07' slot='0x00' function='0x0)
- Marvell 88SE9215 PCIe 2.0 x1 4-port SATA 6 Gb/s Controller

6+1 Unraid Array (7 x 4TB SATA drives):
- Parity: ST4000VN008
- 1 x WDC_WD40EFRX
- 1 x ST4000DM000
- 3 x ST4000DM004
- 1 x ST4000VN008

## Requirements:
- Host OS: Linux
  - Tested: Unraid 6.11.5 (Kernel: 5.19.17-Unraid)
    - *should* work with most modern kernels/distros
- Kernel modules blacklisted
  `modprobe.blacklist=i2c_i801,i2c_smbus,snd_hda_intel,snd_hda_codec_hdmi,i915,drm,drm_kms_helper,i2c_algo_bit`
  - Possibly excessive; more testing required
- EFI & VESA Framebuffers off
  `video=efifb:off,vesafb:off`
  - Possibly optional; more testing required
- (*Likely* optional) Both PCIe ACS overrides and allow VFIO unsafe interrupts
  `pcie_acs_override=downstream,multifunction vfio_iommu_type1.allow_unsafe_interrupts=1`
  - used for passing through associated audio device
  `[8086:43c8] 00:1f.3 Audio device: Intel Corporation Tiger Lake-H HD Audio Controller`
- Secondary GPU or device to log in to Guest OS/VM via VNC/RDP
- Integrated graphics physically connected to powered-on monitor on HDMI port (be patient, watch for changes, always have a remote login method available)
- Specific QEMU XML:
  - `<domain>` tag must be updated to add QEMU namespace (replace id as appropriate):
  ```
  <domain type='kvm' id='#' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  ```
  - override integrated passthrough settings (add before `</domain>`):
  ```
  <qemu:override>
    <qemu:device alias='hostdev0'>
      <qemu:frontend>
        <qemu:property name='x-igd-opregion' type='bool' value='true'/>
        <qemu:property name='driver' type='string' value='vfio-pci-nohotplug'/>
      </qemu:frontend>
    </qemu:device>
  </qemu:override>
  ```

- (Windows Guests/VMs) ROM file: [vbios_gvt_uefi.rom](https://web.archive.org/web/20201020144354/http://120.25.59.132:3000/vbios_gvt_uefi.rom) :grimacing:
  - Hopefully, there is a better option because [i915ovmf.rom](https://github.com/patmagauran/i915ovmfPkg) did not work in testing

## Configurations
[Windows 10 - 6 Core (isolated) passthrough - GVT-D passthrough - NVIDIA passthrough](w10_z590-a_11700k_6core_gvt-d_and_nvidia_passthrough.xml)

NOTES:
- This VM uses 6 isolated cores:
  - Unraid boots using cores 0 and 7 (and HT) so that leaves cores 1-6 (and HT) isolated for VM usage
  ```
  isolcpus=1-6,9-14 nohz_full=1-6,9-14
  ```
  - see `<vcpu>`, `<cputune>`, and `<cpu>` blocks for related configuration
- HDMI audio does not appear to work in Windows nor do any of the analog ports. The SPDIF shows as working but has not been tested.
- Installation process:
  - Set VNC as the only graphics device in XML,  save, review to ensure changes saved successfully, start VM
  - Install Windows, enable and test RDP login, and shutdown the VM
  - With RDP enabled, remove VNC graphics device and add integrated graphics device to XML, save XML, review to ensure changes saved successfully, start VM
  - Login with RDP and install Intel graphics driver (as many times as necessary until error goes away and monitor shows output; 2-3 times in my experience)
  - Ensure Device Manager shows all devices working properly
  - Install NVIDIA graphics drivers and shutdown VM
  - Add NVIDIA graphics device to XML, save, review to ensure changes saved successfully, start VM
  - Dual GPU passthrough should now work (4x4K tested: NVIDIA: DPx2 & Intel DPx1 HDMIx1)
- Adding TPM emulator device and changing loader to OVMF TPM will fulfill requirements for Windows 11 upgrade
  - Driver re-install via RDP may be required after upgrade
- `<qemu:commandline>` block is entirely optional but `<qemu:override>` block is critical for GVT-D
- ROM file is required for Intel device to enable physical display on Windows
  - `/mnt/cache/isos/` is not the best location for the ROM, this will be moved
- `<name>Windows 10 (EXPERIMENTAL CONFIG 2)</name>` will be updated in the near future
- see commit comments for details/history
