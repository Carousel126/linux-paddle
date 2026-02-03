# Ubuntu 下编译 paddle 踩坑纪录
编译paddle的一些过程/踩坑/环境配置记录  
总体过程参考https://www.paddlepaddle.org.cn/documentation/docs/zh/develop/install/compile/linux-compile-by-make.html

## 环境
Linux ubuntu 20.04   
硬件  NVIDIA 3090    
Docker  28.2.1  
cuda 11.8  cudnn 8.9  
python3.10

p.s: 目前docker已经支持gpu虚拟化，因此nvidia-docker已经过时，网上提到的nvidia-docker都无用   
p.s： 题主曾使用过macbook M1进行本地编译 发现在编译中会出现很多错误且难以解决 不太建议使用

## 检查环境
```bash
# 检查linux环境
nvidia-smi | grep -E "CUDA Version|Driver Version"
nvcc -V
```
CUDA Toolkit和Paddle编译时选用的CUDA要匹配  
保证各工具/依赖的版本兼容、源码完整性、硬件与编译配置匹配   

如果环境不匹配的话 在最后一步单元测试的时候会找不到GPU  
但这个问题题主目前也没搞清楚为什么 

## 开始使用Docker编译  
#### 1. 请首先选择您希望储存 PaddlePaddle 的路径，然后在该路径下使用以下命令将 PaddlePaddle 的源码从 github 克隆到本地当前目录下名为 Paddle 的文件夹中：  
```bash
git clone --recursive https://github.com/PaddlePaddle/Paddle.git  
```
##### 1.科学上网
git clone 非常巨大，有10G，需要科学上网以免各种卡顿，下载不全 
##### 2.如果因为网络问题中断
如果克隆 PaddlePaddle时卡住   
手动中断后（^C）   
重新运行上述代码会出现错误：fatal: destination path 'Paddle' already exists and is not an empty directory.   
主要是因为 Paddle目录已创建但子模块拉取不完整，再次执行克隆命令就会报 “目录已存在且非空” 的错误。   
核心解决思路是复用已创建的 Paddle 目录，补全未拉取完成的子模块：   
步骤 1：进入已创建的 Paddle 目录   
```bash
cd ~/baidu/Paddle  
```
步骤 2：补全并更新所有子模块  
初始化所有未初始化的子模块，拉取完整的子模块代码，同时修复之前中断导致的checkout failed问题：  
```bash
运行
git submodule update --init --recursive
```
#### 2. 进入 Paddle 目录下：  
```bash
cd Paddle
```  
#### 3. 拉取 PaddlePaddle 镜像 
```bash
docker pull ccr-2vdh3abv-pub.cnc.bj.baidubce.com/paddlepaddle/paddle:cuda118-dev
```
#### 4. 创建并进入已配置好编译环境的 Docker 容器：   
```bash
docker run --gpus all --name paddle_docker -v $PWD:/paddle --network=host -it ccr-2vdh3abv-pub.cnc.bj.baidubce.com/paddlepaddle/paddle:3.3.0-gpu-cuda11.8-cudnn8.9 /bin/bash  
```
#### 5. 进入 Docker 后进入 paddle 目录下：
```bash
cd /Paddle
```
遗漏这一步可能导致路径变成paddle/paddle 与参考文档不同  
#### 6. 切换到 develop 版本进行编译：
```bash
git checkout develop
```
注意：目前paddle 支持 Python 3.9 以上版本  

#### 7. 创建并进入/paddle/build 路径下：
```bash
mkdir -p /paddle/build && cd /paddle/build
```
#### 8. 使用以下命令安装相关依赖：
安装编译依赖
```bash
pip3.10 install -r /paddle/python/requirements.txt
```
#### 9. 执行 cmake：
```bash
cmake .. -DPY_VERSION=3.10 -DWITH_GPU=ON -DWITH_DISTRIBUTE=ON
```
#### 10. 执行编译：
```bash
make -j$(nproc)
#如果需要计算编译时间
time make -j$(nproc)
```
编译途中报错：  
如果运行上述代码仅显示error2  
可以运行  
```bash
make -j1
```
运行速度较慢 但可以最终输出具体错误原因 再进行处理即可  
以下问题出现在编译途中
<img width="1908" height="542" alt="9663c284127229a53267c6ee5825284c" src="https://github.com/user-attachments/assets/aec332f0-f282-4fa4-8b39-196718859b93" />
需要修改编译代码重新开始   
修复： 
由于 NVIDIA 不同显卡有专属的计算能力版本，CUDA编译时必须指定与显卡匹配的计算能力，否则会出现编译失败、运行时GPU报错、显卡无法被调用等问题；  
RTX3090 属于 NVIDIA Ampere（安培）架构，对应的 CUDA 计算能力是8.6（简称 sm_86/compute_86）   
PaddlePaddle 源码的默认 cmake 配置中，硬编码了低版本的 CUDA 计算能力为100，如果直接编译，CUDA 会尝试为sm_100这个旧架构编译代码，而 RTX3090（sm_86）完全不支持该架构，最终导致 Paddle 编译失败，或编译后无法在 3090 上运行   
  
所以需要将文件内容中为computer100的部分修改成为86，使得环境与文档匹配  
下列代码将文件进行备份后   
修改主要配置与第三方库中的代码并删除build中的构建缓存  
在修复完成后进行重新编译  
即可解决以上报错
  
