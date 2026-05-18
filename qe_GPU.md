Quantum ESPRESSO GPU 版安装、集成与使用报告

本文档记录了在 Ubuntu 22.04（无 sudo 权限）环境下为 TritonDFT 配置 GPU 版 QE 的完整流程：容器运行时、nvidia-container-toolkit rootless 化、NVHPC 编译 QE v7.5CN、写包装脚本接入 agent、跑通端到端 benchmark。

已有的 CPU 版流程见 doc/reports/report_qe_setup.md，本文件只补 GPU 增量。

---

1. 目标与约束

- 目标：让 TritonDFT agent 的 benchmark 在本地 GPU（2× H100）上跑 QE，取代原 CPU 版。
- 硬件：2× NVIDIA H100 80GB HBM3（sm_90），driver 570.195.03，CUDA 12.8 toolkit。
- 约束：无 sudo；不能装系统级 docker/nvidia-docker；不能装系统级 NVHPC SDK；所有东西都必须跑在用户态。

最终成果路径：

角色
路径
CPU QE（v7.5CN，已有）
TritonDFT/tritonDFT-src/QuantumE/bin/pw.x
| GPU QE（v7.5CN，新） | `TritonDFT/tritonDFT-src/QuantumE-gpu/bin/pw.x` |
| Agent 接入 wrapper | `TritonDFT/tritonDFT-src/QuantumE-gpu-wrapper/pw.x`（符号链接到 `qe-gpu-wrap.sh`）|
| Rootless podman 配置 | ~/.config/containers/oci/hooks.d/oci-nvidia-hook.json、~/.config/nvidia-container-runtime/config.toml |

---

2. 关键决策

2.1 容器运行时：podman 3.4 rootless

选项
状态
原因
Docker
❌
需 docker 组或 sudo，用户不在组中
rootless Docker
❌
未配置，安装流程较重
Singularity / Apptainer
❌
未安装
| podman 3.4.4 | ✅ | 已装，rootless 默认开启 |

2.2 两条 GPU QE 路线：先 NGC 验证、再 NVHPC 编译

路线
镜像 / 工具链
版本
用途
| NGC 容器（探路） | `nvcr.io/hpc/quantum_espresso:qe-7.3.1` | qe-7.3.1 | 几分钟就绪，验证加速比是否值得投入 |
| NVHPC 本地编译（生产） | `nvcr.io/nvidia/nvhpc:24.7-devel-cuda_multi-ubuntu22.04` | v7.5CN (与 CPU 同 commit) | 版本对齐，benchmark 数值可复现 |

先跑 NGC 确认加速比（16-atom Si SCF 得到 7.5×），再花 20 分钟编译版本对齐版。不走 NVHPC 主机安装，而是用 NVHPC devel 容器作纯粹的编译环境，不污染宿主。

2.3 Agent 接入：shell wrapper（方案 A）

QE GPU 版不能裸跑（容器内 openmpi hardcode 路径），必须走 podman run ... mpirun -np N pw.x。两种集成方式：

- 方案 A · Wrapper 脚本（采用）：`qe-gpu-wrap.sh` 伪装成 `pw.x`，内部封 `podman`。agent config 只改 `qe_bin_dir`，代码零改动。
- 方案 B · 改 executor：给 src/executor.py 加一个 GPU 后端分支——侵入代码，回归成本高。
---

3. 环境准备

3.1 podman rootless GPU passthrough

podman 3.4 不支持 `--device nvidia.com/gpu=all` 语法（需 4.1+ 的 CDI）。rootless 下须走 OCI prestart hook 方案。

第一步：写用户级 nvidia-container-runtime 配置（默认系统配置要 root 权限）

mkdir -p ~/.config/nvidia-container-runtime
cat > ~/.config/nvidia-container-runtime/config.toml <<'EOF'
disable-require = false

[nvidia-container-cli]
environment = []
ldconfig = "@/sbin/ldconfig.real"
load-kmods = false        # ← rootless 必须 false（加载内核模块需 root）
no-cgroups = true         # ← rootless 必须 true（cgroup 管理需 root）

