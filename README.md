# Ray Tracing on Intel Arc GPUs - Linux

![Intel Arc logo](https://www.intel.in/content/dam/www/central-libraries/us/en/images/2022-09/arc-a-series-desktops-rwd.png.rendition.intel.web.1920.1080.png)
Image Source : [Intel](https://www.intel.in/content/www/in/en/products/details/discrete-gpus/arc.html)

## Mesa 3D Graphics Library

The Mesa project began as an open-source implementation of the OpenGL specification - a system for rendering interactive 3D graphics.

Over the years the project has grown to implement more graphics APIs, including OpenGL ES, OpenCL, OpenMAX, VDPAU, VA-API, Vulkan and EGL.

A variety of device drivers allows the Mesa libraries to be used in many different environments ranging from software emulation to complete hardware acceleration for modern GPUs.

Mesa ties into several other open-source projects: the Direct Rendering Infrastructure, X.org, and Wayland to provide OpenGL support on Linux, FreeBSD, and other operating systems.

Source : [Mesa 3D](https://docs.mesa3d.org/index.html)


## Table of Contents
- [Prerequisites](#prerequisites)
- [Configuration](#configs)
- [Downloads](#downloads)
- [Miscellaneous](#miscellaneous)

# Prerequisites

Check for Resizable BAR (ReBAR) support.
```bash
# check for ReBAR support
lspci -v |grep -A8 VGA

# if output resembles to this
03:00.0 VGA compatible controller: Intel Corporation Device 56a0 (rev 08) (prog-if 00 [VGA controller])
        Subsystem: Intel Corporation Device 4905
        Flags: bus master, fast devsel, latency 0, IRQ 182
        Memory at 84000000 (64-bit, non-prefetchable) [size=16M]
        Memory at 4000000000 (64-bit, prefetchable) [size=16G] # <- THIS
        Expansion ROM at 85000000 [disabled] [size=2M]
        Capabilities: <access denied>
        Kernel driver in use: i915
        Kernel modules: i915

# THIS should match the size of your GPU, if not enable ReBAR via BIOS.
```

# Configuration

| Specifications | Detail                                                  |
| ------------------- | ------------------------------------------- |
| Operating System    | Ubuntu 22.04 LTS                            |
| Kernel Version      | Linux Kernel 6.1-rc5                        |
| CPU                 | Intel i7-1165G7 (8 cores)                   |
| Integrated Graphics | Intel UHD Graphics 630                      |
| Discrete Graphics   | Intel Arc A770 16GB                         |

# Downloads

1. Upgrade to Linux 6.1 Kernel from [Ubuntu archives](https://kernel.ubuntu.com/~kernel-ppa/mainline/). We will manually download the necessary files using wget and install them with dpkg but before that we should make sure that our system is updated.

```bash
sudo apt-get update && sudo apt-get full-upgrade -y # update the system
sudo apt install wget # if not present
cd ~/Downloads 
mkdir kernel-6.1 && cd kernel-6.1 # kernel directory

# downloading the kernel
wget https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.1-rc5/amd64/linux-headers-6.1.0-060100rc5-generic_6.1.0-060100rc5.202211132230_amd64.deb

wget https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.1-rc5/amd64/linux-headers-6.1.0-060100rc5_6.1.0-060100rc5.202211132230_all.deb

wget https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.1-rc5/amd64/linux-image-unsigned-6.1.0-060100rc5-generic_6.1.0-060100rc5.202211132230_amd64.deb

wget https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.1-rc5/amd64/linux-modules-6.1.0-060100rc5-generic_6.1.0-060100rc5.202211132230_amd64.deb

# install
sudo dpkg -i *.deb

# reboot the system
sudo reboot

# check the kernel version
uname -r 
```

2. [Linux-firmware.git](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git) needed for Arc GPU support since the graphics micro-controller requires "GuC" firmware v70 or higher.

```bash
cd ~/Downloads
git clone https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git firmware


# copy all contents to firmware
sudo cp -a firmware/. /lib/firmware/

```

3. Intel [Graphics Compiler](https://github.com/intel/intel-graphics-compiler) and [Graphics Compute](https://github.com/intel/compute-runtime). For OpenCL and Level Zero support, the latest Intel Compute-Runtime on GitHub along with associated GitHub components like Level-Zero and the Intel Graphics Compiler (IGC).

```bash
cd ~/Downloads 
mkdir neo && cd neo

# download 
wget https://github.com/intel/intel-graphics-compiler/releases/download/igc-1.0.12260.1/intel-igc-core_1.0.12260.1_amd64.deb
wget https://github.com/intel/intel-graphics-compiler/releases/download/igc-1.0.12260.1/intel-igc-media_1.0.12260.1_amd64.deb
wget https://github.com/intel/intel-graphics-compiler/releases/download/igc-1.0.12260.1/intel-igc-opencl-devel_1.0.12260.1_amd64.deb
wget https://github.com/intel/intel-graphics-compiler/releases/download/igc-1.0.12260.1/intel-igc-opencl_1.0.12260.1_amd64.deb
wget https://github.com/intel/compute-runtime/releases/download/22.43.24558/intel-level-zero-gpu-dbgsym_1.3.24558_amd64.ddeb
wget https://github.com/intel/compute-runtime/releases/download/22.43.24558/intel-level-zero-gpu_1.3.24558_amd64.deb
wget https://github.com/intel/compute-runtime/releases/download/22.43.24558/intel-opencl-icd-dbgsym_22.43.24558_amd64.ddeb
wget https://github.com/intel/compute-runtime/releases/download/22.43.24558/intel-opencl-icd_22.43.24558_amd64.deb
wget https://github.com/intel/compute-runtime/releases/download/22.43.24558/libigdgmm12_22.2.0_amd64.deb

#install
sudo dpkg -i *.deb

```
__`Alternative`__

Or we can use the ubuntu's package manager and install the dependancies.

```bash
# Source : https://dgpu-docs.intel.com/installation-guides/ubuntu/ubuntu-jammy-arc.html

# add intel package repository
sudo apt-get install -y gpg-agent

wget -qO - https://repositories.intel.com/graphics/intel-graphics.key |
  sudo gpg --dearmor --output /usr/share/keyrings/intel-graphics.gpg

echo 'deb [arch=amd64,i386 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/graphics/ubuntu jammy arc' | \
  sudo tee  /etc/apt/sources.list.d/intel.gpu.jammy.list

# install runtime packages
sudo apt-get install -y intel-opencl-icd intel-level-zero-gpu level-zero intel-media-va-driver-non-free libmfx1 libmfxgen1 libvpl2 libegl-mesa0 libegl1-mesa libegl1-mesa-dev libgbm1 libgl1-mesa-dev libgl1-mesa-dri libglapi-mesa libgles2-mesa-dev libglx-mesa0 libigdgmm12 libxatracker2 mesa-va-drivers mesa-vdpau-drivers mesa-vulkan-drivers va-driver-all libc6-dev udev

# optional : install developer packages
sudo apt-get install -y libigc-dev intel-igc-cm libigdfcl-dev libigfxcmrt-dev level-zero-dev

# reboot for changes 
sudo reboot
```


4. Mesa 23.0-devel, main branch. GitLab repository can be found [here](https://gitlab.freedesktop.org/mesa/mesa/).

Clone the repository
```bash
sudo apt install git
cd Downloads
git clone https://gitlab.freedesktop.org/mesa/mesa.git
```



# Miscellaneous

* Add support for Steam Games : Install i386 packages
```bash
# Source : https://dgpu-docs.intel.com/installation-guides/ubuntu/ubuntu-jammy-arc.html

sudo dpkg --add-architecture i386 
sudo apt-get update

sudo apt-get install  -y \
	udev mesa-va-drivers:i386 mesa-common-dev:i386 mesa-vulkan-drivers:i386 \
	libd3dadapter9-mesa-dev:i386 libegl1-mesa:i386  libegl1-mesa-dev:i386   \
	libgbm-dev:i386 libgl1-mesa-glx:i386 libgl1-mesa-dev:i386   \
	libgles2-mesa:i386 libgles2-mesa-dev:i386 libosmesa6:i386   \
	libosmesa6-dev:i386 libwayland-egl1-mesa:i386  libxatracker2:i386 \
	libxatracker-dev:i386 mesa-vdpau-drivers:i386  libva-x11-2:i386

# reboot for changes
sudo reboot
```