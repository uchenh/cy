# Ubuntu 22.04 科学计算环境依赖安装部署文档

> 适用系统：Ubuntu 22.04 LTS (jammy)
> 对应需求：C++ 编译链 + MPI + CMake + Boost + yaml-cpp + HDF5(MPI) + Python + 后处理工具

---

## 一、原始依赖清单（INSTALLATION）

Requisites：**C++ compiler and MPI library, CMAKE, BOOST, YAML (yaml-cpp version 1.2) & HDF5 libraries/modules (compatible with MPI)**

| 命令 | 内容 | 分类 |
|------|------|------|
| `sudo apt install build-essential make` | C++ 编译器与工具链 | if not already available |
| `sudo apt install libopenmpi-dev cmake libboost-all-dev libyaml-cpp-dev libhdf5-openmpi-dev` | OpenMPI/CMake/Boost/yaml-cpp/HDF5 | required |
| `sudo apt install texlive-full python3.8 python3-pip paraview` | LaTeX/Python/ParaView 后处理 | auxiliary |
| `sudo pip3 install numpy scipy python3-matplotlib h5py` | Python 科学计算包 | auxiliary |

---

## 二、部署执行过程（含关键问题处理）

### 1. 更新源 + 编译工具链
```bash
sudo apt update
sudo apt install -y build-essential make
```

### 2. 必需依赖库（openmpi / cmake / boost / hdf5）
```bash
sudo apt install -y libopenmpi-dev cmake libboost-all-dev libhdf5-openmpi-dev
```

> **问题①**：apt 里的 `libyaml-cpp-dev` 仅为 **0.7.0**，不满足"yaml-cpp version 1.2"要求，需源码编译，见步骤 5。

### 3. 后处理包（texlive-full / paraview）
```bash
sudo apt install -y texlive-full python3-pip paraview
```
> **注意**：`texlive-full` 体积大（约 5GB），下载耗时较长属正常。

### 4. Python 3.8 安装（本项目最麻烦的一点）
`python3.8` 包在 Ubuntu 22.04 源中**不存在**（22.04 默认 Python 3.10），且 deadsnakes PPA 不提供 jammy 版 3.8，只能源码编译：

```bash
# 编译依赖
sudo apt install -y wget build-essential libssl-dev zlib1g-dev \
    libbz2-dev libreadline-dev libsqlite3-dev libncursesw5-dev \
    libgdbm-dev libnss3-dev libffi-dev tk-dev liblzma-dev

# 下载（若 gzip: unexpected end of file = 文件损坏，删掉重试 wget -c）
cd ~/Downloads
wget https://www.python.org/ftp/python/3.8.20/Python-3.8.20.tgz
tar -xzf Python-3.8.20.tgz
cd Python-3.8.20
./configure --prefix=/usr --enable-shared --with-ensurepip=install
make -j$(nproc)
sudo make install
sudo ldconfig
```

> 装完后 `python3.8` 与 `pip3`（指向 3.8）均可直接用。

### 5. yaml-cpp 源码编译（满足 YAML 1.2 规范）
- **关键澄清**："yaml-cpp version 1.2" 指的并非 1.x 库版本，而是 **YAML 1.2 规范**；
- yaml-cpp 官方版本号一直是 **0.x**，最新为 **0.9.0**，即完整支持 YAML 1.2 规范；
- 仓库**不存在** `1.2`、`1.2.2` 等 tag。

```bash
cd ~/Downloads
git clone https://github.com/jbeder/yaml-cpp.git
cd yaml-cpp
git checkout yaml-cpp-0.9.0        # 或直接留在 main 分支，等价满足要求
mkdir -p build && cd build
cmake -DYAML_BUILD_SHARED_LIBS=ON -DCMAKE_INSTALL_PREFIX=/usr ..
sudo make install -j$(nproc)
sudo ldconfig
```

### 6. Python 科学计算包
> 原命令的 `python3-matplotlib` 不是有效的 pip 包名，需改为 `matplotlib`。
```bash
sudo pip3 install numpy scipy matplotlib h5py
```

---

## 三、最终安装结果核对