[nvidia-container-runtime]
log-level = "info"
mode = "auto"
runtimes = ["runc", "crun"]
EOF

第二步：注册 OCI prestart hook

mkdir -p ~/.config/containers/oci/hooks.d
cat > ~/.config/containers/oci/hooks.d/oci-nvidia-hook.json <<EOF
{
    "version": "1.0.0",
    "hook": {
        "path": "/usr/bin/nvidia-container-runtime-hook",
        "args": ["nvidia-container-runtime-hook", "prestart"],
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "XDG_CONFIG_HOME=$HOME/.config"
        ]
    },
    "when": {"always": true, "commands": [".*"]},
    "stages": ["prestart"]
}
EOF

XDG_CONFIG_HOME 注入是关键——hook 需要这个 env 才能找到第一步的用户级 config。

第三步：验证

podman run --rm --hooks-dir=$HOME/.config/containers/oci/hooks.d \
  nvcr.io/hpc/quantum_espresso:qe-7.3.1 nvidia-smi

能看到 2× H100 的输出即成功。

3.2 拉镜像

# 编译环境（用于 build QE）
podman pull nvcr.io/nvidia/nvhpc:24.7-devel-cuda_multi-ubuntu22.04
# NGC 成品镜像（用于快速对照 baseline）
podman pull nvcr.io/hpc/quantum_espresso:qe-7.3.1

---

4. 编译 QE v7.5CN GPU 版

4.1 源码准备

cd TritonDFT/tritonDFT-src
git clone https://github.com/QEF/q-e.git QuantumE-gpu
cd QuantumE-gpu
# 与 CPU 版 (QuantumE/) 同一 commit，保证版本对齐
git checkout ddbec46536cc9c6f8cf3cd9db486573bcc5424a4
# QE 7.5 的 CUDA 支持必须 init submodule
git submodule update --init external/devxlib external/fox

4.2 Configure（在 NVHPC devel 容器内）

podman run --rm \
  -v $(pwd):/qe -w /qe \
  nvcr.io/nvidia/nvhpc:24.7-devel-cuda_multi-ubuntu22.04 \
  bash -c "
    export PATH=/opt/nvidia/hpc_sdk/Linux_x86_64/24.7/compilers/bin:\$PATH
    export PATH=/opt/nvidia/hpc_sdk/Linux_x86_64/24.7/comm_libs/mpi/bin:\$PATH
    ./configure --enable-parallel --enable-openmp \
                --with-cuda=/opt/nvidia/hpc_sdk/Linux_x86_64/24.7/cuda \
                --with-cuda-cc=90 --with-cuda-runtime=12.5 \
                --with-scalapack=no \
                F90=nvfortran F77=nvfortran CC=nvc FC=mpif90
  "

验证生成的 make.inc 包含：

DFLAGS         =  -D__PGI -D__CUDA -D__FFTW -D__MPI -D__MPI_MODULE
GPU_ARCH=90
CUDA_RUNTIME=12.5
CUDA_CFLAGS= -gpu=cc90,cuda12.5 -acc

4.3 Build

podman run --rm \
  -v $(pwd):/qe -w /qe \
  nvcr.io/nvidia/nvhpc:24.7-devel-cuda_multi-ubuntu22.04 \
  bash -c "
    export PATH=/opt/nvidia/hpc_sdk/Linux_x86_64/24.7/compilers/bin:\$PATH
    export PATH=/opt/nvidia/hpc_sdk/Linux_x86_64/24.7/comm_libs/mpi/bin:\$PATH
    make -j16 pw
  "

实测约 20 分钟完成，一次通过，无需手动修补 make.inc。链接选项含 -cudalib=cufft,cublas,cusolver,curand，符合预期。

---

5. 单独运行 GPU pw.x

注意：自编的 pw.x 不能裸跑——容器内 openmpi (hpcx 版本) 的辅助文件路径硬编码到 `/proj/nv/libraries/...`，直接 `./pw.x` 会 MPI_Init 失败。必须通过 `mpirun --allow-run-as-root -np N pw.x` 启动，且必须在容器内运行。

