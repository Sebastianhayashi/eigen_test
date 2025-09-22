# eigen3 对比测试

测试结果：

## 说明（测试简述以及环境）

对比 RVV 补丁测试

为了避免 binutils 太旧，汇编器不认 RVV 指令等问题，所以选用 gcc14：

```
sudo apt-get install gcc-14 g++-14
 
 spacemit-k1-x-MUSE-Pi-Pro-board% gcc-14 --version | head -n1
gcc-14 (Bianbu 14.2.0-4ubuntu2~24.04bb1) 14.2.0
spacemit-k1-x-MUSE-Pi-Pro-board% g++-14 --version | head -n1
g++-14 (Bianbu 14.2.0-4ubuntu2~24.04bb1) 14.2.0
spacemit-k1-x-MUSE-Pi-Pro-board%
```

## 源码获取

首先是 RVV 补丁组（下文称补丁组），首先获取补丁组的源码：

```
git clone https://gitlab.com/libeigen/eigen.git eigen-rvv
cd eigen-rvv

git fetch origin refs/merge-requests/1687/head:rvv-mr1687

git checkout rvv-mr1687
```

再获取官方组：

```
 git clone --depth 1 -b 3.4.0 https://gitlab.com/libeigen/eigen.git eigen-3.4.0
```



## 编译

1. 补丁组：

传入参数

```
export CC=gcc-14
export CXX=g++-14
export RVV_FLAGS="-march=rv64gcv_zvl256b -mabi=lp64d -mrvv-vector-bits=zvl"
export OPT_FLAGS="-O3 -DNDEBUG -ffast-math -fno-math-errno -fno-trapping-math -funroll-loops"
export CFLAGS="${OPT_FLAGS} ${RVV_FLAGS}"
export CXXFLAGS="${OPT_FLAGS} ${RVV_FLAGS}"
```

配置：

```
cmake -S bench/btl -B build-rvv \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER="$CC" \
  -DCMAKE_CXX_COMPILER="$CXX" \
  -DCMAKE_C_FLAGS="$CFLAGS" \
  -DCMAKE_CXX_FLAGS="$CXXFLAGS" \
  -DEigen_SOURCE_DIR="$(pwd)"
```

查看可用目标：

```
cmake --build build-rvv --target help | sed -n '1,200p'

The following are some of the valid targets for this Makefile:
... all (the default if no target is provided)
... clean
... depend
... edit_cache
... rebuild_cache
... test
... copy_scripts
... btl_eigen3_adv
... btl_eigen3_linear
... btl_eigen3_matmat
... btl_eigen3_vecmat
... btl_openblas
... btl_tensor_linear
... btl_tensor_matmat
... btl_tensor_vecmat
... btl_ublas
... main
... regularize
... smooth
```

仅编译基础测试：

```
cmake --build build-rvv --target \
  btl_eigen3_linear \
  btl_eigen3_vecmat \
  btl_eigen3_matmat \
  btl_eigen3_adv \
  btl_tiny_eigen3 \
  btl_tiny_eigen3_novec \
  -j$(nproc)
```

运行测试：

```
./libs/eigen3/btl_eigen3_linear
./libs/eigen3/btl_eigen3_vecmat
./libs/eigen3/btl_eigen3_matmat
./libs/eigen3/btl_eigen3_adv
```

## 测试过程

## 测试对比