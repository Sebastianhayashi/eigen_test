# RISC-V RVV 下复现 Eigen3/BTL 基准（**已完成部分**）

> 本文仅覆盖**已成功复现**的环节：环境准备、源码获取、按两套源码分别构建 BTL 可执行、运行四组基准并生成 `bench_*.dat` 数据。**出图（gnuplot）尚未复现**，将在后续补充。

---

## 1. 目标与产物（本阶段）

* 在 RISC-V（RV64GCV）环境下，使用 **BTL（Bench Template Library）** 跑 **Eigen3** 基准，分别针对两份源码：

  * `eigen-official`（官方 3.4.0）
  * `eigen-rvv`（包含 RVV 补丁的 MR 分支）
* 产物：两份独立的 **`.dat` 数据文件**（`bench_*_eigen3.dat`），用于后续对比与出图。

---

## 2. 目录结构与命名

```bash
mkdir eigen-compare
cd eigen-compare

# 之后目录将变为：
# eigen-compare/
# ├─ eigen-official/   # 官方 3.4.0 源码
# └─ eigen-rvv/        # RVV MR 分支源码
```

---

## 3. 环境与依赖（系统级）

```bash
sudo apt-get update
sudo apt-get install -y git cmake gnuplot-nox gcc-14 g++-14
gnuplot --version
# 期望：gnuplot 6.0 patchlevel 0
```

> 说明：本阶段只是确认 `gnuplot` 存在（后续出图会用到），**本次复现未包含出图**。

---

## 4. 获取两份源码

### 4.1 官方 Eigen（3.4.0）

```bash
cd eigen-compare
git clone --depth 1 -b 3.4.0 https://gitlab.com/libeigen/eigen.git eigen-official
```

### 4.2 RVV 补丁分支（MR 1687）

```bash
git clone https://gitlab.com/libeigen/eigen.git eigen-rvv
cd eigen-rvv
git fetch origin refs/merge-requests/1687/head:rvv-mr1687
git checkout rvv-mr1687
cd ..
```

---

## 5. 统一工具链与编译选项

> 选择 **GCC 14** 是因为其对 **RVV 1.0** 与 `-mrvv-vector-bits=` 的支持更完备，避免老版本行为差异。

```bash
export CC=gcc-14
export CXX=g++-14

# 注意：以下两项需成对出现以固定 VL（向量长度）
export RVV_FLAGS="-march=rv64gcv_zvl256b -mabi=lp64d -mrvv-vector-bits=zvl"

export OPT_FLAGS="-O3 -DNDEBUG -ffast-math -fno-math-errno -fno-trapping-math -funroll-loops"
export CFLAGS="${OPT_FLAGS} ${RVV_FLAGS}"
export CXXFLAGS="${OPT_FLAGS} ${RVV_FLAGS}"
```

> 约定：固定 VL = 256 bit。

---

## 6. 构建 BTL 基准（官方 Eigen 3.4.0）

> 关键点：让 BTL 使用**当前仓库内的 Eigen 源**，因此 `-DEigen_SOURCE_DIR="$(pwd)"` 需要在对应源码根目录下执行。

```bash
cd eigen-official

cmake -S bench/btl -B build-official \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER="$CC" \
  -DCMAKE_CXX_COMPILER="$CXX" \
  -DCMAKE_C_FLAGS="$CFLAGS" \
  -DCMAKE_CXX_FLAGS="$CXXFLAGS" \
  -DEigen_SOURCE_DIR="$(pwd)"
```

**参考/期望输出片段：**

```
-- The C compiler identification is GNU 14.2.0
-- The CXX compiler identification is GNU 14.2.0
...
-- Found OPENBLAS: /usr/lib/riscv64-linux-gnu/libopenblas.so; -lpthread -lgfortran
-- Configuring done
-- Generating done
-- Build files have been written to: .../eigen-official/build-official
```

**编译四个可执行：**

```bash
cmake --build build-official --target \
  btl_eigen3_linear btl_eigen3_vecmat btl_eigen3_matmat btl_eigen3_adv -j"$(nproc)"
```

**参考/期望输出片段：**

```
[100%] Built target btl_eigen3_linear
[100%] Built target btl_eigen3_vecmat
[100%] Built target btl_eigen3_matmat
[100%] Built target btl_eigen3_adv
```

---

## 7. 构建 BTL 基准（RVV 补丁分支）

```bash
cd ../eigen-rvv

cmake -S bench/btl -B build-rvv \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER="$CC" \
  -DCMAKE_CXX_COMPILER="$CXX" \
  -DCMAKE_C_FLAGS="$CFLAGS" \
  -DCMAKE_CXX_FLAGS="$CXXFLAGS" \
  -DEigen_SOURCE_DIR="$(pwd)"

cmake --build build-rvv --target \
  btl_eigen3_linear btl_eigen3_vecmat btl_eigen3_matmat btl_eigen3_adv -j"$(nproc)"
```

> 期望也能顺利生成四个可执行文件，路径位于 `eigen-rvv/build-rvv/libs/eigen3/`。

---

## 8. 运行四组基准，生成 `.dat`（两份源码分别执行）

### 8.1 官方 Eigen 结果

```bash
cd ../eigen-official

./build-official/libs/eigen3/btl_eigen3_linear
./build-official/libs/eigen3/btl_eigen3_vecmat
./build-official/libs/eigen3/btl_eigen3_matmat
./build-official/libs/eigen3/btl_eigen3_adv

# 验证：
ls build-official/bench_*_eigen3.dat
```

