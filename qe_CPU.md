Quantum ESPRESSO 安装与环境配置报告

本文档记录了在 Ubuntu 22.04（无 sudo 权限）环境下，从源码编译 Quantum ESPRESSO 并配置 TritonDFT 计算管线的完整过程，包括遇到的问题和解决方案。

---

1. 环境概况

项目
详情
OS
Ubuntu 22.04 LTS, Linux 6.8.0
CPU
52 核
Python
3.10.12
包管理
Miniconda3（conda 26.1.1），无 sudo 权限
目标 QE 版本
v7.5（兼容论文要求的 v7.4）

2. QE 源码来源

TritonDFT 作者仓库（github.com/Leo9660/TritonDFT）的 dev_lyc 分支中，.gitmodules 引用了 QE 的 git submodule：

- 仓库：https://github.com/QEF/q-e.git
- Commit：ddbec46536cc9c6f8cf3cd9db486573bcc5424a4（develop 分支，比 qe-7.4 tag 新 807 个 commit，标记为 qe-7.5CN）
匿名发布的源码（tritonDFT-src）丢失了这个 submodule，需要手动克隆。

git clone https://github.com/QEF/q-e.git tritonDFT-src/QuantumE
cd tritonDFT-src/QuantumE
git checkout ddbec46536cc9c6f8cf3cd9db486573bcc5424a4

放在 QuantumE/ 目录下可与代码默认配置（config.py: DEFAULT_QE_BIN_DIR = "QuantumE/bin"）对齐。

3. 编译依赖安装

QE 编译需要 Fortran 编译器、MPI、BLAS/LAPACK、FFTW。系统已有 gfortran 和 runtime 库，但缺少 dev headers 和 MPI。

无 sudo 的解决方案：通过 conda-forge 安装全部编译依赖。

# 首次使用需同意 conda TOS
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r

# 安装编译依赖（安装到 base 环境）
conda install -y -c conda-forge openmpi openmpi-mpifort fftw lapack blas

安装后得到：openmpi 5.0.8、fftw 3.3.10、lapack 3.11.0、gfortran 15.2.0（conda-forge 版本）。

4. Configure

坑 1：MPI 检测失败

直接运行 ./configure，输出：

Parallel environment not detected (is this a parallel machine?).  
Configured for compilation of serial executables.

原因：conda 的 `mpif90` 是一个 wrapper 脚本，`./configure` 内部的 `AC_PROG_FC` 会将其解析为底层的 `x86_64-conda-linux-gnu-gfortran`，然后用这个裸编译器去做 MPI 链接测试，自然找不到 MPI 库。

解决方案：显式指定 `FC=mpif90` 并加 `--enable-parallel`：

eval "$(conda shell.bash hook)" && conda activate base
export PATH=$CONDA_PREFIX/bin:$PATH
./configure --enable-parallel FC=mpif90 CC=mpicc

确认输出包含：
setting DFLAGS... -D__FFTW3 -D__MPI -D__MPI_MODULE
Parallel environment detected successfully.

5. 编译

坑 2：GFortran 15 类型检查过严

Error: Type mismatch between actual argument at (1) and actual argument at (2)

原因：GFortran 10+ 默认将 Fortran 接口类型不匹配视为 error。QE 的 MPI wrapper 中存在 `REAL/COMPLEX` 类型混用（这在旧版 Fortran 中是常见做法）。conda-forge 的 GFortran 15.2.0 对此报错。

解决方案：在 `make.inc` 的 `FFLAGS` 行末尾添加：

-fallow-argument-mismatch

坑 3：链接器不识别 -Wl 选项

x86_64-conda-linux-gnu-ld: unrecognized option '-Wl,-O2'

原因：`make.inc` 中 `LD` 被 configure 设为裸链接器 `x86_64-conda-linux-gnu-ld`。`-Wl,xxx` 是编译器 wrapper（gcc/gfortran）传给链接器的选项格式，裸 `ld` 不认识。

解决方案：将 `make.inc` 中的 `LD` 改为：

LD = mpif90

执行编译

make -j16 pw pp ph

编译成功，生成 54 个可执行文件于 QuantumE/bin/，包括 pw.x（v7.5）、pp.x、ph.x、dos.x、bands.x、projwfc.x 等。

6. Python 环境配置

TritonDFT 的 requirements.txt 指定了较老的 pinned 版本（numpy 1.26.4、pymatgen 2023.12.18 等），与系统已有的新版本不兼容（numpy.dtype size changed 错误）。

解决方案：创建独立 conda 环境：

conda create -n tritondft python=3.10 -y
conda activate tritondft
pip install -r requirements.txt
conda install -y -c conda-forge openmpi   # QE 运行时 MPI 依赖

7. 路径配置

坑 4：赝势相对路径在 agent 子目录下失效

代码默认赝势路径为 ../PseudoDojo/SR_v0.4.1/PBE_standard（相对于仓库根目录）。但 DFTAgent 在 <work_dir>/<date>/<run_id>/ 子目录下执行 QE，相对路径解析失败。

