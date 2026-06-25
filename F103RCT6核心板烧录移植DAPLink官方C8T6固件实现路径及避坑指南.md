## 📝 DAPLink 移植到 STM32F103RCT6 完整总结

---

### 一、实现目标

将 ARM 官方的 DAPLink 调试器固件移植到 **STM32F103RCT6** 核心板上（256KB Flash，48KB RAM，2KB Page Size），替代官方默认的 STM32F103CBT6（128KB Flash，20KB RAM，1KB Page Size）。

---

### 二、环境准备

| 项目        | 内容                                                                          |
| --------- | --------------------------------------------------------------------------- |
| 操作系统      | Fedora Linux                                                                |
| 编译器       | ARM GCC 10.3.1（**关键：不能使用 15.x 版本**）                                         |
| 构建工具      | CMake + Make                                                                |
| Python 环境 | 虚拟环境 + progen + project-generator                                           |
| 源码        | [GitHub - ARMmbed/DAPLink · GitHub](https://github.com/ARMmbed/DAPLink.git) |

---

### 三、编译步骤

#### 1. 安装依赖

bash

sudo dnf install python3 python3-pip git cmake make

#### 2. 安装 ARM GCC 10.3.1

bash

cd ~/Downloads
wget https://developer.arm.com/-/media/Files/downloads/gnu-rm/10.3-2021.10/gcc-arm-none-eabi-10.3-2021.10-x86_64-linux.tar.bz2
tar -xjf gcc-arm-none-eabi-10.3-2021.10-x86_64-linux.tar.bz2
export PATH=$HOME/Downloads/gcc-arm-none-eabi-10.3-2021.10/bin:$PATH

#### 3. 克隆源码并配置 Python 环境

bash

git clone https://github.com/ARMmbed/DAPLink.git
cd DAPLink
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip setuptools wheel
pip install -r requirements.txt
pip install project-generator

#### 4. 修改源码（适配 RCT6）

**文件1：`source/hic_hal/stm32/stm32f103xb/daplink_addr.h`**

c

// 修改 Flash/RAM 大小
#define DAPLINK_ROM_SIZE                0x00040000      // 256KB
#define DAPLINK_RAM_SIZE                0x0000C000      // 48KB
// 修改分区大小
#define DAPLINK_ROM_BL_SIZE             0x0000C000      // 48KB Bootloader
#define DAPLINK_ROM_IF_SIZE             0x00033000      // 204KB Interface
#define DAPLINK_RAM_APP_SIZE            0x0000A000      // 40KB App RAM
// 修改 Flash Page 大小（关键：RCT6 是 2KB）
#define DAPLINK_SECTOR_SIZE             0x00000800      // 2KB
#define DAPLINK_MIN_WRITE_SIZE          0x00000800      // 2KB

**文件2：`source/target/flash_manager.c`**

c

// 修改 Flash 缓冲区大小
static uint8_t buf[2048];   // 原为 1024

**文件3：链接脚本（编译后修改 `build/stm32f103xb_bl.ld`）**

ld

MEMORY
{
  m_interrupts (RX) : ORIGIN = 0x08000000, LENGTH = 0x400
  m_text (RX) : ORIGIN = 0x08000400, LENGTH = 255K
  m_cfgrom (RW) : ORIGIN = 0x0803FC00, LENGTH = 0x400
  m_data (RW) : ORIGIN = 0x20000000, LENGTH = 48K
  m_cfgram (RW) : ORIGIN = 0x2000C000, LENGTH = 0x100
}

#### 5. 生成工程并编译

bash

# 生成 CMake 工程

progen generate -t cmake -p stm32f103xb_bl
progen generate -t cmake -p stm32f103xb_stm32f103rb_if

（若此步骤报错，则需要将setuptools降级：

##### 根据社区经验，**setuptools 81.0.0 或更早的 70.x 版本** 是兼容性较好的选择。具体安装命令：

##### 卸载当前版本

pip uninstall setuptools

##### 安装稳定版本（81.0.0）

pip install setuptools==81.0.0# 编译 Bootloader）

若一切正常，则进行继续以下步骤

cd projectfiles/cmake/stm32f103xb_bl
mkdir build && cd build
cmake .. -DCMAKE_C_COMPILER=arm-none-eabi-gcc -DCMAKE_CXX_COMPILER="" -DCMAKE_EXE_LINKER_FLAGS="-Wl,--print-memory-usage -Wl,--gc-sections"
make -j4

# 编译 Interface（同样操作）

cd ../../stm32f103xb_stm32f103rb_if
mkdir build && cd build
cmake .. -DCMAKE_C_COMPILER=arm-none-eabi-gcc -DCMAKE_CXX_COMPILER="" -DCMAKE_EXE_LINKER_FLAGS="-Wl,--print-memory-usage -Wl,--gc-sections"
make -j4

#### 6. 烧录固件

bash

# 用另一个调试器烧录 Bootloader

openocd -f interface/cmsis-dap.cfg -f target/stm32f1x.cfg -c "program stm32f103xb_bl.hex verify reset exit"

# USB 连接核心板，拖拽 Interface 固件到 MAINTENANCE U 盘

---

### 四、遇到的问题及解决方案

| 序号  | 问题                      | 错误现象                                                     | 解决方案                                                                           |
| --- | ----------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------ |
| 1   | `pkg_resources` 模块缺失    | `ModuleNotFoundError: No module named 'pkg_resources'`   | `pip install setuptools`                                                       |
| 2   | `progen` 安装失败           | `FileNotFoundError: 'README.md'`                         | 直接安装 `project-generator`：`pip install project-generator`                       |
| 3   | `cortex-m33` 不支持        | `Target cortex-m33 is not supported`                     | 只生成需要的工程：`progen generate -p stm32f103xb_bl -t cmake`，不要生成全部                   |
| 4   | CMake 找不到源文件            | `Cannot find source file: .../stm32f103xb_bl.c`          | 重新执行 `progen generate`，确保工程正确生成                                                |
| 5   | `arm-none-eabi-g++` 找不到 | `CMAKE_CXX_COMPILER is not a full path`                  | DAPLink 是纯 C 项目，设置 `-DCMAKE_CXX_COMPILER=""` 跳过                                |
| 6   | GCC 版本过高导致链接失败          | `ld returned 1 exit status`，无具体错误                        | 降级到 **GCC 10.3.1**（不能用 15.x）                                                   |
| 7   | 链接器警告被当作错误              | `-Wl,-fatal-warnings` 导致失败                               | 覆盖链接选项：`-DCMAKE_EXE_LINKER_FLAGS="-Wl,--print-memory-usage -Wl,--gc-sections"` |
| 8   | Flash 空间不足（47KB 限制）     | `m_text: 47KB/47KB 100%`                                 | 修改链接脚本，将 `m_text` 长度改为 `255K`                                                  |
| 9   | CMake 缓存路径错误            | `CMakeCache.txt directory is different`                  | `rm -rf build` 后重建                                                             |
| 10  | U 盘拖拽失败                 | `The starting address for the interface update is wrong` | 检查 Bootloader 和 Interface 的起始地址是否匹配，或用 OpenOCD 直接烧录                            |

---

### 五、关键成功因素

| 因素                    | 说明                                       |
| --------------------- | ---------------------------------------- |
| **GCC 版本**            | 必须使用 **10.3.1**，新版 15.x 会导致链接失败          |
| **Flash Page 大小**     | RCT6 是 **2KB**，必须修改 `SECTOR_SIZE` 和缓冲区大小 |
| **内存分区**              | 256KB Flash / 48KB RAM 的正确划分             |
| **跳过 C++ 编译器**        | DAPLink 是纯 C 项目                          |
| **去掉 fatal-warnings** | 避免警告被当作错误                                |
| **只生成需要的工程**          | 避免 `cortex-m33` 等不相关错误                   |

---

### 六、最终成果

| 产出            | 文件                                        |
| ------------- | ----------------------------------------- |
| Bootloader 固件 | `stm32f103xb_bl.hex` / `.bin`             |
| Interface 固件  | `stm32f103xb_stm32f103rb_if.hex` / `.bin` |
| 硬件            | F103RCT6 核心板 → DAPLink 调试器                |

---

### 七、经验总结

1. **官方源码 ≠ 开箱即用**：需要根据实际芯片修改配置

2. **工具链版本至关重要**：不是越新越好，要匹配项目要求

3. **链接脚本是最后一道坎**：内存布局不对，编译永远过不去

4. **开源项目需要耐心**：依赖、版本、路径问题层出不穷，但都能解决

5. **社区经验很有价值**：搜索类似问题的解决方案能节省大量时间

---

**这次移植的价值**：不仅得到了一个可用的 DAPLink 调试器，更重要的是完整掌握了嵌入式开源项目的编译、移植、调试全流程。这份经验比直接买十个成品调试器都有用。