### 8.2 RVV 分支结果

```bash
cd ../eigen-rvv

./build-rvv/libs/eigen3/btl_eigen3_linear
./build-rvv/libs/eigen3/btl_eigen3_vecmat
./build-rvv/libs/eigen3/btl_eigen3_matmat
./build-rvv/libs/eigen3/btl_eigen3_adv

# 验证：
ls build-rvv/bench_*_eigen3.dat
```

> 预期每次运行会在对应 `build-*` 目录下生成多份 `bench_*_eigen3.dat`，用于后续对比与出图。

---

## 9. 可执行与数据产物一览（速查）

| 源码份            | 可执行所在目录                                      | 运行后 `.dat` 目录                    | 核心可执行名                                                                             |
| -------------- | -------------------------------------------- | -------------------------------- | ---------------------------------------------------------------------------------- |
| eigen-official | `eigen-official/build-official/libs/eigen3/` | `eigen-official/build-official/` | `btl_eigen3_linear` / `btl_eigen3_vecmat` / `btl_eigen3_matmat` / `btl_eigen3_adv` |
| eigen-rvv      | `eigen-rvv/build-rvv/libs/eigen3/`           | `eigen-rvv/build-rvv/`           | 同上                                                                                 |

---

## 10. 常见问题与排查（本阶段遇到/规避）

* **BLAS/可选库提示未找到**
  说明：BTL 的部分后端可选依赖，非必需；构建四个 `btl_eigen3_*` 目标不受影响。

* **编译器/指令集不一致**
  说明：务必在同一 Shell 会话中导出 `CC/CXX/CFLAGS/CXXFLAGS`；两份源码保持一致，确保结果可比。

---

## 11. 清理与复跑（数据阶段）

```bash
# 仅清理数据/中间文件（保留构建产物）：
rm -f eigen-official/build-official/bench_*_eigen3.dat
rm -f eigen-rvv/build-rvv/bench_*_eigen3.dat

# 如需完全重来（含构建）：
rm -rf eigen-official/build-official eigen-rvv/build-rvv
# 然后重复第 6、7、8 节
```

---

## 12. 下一步计划（未完成事项占位）

* **出图（gnuplot）**：将两份 `.dat` 在各自目录下批量画图，生成 `*.jpg` 并进行结果对比（脚本兼容 `gnuplot 6.0` 的细节、`set clabel` 的处理等）。
* **对比分析**：按相同 workload 将 `eigen-official` 与 `eigen-rvv` 的结果并列展示与解读。

---

### 附：一次性跑通的最短命令清单（**已复现**）

> 从 **`eigen-compare/`** 目录开始：

```bash
# 1) 依赖
sudo apt-get update
sudo apt-get install -y git cmake gnuplot-nox gcc-14 g++-14

# 2) 取源码
git clone --depth 1 -b 3.4.0 https://gitlab.com/libeigen/eigen.git eigen-official
git clone https://gitlab.com/libeigen/eigen.git eigen-rvv
cd eigen-rvv && git fetch origin refs/merge-requests/1687/head:rvv-mr1687 && git checkout rvv-mr1687 && cd ..

# 3) 统一编译器与标志
export CC=gcc-14
export CXX=g++-14
export RVV_FLAGS="-march=rv64gcv_zvl256b -mabi=lp64d -mrvv-vector-bits=zvl"
export OPT_FLAGS="-O3 -DNDEBUG -ffast-math -fno-math-errno -fno-trapping-math -funroll-loops"
export CFLAGS="${OPT_FLAGS} ${RVV_FLAGS}"
export CXXFLAGS="${OPT_FLAGS} ${RVV_FLAGS}"

# 4) 构建 official
cd eigen-official
cmake -S bench/btl -B build-official \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER="$CC" -DCMAKE_CXX_COMPILER="$CXX" \
  -DCMAKE_C_FLAGS="$CFLAGS" -DCMAKE_CXX_FLAGS="$CXXFLAGS" -DEigen_SOURCE_DIR="$(pwd)"
cmake --build build-official --target btl_eigen3_linear btl_eigen3_vecmat btl_eigen3_matmat btl_eigen3_adv -j"$(nproc)"
./build-official/libs/eigen3/btl_eigen3_linear
./build-official/libs/eigen3/btl_eigen3_vecmat
./build-official/libs/eigen3/btl_eigen3_matmat
./build-official/libs/eigen3/btl_eigen3_adv
cd ..

# 5) 构建 rvv
cd eigen-rvv
cmake -S bench/btl -B build-rvv \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER="$CC" -DCMAKE_CXX_COMPILER="$CXX" \
  -DCMAKE_C_FLAGS="$CFLAGS" -DCMAKE_CXX_FLAGS="$CXXFLAGS" -DEigen_SOURCE_DIR="$(pwd)"
cmake --build build-rvv --target btl_eigen3_linear btl_eigen3_vecmat btl_eigen3_matmat btl_eigen3_adv -j"$(nproc)"
./build-rvv/libs/eigen3/btl_eigen3_linear
./build-rvv/libs/eigen3/btl_eigen3_vecmat
./build-rvv/libs/eigen3/btl_eigen3_matmat
./build-rvv/libs/eigen3/btl_eigen3_adv
```