```bash
cat > fix_for_rtx3090.sh << 'EOF'
#!/bin/bash

echo "=== 修复 RTX 3090 (计算能力 8.6) 的构建问题 ==="

# 进入目录
cd /paddle

# 备份重要文件
cp cmake/cuda.cmake cmake/cuda.cmake.backup
cp .gitmodules .gitmodules.backup

# 修复主配置
echo "修复 cmake/cuda.cmake..."
sed -i 's/set(CUDA_ARCH "100")/set(CUDA_ARCH "86")/g' cmake/cuda.cmake
sed -i 's/compute_100,code=sm_100/compute_86,code=sm_86/g' cmake/cuda.cmake
sed -i 's/"100"/"86"/g' cmake/cuda.cmake

# 修复第三方库
echo "修复第三方库配置..."
for dir in warpctc warprnnt; do
    if [ -f "python/third_party/$dir/CMakeLists.txt" ]; then
        sed -i 's/compute_100/compute_86/g' python/third_party/$dir/CMakeLists.txt
        sed -i 's/sm_100/sm_86/g' python/third_party/$dir/CMakeLists.txt
        sed -i 's/-arch=compute_100/-arch=compute_86/g' python/third_party/$dir/CMakeLists.txt
        echo "已修复 $dir"
    fi
done

# 清理构建缓存
echo "清理构建缓存..."
rm -rf build
rm -rf python/third_party/*/src/extern_*-build

# 重新配置
echo "重新配置..."
mkdir build && cd build
cmake .. \
  -DCUDA_ARCH_NAME=Ampere \
  -DCUDA_ARCH_BIN="86" \
  -DCUDA_ARCH_PTX="86" \
  -DCMAKE_BUILD_TYPE=Release \
  -DWITH_GPU=ON \
  -DWITH_TESTING=OFF \
  -DWITH_DISTRIBUTE=OFF

echo "开始构建..."
make -j$(nproc) 2>&1 | tee build_rtx3090.log

echo "=== 完成 ==="
EOF
```
之后即可重新运行（此处注意自己的目录是否与代码中相同&如果计算时间在make前加time）
    

编译结束后，可以检查一下自己的paddle版本  
```bash
# 容器内执行，先回到Paddle源码根目录
cd /paddle
# 执行官方版本查询脚本
python tools/version.py
```
题主本人在编译后发现自己版本为0.15.XX  
可能是由于在前面debug时重新开始时 漏写了部分代码  
导致存在版本问题   
如果发现此类问题   
可以重新拉取3.3/其他想要的版本进行cmake与make重新编译   

#### 11.初次编译/二次编译
此处参考文档为 https://github.com/PaddlePaddle/FastDeploy/issues/6225
初次编译时间相当于二次编译速度更快，因为有编译缓存的存在，时间会较短  
  
修改底层的头文件：paddle/fluid/platform/enforce.h   
  
修改Op的cc文件：paddle/fluid/operators/rank_loss_op.cc  
这个文档不知道为何在 paddle 3.3 中没有出现 可能在版本迭代中，对算子做了移除、重命名或架构迁移。  
题主在该文件夹下找了一个其他的文件进行二次编译   
  
修改python文件：python/paddle/tensor/math.py   
二次编译方式：对应文件加一个空行/空格保存退出后，然后执行编译命令time make -j$(nproc)，二次编译需要执行cmake  

#### 12. 编译成功后进入/paddle/build/python/dist目录下找到生成的.whl包：
```bash
cd /paddle/build/python/dist
```

#### 13. 在当前机器或目标机器安装编译好的.whl包：
```bash
pip3.10 install -U [whl 包的名字]
```

#### 14. 运行单元测试
不同的编译选项，能编译出不同的功能，对应的编译时间也各不相同。可以参考编译选项表，尝试打开WITH_TESTING=ON编译出单元测试，并正确运行一个单测。
```bash
# 重新运行cmake命令：
cmake .. -DPY_VERSION=3.10 -DWITH_GPU=ON -DWITH_TESTING=ON
（在原来的cmake命令后加入-DWITH_TESTING=ON）
# 执行编译命令
make -j$(nproc)
# 安装第三方库
pip3.10 install -r ../python/requirements.txt
# cd
cd /paddle/build
#运行logsumexp的单测
ctest -R test_logsumexp运行logsumexp的单测。
```
运行单测的时候可能出现  
一般来讲应该是各个模块之间不匹配造成的  
但题主全部检查了一遍发现都是匹配的但依旧报这个错误  
最后只能重新编译进行尝试  
<img width="1404" height="504" alt="d20dc473052630dec134cff001d67625" src="https://github.com/user-attachments/assets/90863fa2-8495-4a8a-899f-dd798160b407" />


#### 15. 一些其他东西
##### 1.如果出现
```bash
ValueError: (InvalidArgument) use wrong place, Please check. (at /paddle/Paddle/paddle/fluid/pybind/place.cc:427)  
```
然后在nvidia-smi时找不到gpu
可以参考https://dev-solve.com/zh/posts/458de4c  
但是它其中有一步写错了  
<img width="738" height="281" alt="d5fee88d8dced7460a1f567c0c48614e" src="https://github.com/user-attachments/assets/20a977df-4e1f-4b67-91d9-e53ea147a03e" />   
参照外文文档中     
应该是把no-cgroups=true改成no-cgroups=false     
但中文文档写错了……   
##### 2. 退出docker后重新启动
退出docker后，容器自动会自动stop, 这时需要
```bash
docker ps -a  #列出所有容器后找到docker
docker start <docker_name>
docker attach <docker_name>
```
##### 3.报错显示路径找不到
当你的编译报错说XX路径找不到时可以   
创造软链接    
或者使用rm对文件进行移动    













