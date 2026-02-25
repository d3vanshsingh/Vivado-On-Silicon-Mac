# Vivado-On-Silicon-Mac

When trying to run Xilinx Vivado on Apple Silicon (M1/M2/M3), the common advice is to use a Windows 11 ARM Virtual Machine. This method works but comes with significant OS overhead, slow user interface response, and excessive resource use. More importantly, it goes against the industry standard. The global VLSI, EDA, and semiconductor industries mainly use Linux.

This repository outlines the specific architectural spoofs, compiler routing logic, and system-level linker traps needed to make the native Linux version of Xilinx Vivado work smoothly on an Ubuntu ARM VM. We will carefully bypass hardcoded OS architecture checks, direct cross-compilers to deceive the Vivado compilation engine, and modify multi-threading behaviors to avoid translation-layer crashes.


## The-Architecture

### The Translation layer: Rosetta
