# Ray Tracing on Intel Arc GPUs - Ubuntu 22.04 LTS

![Intel Arc logo](https://www.intel.in/content/dam/www/central-libraries/us/en/images/2022-09/arc-a-series-desktops-rwd.png.rendition.intel.web.1920.1080.png)
Image Source : [Intel](https://www.intel.in/content/www/in/en/products/details/discrete-gpus/arc.html)

# Mesa 3D Graphics Library

The Mesa project began as an open-source implementation of the OpenGL specification - a system for rendering interactive 3D graphics.

Over the years the project has grown to implement more graphics APIs, including OpenGL ES, OpenCL, OpenMAX, VDPAU, VA-API, Vulkan and EGL.

A variety of device drivers allows the Mesa libraries to be used in many different environments ranging from software emulation to complete hardware acceleration for modern GPUs.

Mesa ties into several other open-source projects: the Direct Rendering Infrastructure, X.org, and Wayland to provide OpenGL support on Linux, FreeBSD, and other operating systems.

Source : [Mesa 3D](https://docs.mesa3d.org/index.html)


# Table of Contents
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Downloads](#downloads)
- [Installation](#installation)
  - [Upgrade Kernel](#upgrade-kernel)
  - [Update to linux-firmware.git](#update-to-linux-firmwaregit)
  - [Installation of necessary components](#installation-of-necessary-components)
  - [Build Mesa3D and install](#build-mesa3d-and-install)
- [Testing Vulkan Ray Tracing Support](#testing-vulkan-ray-tracing-support)
- [Known issues](#known-issues)
- [Miscellaneous](#miscellaneous)

# Prerequisites

* Check for Resizable BAR (ReBAR) support.
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

* Install dependencies for building Mesa. We will be using default LLVM 13 shipped with Ubuntu 22.04.
```bash
sudo apt-get install build-essential git cmake
sudo apt-get build-dep mesa

# additional packages needs to installed
sudo apt-get install libllvmspirvlib15 libllvmspirvlib-dev libclc-15 libclang-dev python3-ply python-is-python3
```

# Configuration

| Specifications | Detail                                                  |
| ------------------- | ------------------------------------------- |
| Operating System    | Ubuntu 22.04 LTS                            |
| Kernel Version      | Linux Kernel 6.3-rc6                        |
| CPU                 | Intel i7-1165G7 (8 cores)                   |
| Integrated Graphics | Intel® Iris® Xe Graphics                    |
| Discrete Graphics   | Intel Arc A770 16GB                         |

# Downloads

1. Linux 6.5.1 Kernel from [Ubuntu archives](https://kernel.ubuntu.com/~kernel-ppa/mainline/). 

```bash
sudo apt-get update && sudo apt-get full-upgrade -y # update the system
sudo apt install wget # if not present
cd ~/Downloads 
mkdir kernel-6.5.1 && cd kernel-6.5.1 # kernel directory

# downloading the kernel
https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.5.1/amd64/linux-headers-6.5.1-060501-generic_6.5.1-060501.202309020842_amd64.deb

https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.5.1/amd64/linux-headers-6.5.1-060501_6.5.1-060501.202309020842_all.deb

https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.5.1/amd64/linux-image-unsigned-6.5.1-060501-generic_6.5.1-060501.202309020842_amd64.deb

https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.5.1/amd64/linux-modules-6.5.1-060501-generic_6.5.1-060501.202309020842_amd64.deb
```

2. [Linux-firmware.git](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git).

```bash
cd ~/Downloads
git clone https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git firmware
```

3. Intel [Graphics Compiler](https://github.com/intel/intel-graphics-compiler) and [Graphics Compute](https://github.com/intel/compute-runtime). 

```bash
cd ~/Downloads 
mkdir neo && cd neo

# download 
wget https://github.com/intel/intel-graphics-compiler/releases/download/igc-1.0.14062.11/intel-igc-core_1.0.14062.11_amd64.deb
wget https://github.com/intel/intel-graphics-compiler/releases/download/igc-1.0.14062.11/intel-igc-opencl_1.0.14062.11_amd64.deb
wget https://github.com/intel/compute-runtime/releases/download/23.22.26516.18/intel-level-zero-gpu-dbgsym_1.3.26516.18_amd64.ddeb
wget https://github.com/intel/compute-runtime/releases/download/23.22.26516.18/intel-level-zero-gpu_1.3.26516.18_amd64.deb
wget https://github.com/intel/compute-runtime/releases/download/23.22.26516.18/intel-opencl-icd-dbgsym_23.22.26516.18_amd64.ddeb
wget https://github.com/intel/compute-runtime/releases/download/23.22.26516.18/intel-opencl-icd_23.22.26516.18_amd64.deb
wget https://github.com/intel/compute-runtime/releases/download/23.22.26516.18/libigdgmm12_22.3.0_amd64.deb
https://github.com/intel/intel-graphics-compiler/releases/download/igc-1.0.14062.11/intel-igc-media_1.0.14062.11_amd64.deb
https://github.com/intel/intel-graphics-compiler/releases/download/igc-1.0.14062.11/intel-igc-opencl-devel_1.0.14062.11_amd64.deb
```
__`Alternative`__

We can use the ubuntu's package manager and install the necessary dependancies. Scroll down to installation for details.

<br>

4. Mesa 23.3.0-devel, main branch. GitLab repository can be found [here](https://gitlab.freedesktop.org/mesa/mesa/).

Clone the repository
```bash
cd ~/Downloads
git clone https://gitlab.freedesktop.org/mesa/mesa.git mesa
```

# Installation

## Upgrade Kernel
Linux 6.5.1 Kernel from [Ubuntu archives](https://kernel.ubuntu.com/~kernel-ppa/mainline/). We will manually download the necessary files using wget and install them with dpkg but before that we should make sure that our system is updated.

```bash
cd ~/Downloads/kernel-6.5.1

# install
sudo dpkg -i *.deb

# reboot the system
sudo reboot

# check the kernel version
uname -r 
```

## Update to linux-firmware.git
[Linux-firmware.git](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git) needed for Arc GPU support since the graphics micro-controller requires "GuC" firmware v70 or higher.

```bash
cd ~/Downloads

# copy all contents to firmware
sudo cp -a firmware/. /lib/firmware/
```

## Installation of necessary components
Intel [Graphics Compiler](https://github.com/intel/intel-graphics-compiler) and [Graphics Compute](https://github.com/intel/compute-runtime). For OpenCL and Level Zero support, the latest Intel Compute-Runtime on GitHub along with associated GitHub components like Level-Zero and the Intel Graphics Compiler (IGC).

```bash
cd ~/Downloads/neo

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

## Build Mesa3D and install
Build mesa from source with few parameters required for Intel Arc Ray Tracing to work.
```bash
cd mesa
mkdir build && cd build

# configure meson to build accordingly
meson .. -Dvulkan-drivers=intel -Dintel-clc=enabled  -Dglvnd=true

# install using ninja
sudo ninja install
```

Since there are multiple mesa versions present in the system it will create redundancy and will error out during runtime. Thus a brute-force measure would be to rename the older Vulkan folder consisting of icd drivers.

```bash
cd /usr/share/
sudo mv vulkan vulkan.old.bak
```


# Testing Vulkan Ray Tracing Support
We will be using the official Khronos Vulkan samples provided by [Sacsha Willems](https://www.saschawillems.de/) in the following [repository](https://github.com/SaschaWillems/Vulkan) for our testing purpose.

__About the repository__ 

To help people get started with the Vulkan graphics api Sacsha released a repository of examples C++ along with the start of Vulkan. He started developing these before Vulkan was publicly released while being a member of the Vulkan advisory panel, and have since then added more and more examples, demonstrating many different aspects of the api.

The list of examples (already more than 60) range from basic api usage to more complex setups, and also include examples for different rendering methods and effects (physical based rendering, screen space ambient occlusion, deferred rendering, etc.) and demonstrating use of several extensions.

Source : [saschawillems.de](https://www.saschawillems.de/creations/vulkan-examples/)

Clone the repository
```bash
git clone --recursive https://github.com/SaschaWillems/Vulkan.git

# download additional assets
cd Vulkan
python download_assets.py
```

Build the demos
```bash
cd ~/Downloads/Vulkan
mkdir build && cd build

cmake ..
cmake --build .
```

How to run a demo 
```bash
cd build/bin
./raytracingbasics
```

__Ray Tracing__

Simple GPU ray tracer with shadows and reflections using a compute shader. No scene geometry is rendered in the graphics pass.

![compute_ray_tracing](https://github.com/afkniladri/Intel-Arc-Ray-Tracing-Enablement/blob/main/assets/demos/computeRayTracing.gif)


__Mesh Shaders (VK_EXT_mesh_shader)__

Basic sample demonstrating how to use the mesh shading pipeline as a replacement for the traditional vertex pipeline.

Needs additional runtime parameters : ANV_EXPERIMENTAL_NV_MESH_SHADER=1

![mesh_shaders](https://github.com/afkniladri/Intel-Arc-Ray-Tracing-Enablement/blob/main/assets/demos/meshShader.gif)


__Ray Query__

Ray queries add acceleration structure intersection functionality to non ray tracing shader stages. This allows for combining ray tracing with rasterization. This example makes uses ray queries to add ray casted shadows to a rasterized sample in the fragment shader.

![ray_query](https://github.com/afkniladri/Intel-Arc-Ray-Tracing-Enablement/blob/main/assets/demos/rayQuery.gif)


__Ray Traced Shadows__

Adds ray traced shadows casting using the new ray tracing extensions to a more complex scene. Shows how to add multiple hit and miss shaders and how to modify existing shaders to add shadow calculations.

![ray_traced_shadows](https://github.com/afkniladri/Intel-Arc-Ray-Tracing-Enablement/blob/main/assets/demos/rayTracedShadows.gif)


__Basic Ray Tracing__

Basic example for doing hardware accelerated ray tracing using the VK_KHR_acceleration_structure and VK_KHR_ray_tracing_pipeline extensions. Shows how to setup acceleration structures, ray tracing pipelines and the shader binding table needed to do the actual ray tracing.

![basic_ray_tracing](https://github.com/afkniladri/Intel-Arc-Ray-Tracing-Enablement/blob/main/assets/demos/rayTracingBasic.gif)


__Callable Ray Tracing Shaders__

Callable shaders can be dynamically invoked from within other ray tracing shaders to execute different shaders based on dynamic conditions. The example ray traces multiple geometries, with each calling a different callable shader from the closest hit shader.

![ray_tracing_shaders](https://github.com/afkniladri/Intel-Arc-Ray-Tracing-Enablement/blob/main/assets/demos/rayTracingCallable.gif)


__Ray Traced Reflections__

Renders a complex scene with reflective surfaces using the new ray tracing extensions. Shows how to do recursion inside of the ray tracing shaders for implementing real time reflections.

![ray_traced_shaders](https://github.com/afkniladri/Intel-Arc-Ray-Tracing-Enablement/blob/main/assets/demos/rayTracingReflections.gif)




# Known issues

__Graphics Pipeline Library (VK_EXT_graphics_pipeline_library)__

Uses the graphics pipeline library extensions to improve run-time pipeline creation. Instead of creating the whole pipeline at once, this sample pre builds shared pipeline parts like like vertex input state and fragment output state. These are then used to create full pipelines at runtime, reducing build times and possible hick-ups.

`Reason : Not yet supported upstream, the MR adding support : https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/15637 `

![graphics_pipeline_library](https://github.com/afkniladri/Intel-Arc-Ray-Tracing-Enablement/blob/main/assets/demos/errors/graphicsPipeline.gif)


__Texture Mapping & Texture Arrays__

Loads a 2D texture from disk (including all mip levels), uses staging to upload it into video memory and samples from it using combined image samplers. 

Loads a 2D texture array containing multiple 2D texture slices (each with its own mip chain) and renders multiple meshes each sampling from a different layer of the texture. 2D texture arrays don't do any interpolation between the slices.

`Reason : There is no support for sparse binding the kernel driver.`

![sparse_binding](https://github.com/afkniladri/Intel-Arc-Ray-Tracing-Enablement/blob/main/assets/demos/errors/textureSparseResidency.gif)


__Variable Rate Shading (VK_NV_shading_rate_image)__

Uses a special image that contains variable shading rates to vary the number of fragment shader invocations across the framebuffer. This makes it possible to lower fragment shader invocations for less important/less noisy parts of the framebuffer.

`Reason : The extension VK_KHR_shading_rate_image is not supported. Although a similar extension VK_KHR_fragment_shading_rate is supported on Intel Arc.`

![variable_rate_shading](https://github.com/afkniladri/Intel-Arc-Ray-Tracing-Enablement/blob/main/assets/demos/errors/variableRateShading.gif)


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
