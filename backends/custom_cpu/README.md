# PaddlePaddle Custom Device Implementaion for Custom CPU

English | [简体中文](./README_cn.md)

Please refer to the following steps to compile, install and verify the custom device implementaion for Custom CPU.

## Prepare environment and source code

```bash
# 1. pull PaddlePaddle CPU development docker image
# dockerfile of the image is in tools/dockerfile directory
docker pull registry.baidubce.com/device/paddle-cpu:ubuntu18-x86_64-gcc82
docker pull registry.baidubce.com/device/paddle-cpu:ubuntu18-aarch64-gcc82

# 2. refer to the following commands to start docker container
docker run -it --name paddle-dev-cpu -v `pwd`:/workspace \
       --network=host --shm-size=128G --workdir=/workspace \
       --cap-add=SYS_PTRACE --security-opt seccomp=unconfined \
       registry.baidubce.com/device/paddle-cpu:ubuntu18-$(uname -m)-gcc82 /bin/bash

# 3. clone the source code recursively along with Paddle source code
git clone --recursive https://github.com/PaddlePaddle/PaddleCustomDevice
cd PaddleCustomDevice

# 4. execute the following commands to update submodule
git submodule sync
git submodule update --remote --init --recursive
```

## Compile and Install

```bash
# navigate to implementaion for Custom CPU
cd backends/custom_cpu

# before compiling, ensure that Paddle is installed, you can run the following command
pip install paddlepaddle==0.0.0 -f https://www.paddlepaddle.org.cn/whl/linux/cpu-mkl/develop.html

# create the build directory and navigate in
mkdir build && cd build

cmake ..
make -j8

# using pip to install the output
pip install dist/paddle_custom_cpu*.whl
```

## Verification

```bash
# list available hardware backends
python -c "import paddle; print(paddle.device.get_all_custom_device_type())"

# expected output
['custom_cpu']

# run a simple model
python ../tests/test_MNIST_model.py

# expected similar output
... ...
Epoch 0 step 0, Loss = [2.2956038], Accuracy = 0.15625
Epoch 0 step 100, Loss = [2.1552896], Accuracy = 0.3125
Epoch 0 step 200, Loss = [2.1177733], Accuracy = 0.4375
Epoch 0 step 300, Loss = [2.0089214], Accuracy = 0.53125
Epoch 0 step 400, Loss = [2.0845466], Accuracy = 0.421875
Epoch 0 step 500, Loss = [2.0473], Accuracy = 0.453125
Epoch 0 step 600, Loss = [1.8561764], Accuracy = 0.71875
Epoch 0 step 700, Loss = [1.9915285], Accuracy = 0.53125
Epoch 0 step 800, Loss = [1.8925955], Accuracy = 0.640625
Epoch 0 step 900, Loss = [1.8199624], Accuracy = 0.734375
```

## Using PaddleInference

Re-compile plugin

```bash
# Compile PaddleInference
git clone https://github.com/PaddlePaddle/Paddle.git
git clone https://github.com/ronny1996/Paddle-Inference-Demo.git

mkdir -p Paddle/build
pushd Paddle/build

cmake .. -DPY_VERSION=3.7 -DWITH_GPU=OFF -DWITH_TESTING=ON -DCMAKE_BUILD_TYPE=Release -DON_INFER=ON -DWITH_MKL=ON -DWITH_CUSTOM_DEVICE=ON

make -j8

popd
cp -R Paddle/build/paddle_inference_install_dir Paddle-Inference-Demo/c++/lib/paddle_inference
export PADDLE_INFERENCE_LIB_DIR=$(realpath Paddle-Inference-Demo/c++/lib/paddle_inference/paddle/lib)

# Compile the plug-in
mkdir -p PaddleCustomDevice/backends/custom_cpu/build
pushd PaddleCustomDevice/backends/custom_cpu/build

cmake .. -DON_INFER=ON -DPADDLE_INFERENCE_LIB_DIR=${PADDLE_INFERENCE_LIB_DIR}
make -j8

# Specify the plug-in directory
export CUSTOM_DEVICE_ROOT=$PWD
popd
```

Using PaddleInference

```bash
pushd Paddle-Inference-Demo/c++/resnet50

# Modify resnet50_test.cc, use config.EnableCustomDevice("custom_cpu", 0) to replace config.EnableUseGpu(100, 0)

bash run.sh
```

expected similar output

```bash
I0713 09:02:38.808723 24792 resnet50_test.cc:74] run avg time is 297.75 ms
I0713 09:02:38.808859 24792 resnet50_test.cc:89] 0 : 8.76192e-29
I0713 09:02:38.808894 24792 resnet50_test.cc:89] 100 : 8.76192e-29
I0713 09:02:38.808904 24792 resnet50_test.cc:89] 200 : 8.76192e-29
I0713 09:02:38.808912 24792 resnet50_test.cc:89] 300 : 8.76192e-29
I0713 09:02:38.808920 24792 resnet50_test.cc:89] 400 : 8.76192e-29
I0713 09:02:38.808928 24792 resnet50_test.cc:89] 500 : 8.76192e-29
I0713 09:02:38.808936 24792 resnet50_test.cc:89] 600 : 1.05766e-19
I0713 09:02:38.808945 24792 resnet50_test.cc:89] 700 : 2.04093e-23
I0713 09:02:38.808954 24792 resnet50_test.cc:89] 800 : 3.85255e-25
I0713 09:02:38.808961 24792 resnet50_test.cc:89] 900 : 8.76192e-29
```
