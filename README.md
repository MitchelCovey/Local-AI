Proxmox GPU Passthrough Guide

Overview
This guide provides a step-by-step approach to setting up an Ubuntu Server VM with GPU passthrough in Proxmox and installing Ollama, CUDA Toolkit, Docker, and Stable Diffusion.

Prerequisites
Ensure you have the following before starting:
- A system that supports IOMMU for PCIe passthrough (Intel VT-d or AMD-Vi)
- Proxmox VE installed and running
- NVIDIA drivers installed on your Ubuntu Server VM (if using an NVIDIA GPU)
- Ubuntu Server ISO ready
- Ollama installation files prepared

---

1. Create an Ubuntu Server VM with GPU Passthrough

1.1 Create the VM
1. Log in to the Proxmox Web Interface: `https://<your-proxmox-ip>:8006`
2. Click **Create VM** in the Proxmox dashboard.
3. Configure VM settings:
   - **VM ID & Name:** Choose an appropriate name (e.g., "Ubuntu-Server").
   - **OS:** Select *Linux* and *Ubuntu* as the version.
   - **Machine Type:** Set to **Q35** for PCIe passthrough support.
   - **CPU Type:** Set to **host** for full hardware feature access.
   - **Memory:** Allocate at least 4GB RAM (more for AI workloads).
   - **Hard Disk:** Use **SCSI or VirtIO** for better performance.
   - **Network:** Choose **VirtIO** for optimal network performance.
   - **CD/DVD:** Select the Ubuntu Server ISO file.

1.2 Configure PCIe Passthrough

 Enable IOMMU and VFIO on the Proxmox Host
1. Enable IOMMU in BIOS/UEFI:
   - Enable **VT-d (Intel)** or **AMD-Vi (AMD)** in your BIOS settings.
2. Configure IOMMU in Proxmox:
   ```bash
   sudo nano /etc/default/grub
   ```
   - For Intel, modify: `GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"`
   - For AMD, modify: `GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on"`
   - Save and exit, then run:
   ```bash
   sudo update-grub && sudo reboot
   ```
3. Verify IOMMU is enabled:
   ```bash
   dmesg | grep -e DMAR -e IOMMU
   ```

 Blacklist Default GPU Drivers
```bash
sudo nano /etc/modprobe.d/blacklist.conf
```
Add:
```bash
blacklist nouveau
blacklist radeon
blacklist amdgpu
```
Save and reboot:
```bash
sudo reboot
```

#### Load VFIO Modules
```bash
sudo nano /etc/modules
```
Add:
```bash
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
Save and reboot.

Bind GPU to VFIO Driver
1. Identify GPU PCI ID:
   ```bash
   lspci | grep -i nvidia
   ```
2. Create VFIO configuration:
   ```bash
   sudo nano /etc/modprobe.d/vfio.conf
   ```
   Add:
   ```bash
   options vfio-pci ids=<your-gpu-pci-id>,<your-audio-pci-id>
   ```
3. Update initramfs and reboot:
   ```bash
   sudo update-initramfs -u && sudo reboot
   ```

1.3 Configure VM for GPU Passthrough
1. In the Proxmox Web Interface, go to **Hardware > Add > PCI Device**.
2. Select your GPU and enable **Primary GPU** (if needed).
3. In **Options**, set CPU type to **host**.
4. Enable **UEFI Boot** for better GPU support.
5. Save and start the VM.

---

2. Install CUDA Toolkit
1. Install CUDA:
   ```bash
   sudo apt install nvidia-cuda-toolkit
   ```
2. Verify installation:
   ```bash
   nvcc --version
   ```
3. Monitor GPU usage:
   ```bash
   watch -n 0.5 nvidia-smi
   ```

---

3. Install Ollama
1. Download and install Ollama:
   ```bash
   curl -fsSL https://ollama.com/download | sh
   ```
2. Add a model:
   ```bash
   ollama pull llama2
   ```

---

4. Install Docker
1. Install dependencies:
   ```bash
   sudo apt-get update
   sudo apt-get install ca-certificates curl
   ```
2. Add Docker GPG key and repository:
   ```bash
   sudo install -m 0755 -d /etc/apt/keyrings
   sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
   sudo chmod a+r /etc/apt/keyrings/docker.asc
   ```
3. Install Docker:
   ```bash
   sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

---

5. Run Open WebUI with Docker
```bash
docker run -d --network=host -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

---

6. Install Stable Diffusion
Install Pyenv
```bash
sudo apt install -y make build-essential libssl-dev zlib1g-dev \
  libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev \
  libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev git
```
```bash
curl https://pyenv.run | bash
```
```bash
pyenv install 3.10
pyenv global 3.10
```

Install Stable Diffusion
```bash
wget -q https://raw.githubusercontent.com/AUTOMATIC1111/stable-diffusion-webui/master/webui.sh
chmod +x webui.sh
./webui.sh --listen --api
```

---

Conclusion
Following this guide, you can successfully set up GPU passthrough in Proxmox, install CUDA, Ollama, Docker, and Stable Diffusion, and efficiently run AI workloads on your VM. Happy computing!