解决方案：创建 `config/config.yaml`，使用绝对路径：

pseudo:
  LDA: /absolute/path/to/PseudoDojo/SR_v0.4.1/LDA_standard
  PBE: /absolute/path/to/PseudoDojo/SR_v0.4.1/PBE_standard
  PBEsol: /absolute/path/to/PseudoDojo/SR_v0.4.1/PBEsol_standard
qe_bin_dir: /absolute/path/to/QuantumE/bin

8. 验证结果

手动验证（不经过 agent）

Si vc-relax 计算，PBE 泛函，ecutwfc=40 Ry，6×6×6 k-mesh：

指标
结果
PBE 参考 (Materials Project)
偏差
晶格常数
5.4687 Å
5.466 Å
0.05%
能量/原子
-115.140 eV
—
—

Agent 端到端验证

Claude Sonnet → Plan → 参数推断 → 脚本生成 → QE 执行 → 结果解析，全流程成功。

9. 关于赝势选择

TritonDFT 自带 PseudoDojo 赝势库（`PseudoDojo/SR_v0.4.1/`），包含 LDA/PBE/PBEsol 泛函各 72 个元素。`dft-quick-start.md` 中提到的 GRBV 是另一套赝势库（Garrity 等人发布），功能类似但来源不同。我们使用项目自带的 PseudoDojo，与代码配置一致。

10. 踩坑汇总

#
问题
原因
解决方案
1
无 sudo 权限
服务器限制
conda-forge 安装所有编译依赖
2
MPI 检测失败
conda mpif90 wrapper 被 configure 解析为裸 gfortran
./configure --enable-parallel FC=mpif90 CC=mpicc
3
类型不匹配 error
GFortran 15 默认严格类型检查
FFLAGS 加 -fallow-argument-mismatch
4
链接器不识别 -Wl
LD 设为裸 ld 而非编译器 wrapper
make.inc 中 LD = mpif90
5
numpy 版本不兼容
requirements.txt pinned 老版本
独立 conda 环境 tritondft
6
赝势文件找不到
agent 在子目录执行 QE，相对路径失效
config.yaml 使用绝对路径

12. GPU 版本补充（2026-04-15）

CPU 版完成后，另编了 v7.5CN 的 GPU 版位于 QuantumE-gpu/bin/pw.x（同 commit ddbec4653）。完整记录见 progress/logs/qe-gpu-rebuild.md，这里只列出 GPU 构建新增的踩坑：

#
问题
原因
解决方案
| 7 | docker 无 sudo 不能用 | 用户不在 docker 组 | 改用 podman 3.4 rootless |
| 8 | podman 3.4 `--device nvidia.com/gpu=all` 报错 | CDI 需 podman 4.1+ | 改用 OCI prestart hook（`~/.config/containers/oci/hooks.d/`）|
| 9 | nvidia-container-runtime-hook 默认 config 需 root | load-kmods=true / no-cgroups=false | 在 ~/.config/nvidia-container-runtime/config.toml 覆盖为 load-kmods=false、no-cgroups=true；hook JSON env 注入 XDG_CONFIG_HOME |
| 10 | NGC quantum_espresso 镜像不含 nvfortran | 是 runtime-only 镜像 | 改用 nvcr.io/nvidia/nvhpc:24.7-devel-cuda_multi-ubuntu22.04 作编译环境 |
| 11 | ./configure 成功但找不到子模块 | QE 7.5 的 external/devxlib 仅 "registered"，未 clone | git submodule update --init external/devxlib external/fox |
| 12 | 自编 GPU pw.x 裸跑 MPI_Init 失败 | 容器内 hpcx openmpi 硬编码路径 /proj/nv/... | 必须通过 mpirun --allow-run-as-root -np N pw.x 启动 |
| 13 | Si 2-atom 测试 GPU 比 CPU 慢 | 小体系 GPU 启动开销主导 | 用 16-atom 及以上体系做 bench |

11. 快速复现命令

# 1. 安装编译依赖
conda install -y -c conda-forge openmpi openmpi-mpifort fftw lapack blas

# 2. 克隆 QE
git clone https://github.com/QEF/q-e.git tritonDFT-src/QuantumE
cd tritonDFT-src/QuantumE
git checkout ddbec46536cc9c6f8cf3cd9db486573bcc5424a4

# 3. Configure
eval "$(conda shell.bash hook)" && conda activate base
./configure --enable-parallel FC=mpif90 CC=mpicc

# 4. 修改 make.inc（两处）
#    FFLAGS 末尾加:  -fallow-argument-mismatch
#    LD 改为:         mpif90

# 5. 编译
make -j16 pw pp ph

# 6. 验证
bin/pw.x --version    # 应显示 PWSCF v.7.5

# 7. Python 环境
conda create -n tritondft python=3.10 -y
conda activate tritondft
pip install -r requirements.txt
conda install -y -c conda-forge openmpi

# 8. 创建 config/config.yaml（填入绝对路径）
