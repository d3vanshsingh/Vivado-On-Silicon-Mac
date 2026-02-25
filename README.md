# Vivado-On-Silicon-Mac

When trying to run Xilinx Vivado on Apple Silicon (M1/M2/M3), the common advice is to use a Windows 11 ARM Virtual Machine. This method works but comes with significant OS overhead, slow user interface response, and excessive resource use. More importantly, it goes against the industry standard. The global VLSI, EDA, and semiconductor industries mainly use Linux.

This repository outlines the specific architectural spoofs, compiler routing logic, and system-level linker traps needed to make the native Linux version of Xilinx Vivado work smoothly on an Ubuntu ARM VM. We will carefully bypass hardcoded OS architecture checks, direct cross-compilers to deceive the Vivado compilation engine, and modify multi-threading behaviors to avoid translation-layer crashes.


## The Architecture

### The Translation layer: Rosetta
Apple Silicon processors use the ARM64 (AArch64) instruction set architecture. In contrast, Vivado’s binaries are compiled specifically for x86_64 (Intel/AMD). To connect these two systems, macOS offers Rosetta for Linux through the Virtualization.framework. When you boot an Ubuntu ARM VM, macOS mounts the Rosetta runtime directly into the Linux file system. Linux uses a kernel feature called binfmt_misc (Miscellaneous Binary Formats) to recognize x86_64 ELF (Executable and Linkable Format) binaries. When you run Vivado, the Linux kernel intercepts the Intel binary and sends it to Rosetta. Rosetta then performs Just-In-Time (JIT) and Ahead-Of-Time (AOT) translation of the Intel instructions into native ARM instructions.

### The Compiler Pipeline & Link-Time Optimization (LTO)
During Behavioral Simulation, Vivado's xelab engine converts Verilog Register-Transfer Level (RTL) code into C code. It then uses a bundled GCC (GNU Compiler Collection) toolchain to compile this C code into an executable simulation snapshot. The standard GCC pipeline has two main phases: Compilation: gcc converts C source code into object files (.o). Linking: The system linker (ld) combines these object files with Vivado's proprietary simulation libraries. The Crash Mechanism: Vivado’s GCC passes proprietary Link-Time Optimization (LTO) flags, specifically -plugin and -plugin-opt, to the Linux system linker. Because the ARM system linker cannot process x86-specific LTO plugins, it panics and throws a fatal bad -plugin-opt option or [Common 17-39] error.

## Multi-Threading Issue
Synthesis is the resource-intensive process of converting RTL code into physical FPGA logic gates. Vivado is built for enterprise workstations and uses several x86 threads, usually 8, to speed up this process. When Rosetta tries to translate and map 8 highly parallelized, heavy x86 threads to macOS's ARM thread scheduler, it causes serious race conditions. This leads to Segmentation Faults or the well-known ERROR: [XSIM 43-3984] Multiple occurrences of option --mt crash.

## Setting up Ubuntu and Installing Vivado .tar file

### Virtual Machine setup
1. Deploy an Ubuntu ARM VM (24.04 LTS or newer) utilizing virtualization software such as UTM, Parallels Desktop, or VMware Fusion.
2. Ensure that "Rosetta for Linux" is explicitly enabled in your VM configuration settings.
3. Download the All OS Single-File Download (TAR/GZIP) from the AMD/Xilinx website and extract it within the VM:
   `tar -xvf FPGAs_AdaptiveSoCs_Unified_202X.X_XXXX_XXXX.tar.gz`
### Provisioning Cross Compilers
Because the host environment is ARM-based, the native compilers generate ARM binaries. We must install dedicated cross-compilers capable of generating the x86_64 code that Vivado expects.
```
sudo apt update
sudo apt install gcc-x86-64-linux-gnu g++-x86-64-linux-gnu
```
## Creating a fake_bin spoof for Installer
Vivado’s setup script (xsetup) checks the system architecture with the uname -m command. If the system shows aarch64, the installer immediately stops and shows an "Unsupported architecture" error. We will create a fake_bin directory to intercept these system calls and mimic an x86_64 environment.

1. Initialize the directory
   `mkdir -p ~/fake_bin`
2. Spoof the Architecture (uname):
   `nano ~/fake_bin/uname`

   paste this inside
   ```
   #!/bin/bash
   if [[ "$1" == "-m" ]]; then
        echo "x86_64" # Deceive the Vivado installer
   else
    /usr/bin/uname "$@"
   fi
   ```
2. Reroute the Compilers (gcc and g++):
   Create `nano ~/fake_bin/gcc:`
   ```
   #!/bin/bash
   /usr/bin/x86_64-linux-gnu-g++ "$@"
   ```
   Create `nano ~/fake_bin/g++`
   ```
   #!/bin/bash
   /usr/bin/x86_64-linux-gnu-g++ "$@"
   ```
3. Activation and Installation
   ```
   chmod +x ~/fake_bin/*
   export PATH=~/fake_bin:$PATH
   ```
   With the PATH environment variable temporarily modified, execute the installer:
   `sudo -E ./xsetup`
   (Note: The -E flag is mandatory. It preserves your user environment variables, ensuring the root user executing xsetup still routes through your spoofed fake_bin directory). Proceed to install the Vivado ML Standard Edition.

## Resolving Simulation Issue 
To resolve the [Common 17-39] LTO plugin crash during Behavioral Simulation, we must replace the default system linker with a middleware Bash script. This script will act as a filter, scrubbing out the incompatible optimization flags before forwarding the sanitized arguments to the true x86 cross-linker.

1. Open the system linker for modification
   `sudo nano /usr/bin/ld`
2. Implement the Interceptor Logic:
   Replace the file contents with the following script:
```
    #!/bin/bash
    declare -a args
    skip=0

    # Iterate through all arguments passed by Vivado's GCC engine
    for i in "$@"; do
        if [ $skip -eq 1 ]; then
            skip=0
            continue
        fi
    
        # TRAP 1: Identify and discard standalone '-plugin' flags and their subsequent path arguments
        if [[ "$i" == "-plugin" ]]; then
            skip=1
            continue
        fi
    
        # TRAP 2: Identify and discard attached plugin arguments (e.g., -plugin-opt=...)
        if [[ "$i" == -plugin* ]]; then
            continue
        fi
    
         # Retain all validated, safe arguments
        args+=("$i")
   done

   # Route the sanitized arguments to the genuine x86_64 cross-linker
   if [[ "$*" == *"elf_x86_64"* ]]; then
       /usr/bin/x86_64-linux-gnu-ld "${args[@]}"
   else
       /usr/bin/ld_arm "$@"
   fi
   ```