典型一次性调用模板：

podman run --rm \
  --hooks-dir=$HOME/.config/containers/oci/hooks.d \
  -v /home/weilin/workspace/TritonDFT/tritonDFT-src:/src \
  -v /path/to/workdir:/work -w /work \
  -e CUDA_VISIBLE_DEVICES=0 \
  nvcr.io/nvidia/nvhpc:24.7-devel-cuda_multi-ubuntu22.04 \
  bash -c "
    export PATH=/opt/nvidia/hpc_sdk/Linux_x86_64/24.7/comm_libs/mpi/bin:\$PATH
    export LD_LIBRARY_PATH=/opt/nvidia/hpc_sdk/Linux_x86_64/24.7/compilers/lib:\$LD_LIBRARY_PATH
    mpirun --allow-run-as-root -np 1 /src/QuantumE-gpu/bin/pw.x -in input.in
  "

启动后 QE banner 应出现：

Program PWSCF v.7.5 starts on ...
GPU acceleration is ACTIVE.  1 visible GPUs per MPI rank

5.1 基准对比（16-atom Si SCF, ecutwfc=60, k=4×4×4）

后端
WALL 时间
能量 (Ry)
加速比
GPU 峰值
CPU（自编 QuantumE/，8 MPI）
929.26s
-133.07683345
1×
—
NGC qe-7.3.1 (1× H100)
123.69s
-133.07683390
7.5×
94%
| v7.5CN GPU (1× H100) | 121.73s | -133.07683345 | 7.6× | 95% |

v7.5CN 与 CPU 能量精确到 10 位小数，完全复现。

---

6. 接入 TritonDFT Agent

6.1 问题：agent 的 executor 假设`pw.x` 是本地二进制

src/executor.py → src/execute_code/mpi_run.py 里写死了命令模板：


mpirun --allow-run-as-root -np {parallel_np} {exec_path} -in {input} | tee {output}

`exec_path = os.path.join(qe_bin_dir, 'pw.x')` 来自 `config/config.yaml`。如果让 exec_path 是一个 shell 脚本（wrapper），`bash -lc` 能直接运行它，只要它对外行为与 pw.x 一致。
6.2 Wrapper 实现

TritonDFT/tritonDFT-src/QuantumE-gpu-wrapper/qe-gpu-wrap.sh：

#!/usr/bin/env bash
set -euo pipefail

BIN_NAME="$(basename "$0")"     # via symlink: pw.x / pp.x / dos.x / ...
WORKSPACE_ROOT="${QE_GPU_WORKSPACE_ROOT:-/home/weilin/workspace}"
QE_GPU_BIN_DIR="${QE_GPU_BIN_DIR:-$WORKSPACE_ROOT/TritonDFT/tritonDFT-src/QuantumE-gpu/bin}"
IMAGE="${QE_GPU_IMAGE:-nvcr.io/nvidia/nvhpc:24.7-devel-cuda_multi-ubuntu22.04}"
HOOKS_DIR="${QE_GPU_HOOKS_DIR:-$HOME/.config/containers/oci/hooks.d}"
GPU_ID="${CUDA_VISIBLE_DEVICES:-0}"
NP="${QE_GPU_NP:-1}"

CWD="$(pwd)"
ARGS_Q=""; for arg in "$@"; do ARGS_Q+=" $(printf '%q' "$arg")"; done

exec podman run --rm \
  --hooks-dir="$HOOKS_DIR" \
  -v "$WORKSPACE_ROOT:$WORKSPACE_ROOT" \
  -w "$CWD" \
  -e "CUDA_VISIBLE_DEVICES=$GPU_ID" \
  "$IMAGE" \
  bash -c "
    export PATH=/opt/nvidia/hpc_sdk/Linux_x86_64/24.7/comm_libs/mpi/bin:\$PATH
    export LD_LIBRARY_PATH=/opt/nvidia/hpc_sdk/Linux_x86_64/24.7/compilers/lib:\$LD_LIBRARY_PATH
    exec mpirun --allow-run-as-root -np $NP \"$QE_GPU_BIN_DIR/$BIN_NAME\" $ARGS_Q
  "

