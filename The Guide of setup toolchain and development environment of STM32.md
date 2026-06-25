# The Guide of setup toolchain and development environment of STM32

## Toolchain：

1.IDE:VS Code+STM32CubeIDE for VS Code plugin+cortex-debug(by marus25) plugin

2.Compiler and Debuger:arm-none-eabi-gcc-10.3.1 + gdb

3.Hardware:DAPLink debuger,STM32 development board

4.DAPLink driver:OpenOCD or pyOCD

5.Code generator:STM32CubeMX

6.Build System:cmake,make

7.Others:git,screen/minicom/putty(serial port printing)

## Flow chart of setup develop environment

1.Install STM32CubeIDE for VS Code plugin+cortex-debug(by marus25) plugin in your VSC

2.Download gcc-arm-none-eabi-10.3 pakage and unzip it in D:\ (for example),add D:\arm-gnu-toolchain\bin into PATH

3.Check for installation,CLI: arm-none-eabi-gdb --version,arm-none-eabi-gcc --version

4.Download STM32CubeMX from ST official web and install it,use it to define the pin and pregenerate the code

5.Install CMake,Git and Make,you need MSYS2 environment to install make ,CLI: pacman -S mingw-w64-x86_64-make

6.Download OpenOCD prebuild from github ,add it into PATH.If you have Python environment,you can also try pyOCD,CLI: python -m pip install -U pyocd  check,CLI: pyocd --version

7.About the driver of DAPLink:If your Windows11 can not identify DAPLink,just restart it in the control panel,or use Zadig to change the driver to WinUSB

8.Use STM32CubeMX to configure pins and peripherals, then generate the project. **Crucially, under Project Manager → Toolchain/IDE, select `CMake`** (not Makefile).t is important

9.Open the project folder in VSCode. A popup should appear asking if you want to configure it as an ST project—click **Yes**. If not, press `Ctrl+Shift+P` and run `CMake: Configure` manually.

10.Configure the debuger: based on Cortex-debug plugin,setup a launch.json,configure as follow:

            {
    "version": "0.2.0",
    "configurations": [
        {
            "name": "STM32 Debug (DAPLink)",
            "type": "cortex-debug",
            "request": "launch",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceFolder}/build/Debug/your_project.elf",
            "servertype": "openocd",
"runToEntryPoint": "main",      

"configFiles": ["interface/cmsis-dap.cfg", "target/stm32f4x.cfg"],
            "gdbPath": "<path to your arm gcc>/arm-gnu-toolchain/bin/arm-none-eabi-gdb.exe"
        }
    ]
}    
