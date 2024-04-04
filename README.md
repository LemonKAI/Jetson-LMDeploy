
## Setup
since the lmdeploy is not for jetson, it need to install jetson device wheel and CUDA version, CMAKE version and lmdeploy version.

### Install Miniconda 

``` bash
wget https://github.com/conda-forge/miniforge/releases/lastest/download/Miniforge3-Linux-aarch64.sh
bash Miniforge3-Linux-aarch64.sh
```

### Setup conda

```bash
conda create -n lmdeploy python=3.8
conda activate lmdeploy
```

### Install Pytorch for Jetson Orin Nano
Requirement
> - CUDA 11.8
> - Pytorch 2.1.0
> - JetPack >=5.1


#### Update to CUDA 11.8

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/arm64/cuda-ubuntu2004.pin
sudo mv cuda-ubuntu2004.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/11.8.0/local_installers/cuda-tegra-repo-ubuntu2004-11-8-local_11.8.0-1_arm64.deb
sudo dpkg -i cuda-tegra-repo-ubuntu2004-11-8-local_11.8.0-1_arm64.deb
sudo cp /var/cuda-tegra-repo-ubuntu2004-11-8-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get -y install cuda
```


src: [CUDA Toolkit 11.8 Downloads](https://developer.nvidia.com/cuda-11-8-0-download-archive?target_os=Linux&target_arch=aarch64-jetson&Compilation=Native&Distribution=Ubuntu&target_version=20.04&target_type=deb_local)

#### PyTorch for Jetson

- Installation

Jetpacket 5.12 - Python3.8 > [https://developer.download.nvidia.cn/compute/redist/jp/v512/pytorch/torch-2.1.0a0+41361538.nv23.06-cp38-cp38-linux_aarch64.whl](https://developer.download.nvidia.cn/compute/redist/jp/v512/pytorch/torch-2.1.0a0+41361538.nv23.06-cp38-cp38-linux_aarch64.whl)
```bash
sudo apt-get install python3-pip libopenblas-base libopenmpi-dev libomp-dev
pip install torch-2.1.0a0+41361538.nv23.06-cp38-cp38-linux_aarch64.whl
```

### Installation for rapidjosn

```bash
git clone https://github.com/Tencent/rapidjson.git

cd rapidjson

mkdir build && cd build
cmake .. \
    -DRAPIDJSON_BUILD_DOC=OFF \
    -DRAPIDJSON_BUILD_EXAMPLES=OFF \
    -DRAPIDJSON_BUILD_TESTS=OFF
make -j4

sudo make install
```

### Installation CMake Version
```bash
cd ~
wget https://github.com/Kitware/CMake/releases/download/v3.29.0-rc1/cmake-3.29.0-rc1-linux-aarch64.tar.gz
tar xf cmake-3.29.0-rc1-linux-aarch64.tar.gz && rm cmake-3.29.0-rc1-linux-aarch64.tar.gz

# rename the folder
mv cmake-3.29.0-rc1-linux-aarch64 cmake-3.29.0
cd cmake-3.29.0

# verify version
./bin/cmake --version

# set the path variable
export PATH=/home/{hostname}/cmake-3.29.0/bin:$PATH

# Verify again
cmake --version
```

### Install LMDeploy-0.2.5

```bash 
cd ~
git clone https://github.com/InternLM/lmdeploy.git
cd lmdeploy
git checkout c5f4014
```

Under ~/lmdeploy create a generation_jetson.sh

```bash
#!/bin/sh
builder="-G Ninja"

if [ "$1" == "make" ]; then
    builder=""
fi

cmake ${builder} .. \
    -DCMAKE_BUILD_TYPE=RelWithDebInfo \
    -DCMAKE_EXPORT_COMPILE_COMMANDS=1 \
    -DCMAKE_INSTALL_PREFIX=./install \
    -DBUILD_PY_FFI=ON \
    -DBUILD_MULTI_GPU=OFF \
    -DCMAKE_CUDA_FLAGS="-lineinfo" \
    -DUSE_NVTX=ON
```

Then using following command:

```bash

chmod +x generate_jetson.sh

sudo apt-get install ninja-build

mkdir build && cd build

../generate_jetson.sh

ninja install
```

Comment some dependency in the file requirements/runtime.txt

```txt
# torch<=2.1.2,>=2.0.0
# triton>=2.1.0,<=2.2.0
```
Install lmdeploy

```bash
cd ~/lmdeploy

pip install -e .[serve]
```

## Quantization Model

### AWQ Quantization
```bash 
export HF_MODEL=./path/to/hf-model
export WORK_DIR=./paht/to/hf-model-4bit

lmdeploy lite auto_awq \
   $HF_MODEL \
  --calib-dataset 'ptb' \
  --calib-samples 128 \
  --calib-seqlen 2048 \
  --w-bits 4 \
  --w-group-size 128 \
  --work-dir $WORK_DIR
```

### Turbomind Model
```bash
export WORK_DIR=./path/to/hf-model-4bit
export TM_DIR=./path/to/hf-model-turbomind

lmdeploy convert  model-type \
    $WORK_DIR \
    --model-format awq \
    --group-size 128 \
    --dst-path $TM_DIR
```

## Evaluation

### webui-genration

#### base-line
|Device | llama2-chat-7b | mistral-instruction-7b | 
|---| ---| ---|
|Jetson Orin Nano| (Memory: 3.89G) 1.03 tokens/s |(Memory: 4.16G) 0.96 tokens/s |

#### Capture of text output
|Question | llama2-chat-7b | mistral-instruction-7b | 
| --- | --- | --- |
|Hi, how are you?| ![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404041750989.png) | ![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404041755784.png)|
|whats the square root of 900? | ![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404041746798.png)| ![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404041757603.png)|
|can I get a recipie for french onion soup? |![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404041749849.png)![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404041750731.png) |![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404041803412.png)![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404041804728.png) |


### Turbomind

#### base-line
|Device | llama2-chat-7b | mistral-instruction-7b | 
|---| ---| ---|
|Jetson Orin Nano| (Memory: 5.1G) 13.36 tokens/s |(Memory: 4.9G) 12.61 tokens/s |

#### Capture of text output
|Question | llama2-chat-7b | mistral-instruction-7b | 
| --- | --- | --- |
|Hi, how are you?| ![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404042056737.png) | ![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404042011416.png)|
|whats the square root of 900? | ![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404042058712.png)| ![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404042013525.png)|
|can I get a recipie for french onion soup? |![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404042148118.png) |![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404042014754.png)![](https://raw.githubusercontent.com/Lemonkaikai/pics/main/202404042015181.png) |