同目录下创建符号链接让多种 QE 二进制都走同一 wrapper：

for bin in pw.x pp.x dos.x bands.x projwfc.x ph.x; do
  ln -sf qe-gpu-wrap.sh "$bin"
done

关键设计：
- BIN_NAME=$(basename "$0") 根据被调用的符号链接名选真实二进制
- WORKSPACE_ROOT 整个挂载进容器，绝对路径原样工作，免去路径重映射
- -w "$CWD" 把 agent 的 <work_dir>/<date>/<run_id>/ 设为容器工作目录
- printf '%q' 保留参数 quoting，允许含空格的路径
- 通过 env 变量（QE_GPU_NP、QE_GPU_BIN_DIR、QE_GPU_IMAGE）可运行时覆盖，不必改脚本
6.3 config.yaml 改动

qe_bin_dir: /home/weilin/workspace/TritonDFT/tritonDFT-src/QuantumE-gpu-wrapper
# CPU 版（切回时改）：
# qe_bin_dir: /home/weilin/workspace/TritonDFT/tritonDFT-src/QuantumE/bin

6.4 调用约束

- agent 的外层 `mpirun --allow-run-as-root -np N wrapper.sh` 会启 N 个 wrapper → N 个 podman → 每个 podman 内部再跑 `-np N` 的 mpirun。这样嵌套会浪费。
- 正确用法：agent 侧 --parallel-np 1，通过 QE_GPU_NP 控制容器内真实并行度。
- 单 GPU 单进程通常够用；多 GPU 扩展通过外层改 CUDA_VISIBLE_DEVICES 或写 GPU 分配器（未实现）。
6.5 端到端验证

conda activate tritondft
export ANTHROPIC_API_KEY=<your key>
cd TritonDFT/tritonDFT-src
python test_benchmark/benchmark_agent_test.py \
  --model claude-haiku-4-5-20251001 --backend claude \
  --task-type vc_relax --category semiconductor \
  --evaluation-mode --limit 1 \
  --run-mode mpirun --parallel-np 1 \
  --work-dir tmp_bench_gpu

实测：Si vc-relax 端到端 19.6s，agent 推断 + 3 次 BFGS 步 + 评分全部跑完，QE 输出含 GPU acceleration is ACTIVE.  1 visible GPUs per MPI rank，晶格常数 5.43 Å（符合 Si 参考值）。

---

7. 踩坑汇总（GPU 专属）

#
问题
症状
解决方案
1
Docker 无 sudo 不可用
permission denied … docker.sock
改用 podman rootless
2
podman 3.4 无 CDI 支持
--device nvidia.com/gpu=all 报 no such file
改用 OCI prestart hook（--hooks-dir）
3
默认 nvidia-container-toolkit 配置需 root
Hook 执行 exit code 1
在 ~/.config/nvidia-container-runtime/config.toml 覆盖 load-kmods=false、no-cgroups=true
4
Hook 找不到用户配置
同上 exit 1
hook JSON env 注入 XDG_CONFIG_HOME=$HOME/.config
5
NGC quantum_espresso 镜像不含 nvfortran
想用它编译新版 QE 发现无编译器
换 nvcr.io/nvidia/nvhpc:24.7-devel-cuda_multi-ubuntu22.04
6
QE submodule 未自动拉取
configure 通过但 make 找不到 devxlib
git submodule update --init external/devxlib external/fox
7
自编 GPU pw.x 裸跑 MPI_Init 失败
/proj/nv/libraries/... No such file
必须走 mpirun --allow-run-as-root -np N pw.x，且必须在容器内
8
Si 2-atom GPU 反慢
7.11s vs CPU 4.62s
小体系 GPU 启动开销主导，用 ≥ 16 atom benchmark
9
宿主 mpirun 缺失（非交互 shell）
wrapper 调用 mpirun: command not found
运行前 conda activate tritondft（env 内有 openmpi），env 透传给 bash -lc 子进程
10
第一次 sed 修 pseudo_dir 破坏 quoting
QE 报 /root/espresso/pseudo/si.upf not found
sed 规则带上开头引号：sed "s|'../PseudoDojo|'/abs/path"
11
CDI spec 生成无效果
nvidia-ctk cdi generate 生成了但 podman 3.4 不识别
搁置 CDI，用 OCI hook