| 组件 | 期望版本 | 实测版本 | 状态 |
|------|---------|---------|:---:|
| GCC / G++ | — | 11.4.0 | ✅ |
| Make | — | 4.3 | ✅ |
| Open MPI | — | 4.1.2 | ✅ |
| CMake | — | 3.22.1 | ✅ |
| Boost | 1.74 | `1_74` | ✅ |
| HDF5 (MPI 并行版) | — | 头文件齐全 | ✅ |
| yaml-cpp | 支持 YAML 1.2 | 0.9.0 | ✅ |
| Python | 3.8 | 3.8.x | ✅ |
| pip | — | 23.0.1 | ✅ |
| NumPy | — | 1.24.x | ✅ |
| SciPy | — | 1.10.1 | ✅ |
| Matplotlib | — | 3.7.x | ✅ |
| h5py | — | 3.11.0 | ✅ |
| pdfTeX / TeX Live | — | 2022 | ✅ |
| ParaView | — | 5.10.0-RC1 | ✅ |

> 结论：核心科学计算栈（编译链 + MPI + HDF5 + Python + numpy/scipy/matplotlib/h5py + LaTeX + ParaView）**全部就绪**。

---

## 四、验证命令（逐条手输）

### 1. C++ 编译链
```bash
g++ --version
```
应出现 `g++ (Ubuntu 11.4.0...)`。

### 2. MPI
```bash
mpirun --version
```
应出现 `Open MPI ... 4.1.2`。

### 3. CMake
```bash
cmake --version
```
应出现 `cmake version 3.22.1`。

### 4. Boost
```bash
grep BOOST_LIB_VERSION /usr/include/boost/version.hpp
```
应出现 `"1_74"`。

### 5. HDF5（MPI 并行版）
```bash
ls /usr/include/hdf5/openmpi
h5cc -showconfig | grep -i parallel
```
头文件存在，且出现 `Parallel HDF5 ... yes` 即 OK（无 `h5cc` 则试 `h5pcc`）。

### 6. yaml-cpp
```bash
pkg-config --modversion yaml-cpp
```
应出现 `0.9.0`（新版无 `/usr/include/yaml-cpp/yaml-cpp.hpp` 属正常，不影响链接 `-lyaml-cpp`）。

### 7. Python 3.8
```bash
python3.8 --version
```
应出现 `Python 3.8.x`。

### 8. pip
```bash
pip3 --version
```
应出现 `...(python 3.8)`。

### 9. Python 科学计算包（逐条）
```bash
python3.8 -c "import numpy; print('numpy OK', numpy.__version__)"
python3.8 -c "import scipy; print('scipy OK', scipy.__version__)"
python3.8 -c "import matplotlib; print('matplotlib OK', matplotlib.__version__)"
python3.8 -c "import h5py; print('h5py OK', h5py.__version__)"
```
每条出现 `xxx OK` 且无 `ModuleNotFoundError` 即对应包已装。

### 10. LaTeX
```bash
pdflatex --version
```
出现版本号即 OK。

### 11. ParaView
```bash
paraview --version
```
出现 `paraview version 5.10.x` 即 OK；若 `command not found` 则未装，执行 `sudo apt install -y paraview`。

---

## 五、踩坑与经验总结

1. **python3.8 在 22.04 无预装包**：必须源码编译，`--prefix=/usr` 装进系统后才有全局 `python3.8`/`pip3`。
2. **yaml-cpp 没有 1.x 版本**：需求中的 "version 1.2" 指 YAML 1.2 规范，装 0.9.0 即可；apt 里的 0.7.0 也符合要求（但按原需求建议编译最新版）。
3. **`gzip: stdin: unexpected end of file`**：表示下载的 `.tgz` 损坏/不完整，`wget -c` 续传或换国内镜像重下。
4. **`python3-matplotlib` 不是有效 pip 包名**：用 `matplotlib` 替代。
5. **忽略的错误**：`python3 -c "import crash; parallel"` 报 `ModuleNotFoundError: No module named 'CommandNotFound'` 属命令拼写错误，与软件安装无关，不影响运行。