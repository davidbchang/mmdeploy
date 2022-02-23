FROM openvino/ubuntu18_dev:2021.4.2
ARG PYTHON_VERSION=3.7
ARG TORCH_VERSION=1.8.0
ARG TORCHVISION_VERSION=0.9.0
ARG ONNXRUNTIME_VERSION=1.8.1
ARG MMCV_VERSION=1.4.0
ARG CMAKE_VERSION=3.20.0
USER root
RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates \
        libopencv-dev libspdlog-dev \
        gnupg \
        libssl-dev \
        libprotobuf-dev protobuf-compiler \
        build-essential \
        libjpeg-dev \
        libpng-dev \
        ccache \
        cmake \
        gcc \
        g++ \
        git \
        vim \
        wget \
        curl \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL -v -o ~/miniconda.sh -O  https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh  && \
    chmod +x ~/miniconda.sh && \
    ~/miniconda.sh -b -p /opt/conda && \
    rm ~/miniconda.sh && \
    /opt/conda/bin/conda install -y python=${PYTHON_VERSION} conda-build pyyaml numpy ipython cython typing typing_extensions mkl mkl-include ninja && \
    /opt/conda/bin/conda clean -ya

### pytorch
RUN /opt/conda/bin/pip install torch==${TORCH_VERSION}+cpu torchvision==${TORCHVISION_VERSION}+cpu -f https://download.pytorch.org/whl/cpu/torch_stable.html
ENV PATH /opt/conda/bin:$PATH

### install open-mim
RUN /opt/conda/bin/pip install mmcv-full==${MMCV_VERSION} -f https://download.openmmlab.com/mmcv/dist/cpu/torch${TORCH_VERSION}/index.html

WORKDIR /root/workspace

### get onnxruntime
RUN wget https://github.com/microsoft/onnxruntime/releases/download/v${ONNXRUNTIME_VERSION}/onnxruntime-linux-x64-${ONNXRUNTIME_VERSION}.tgz \
    && tar -zxvf onnxruntime-linux-x64-${ONNXRUNTIME_VERSION}.tgz

ENV ONNXRUNTIME_DIR=/root/workspace/onnxruntime-linux-x64-${ONNXRUNTIME_VERSION}

### update cmake to 20
RUN wget https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}/cmake-${CMAKE_VERSION}.tar.gz &&\
    tar -zxvf cmake-${CMAKE_VERSION}.tar.gz &&\
    cd cmake-${CMAKE_VERSION} &&\
    ./bootstrap &&\
    make &&\
    make install

### install onnxruntme and openvino
RUN /opt/conda/bin/pip install onnxruntime==${ONNXRUNTIME_VERSION} openvino-dev

### build ncnn
RUN git clone https://github.com/Tencent/ncnn.git &&\
    cd ncnn &&\
    export NCNN_DIR=$(pwd) &&\
    git submodule update --init &&\
    mkdir -p build && cd build &&\
    cmake -DNCNN_VULKAN=OFF -DNCNN_SYSTEM_GLSLANG=ON -DNCNN_BUILD_EXAMPLES=ON -DNCNN_PYTHON=ON -DNCNN_BUILD_TOOLS=ON -DNCNN_BUILD_BENCHMARK=ON -DNCNN_BUILD_TESTS=ON .. &&\
    make install &&\
    cd /root/workspace/ncnn/python &&\
    pip install -e .

### install mmdeploy
WORKDIR /root/workspace
ARG VERSION
RUN git clone https://github.com/open-mmlab/mmdeploy.git &&\
    cd mmdeploy &&\
    if [ -z ${VERSION} ] ; then echo "No MMDeploy version passed in, building on master" ; else git checkout tags/v${VERSION} -b tag_v${VERSION} ; fi &&\
    git submodule update --init --recursive &&\
    rm -rf build &&\
    mkdir build &&\
    cd build &&\
    cmake -DMMDEPLOY_TARGET_BACKENDS=ncnn -Dncnn_DIR=/root/workspace/ncnn/build/install/lib/cmake/ncnn .. &&\
    make -j$(nproc) &&\
    cmake -DMMDEPLOY_TARGET_BACKENDS=ort .. &&\
    make -j$(nproc) &&\
    cd .. &&\
    pip install -e .

### build SDK
ENV LD_LIBRARY_PATH="/root/workspace/mmdeploy/build/lib:/opt/intel/openvino/deployment_tools/ngraph/lib:/opt/intel/openvino/deployment_tools/inference_engine/lib/intel64:${LD_LIBRARY_PATH}"
RUN cd mmdeploy && rm -rf build/CM* && mkdir -p build && cd build && cmake .. \
    -DMMDEPLOY_BUILD_SDK=ON \
    -DCMAKE_CXX_COMPILER=g++-7 \
    -DONNXRUNTIME_DIR=${ONNXRUNTIME_DIR} \
    -Dncnn_DIR=/root/workspace/ncnn/build/install/lib/cmake/ncnn \
    -DInferenceEngine_DIR=/opt/intel/openvino/deployment_tools/inference_engine/share \
    -DMMDEPLOY_TARGET_DEVICES=cpu \
    -DMMDEPLOY_BUILD_SDK_PYTHON_API=ON \
    -DMMDEPLOY_TARGET_BACKENDS="ort;ncnn;openvino" \
    -DMMDEPLOY_CODEBASES=all &&\
    cmake --build . -- -j$(nproc) && cmake --install . &&\
    cd install/example  && mkdir -p build && cd build &&\
    cmake -DMMDeploy_DIR=/root/workspace/mmdeploy/build/install/lib/cmake/MMDeploy .. &&\
    cmake --build . && export SPDLOG_LEVEL=warn &&\
    if [ -z ${VERSION} ] ; then echo "Built MMDeploy master for CPU devices successfully!" ; else echo "Built MMDeploy version v${VERSION} for CPU devices successfully!" ; fi