---

8. 性能与集成结果

Si 16-atom SCF（ecutwfc=60, k=4×4×4）

- CPU 8 MPI：929s → GPU 1× H100 (v7.5CN)：121.7s，7.6× 加速
- 能量精确到 10 位小数
- GPU 利用率 p90=90%，峰值 95%
端到端 benchmark（Si vc_relax，`--limit 1`）

- Agent 总耗时 19.6s（含 LLM 推断 + QE 执行 + 结果评估）
- QE WALL 1.75s（Si 太小所以 GPU util 短暂）
- Wrapper 透明接入，config 只改一行
尚未做的事

- 多 GPU 支持：当前单卡；多卡需要 agent pool 级的 GPU 分配器（`todo/qe-gpu-integrate.md` 后续子任务）。
- **`make pw pp ph` 扩展**：只编了 `pw.x`；`pp.x`/`ph.x` 在跑完整 benchmark（band_gap / dos）时需要再补 `make -j16 pw pp ph`。
- 启动开销摊销：小任务 podman 启动 1–2s 的开销占比显著。长期可考虑常驻容器 + exec 复用（`podman container`）。
- 大规模 benchmark 复测：仅跑了 `--limit 1`；下一步用 `--limit 10` 或跑整类别以量化整体加速。

---

9. 快速复现清单

# 一次性环境配置（rootless GPU）
mkdir -p ~/.config/nvidia-container-runtime ~/.config/containers/oci/hooks.d
# 写入 section 3.1 的两个 config 文件

# 拉镜像
podman pull nvcr.io/nvidia/nvhpc:24.7-devel-cuda_multi-ubuntu22.04
podman pull nvcr.io/hpc/quantum_espresso:qe-7.3.1    # 可选，探路用

# 编译 GPU QE
cd TritonDFT/tritonDFT-src
git clone https://github.com/QEF/q-e.git QuantumE-gpu
cd QuantumE-gpu
git checkout ddbec46536cc9c6f8cf3cd9db486573bcc5424a4
git submodule update --init external/devxlib external/fox
# configure + make 见 section 4

# Wrapper 接入
# QuantumE-gpu-wrapper/qe-gpu-wrap.sh 已签入；其余 pw.x 等符号链接：
cd QuantumE-gpu-wrapper
for bin in pw.x pp.x dos.x bands.x projwfc.x ph.x; do ln -sf qe-gpu-wrap.sh "$bin"; done

# Config 指向 wrapper
sed -i 's|QuantumE/bin|QuantumE-gpu-wrapper|' config/config.yaml

# 跑 benchmark
conda activate tritondft
export ANTHROPIC_API_KEY=<key>
python test_benchmark/benchmark_agent_test.py \
  --model claude-haiku-4-5-20251001 --backend claude \
  --evaluation-mode --limit 1 --parallel-np 1

---

10. 参考

- CPU 版安装报告：doc/reports/report_qe_setup.md（section 12 追加了 GPU 踩坑摘要）
- 编译执行记录：progress/logs/qe-gpu-rebuild.md
- 支线探查记录：progress/logs/qe-gpu-check.md
- 后续优化 todo：todo/qe-gpu-integrate.md
- QE commit：ddbec46536cc9c6f8cf3cd9db486573bcc5424a4（v7.5CN）
- NVHPC 版本：24.7；nvfortran 24.7-0；CUDA 12.5（容器内）/ 12.8（宿主 driver）