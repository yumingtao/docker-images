# example of how to build this docker:
# docker build -t tensorflowem --build-arg CHECKOUT_TF_SRC=1 --build-arg USE_PYTHON_3_NOT_2=1 . 
# example of how to run docker:
# docker run --runtime=nvidia -ti -u $(id -u):$(id -g) -v /home/yannick/:/home/yannick

ARG UBUNTU_VERSION=16.04

FROM nvidia/cuda:10.0-devel-ubuntu"${UBUNTU_VERSION}" as base
LABEL MAINTAINER "Research Development Team <dev@easymile.com>"

RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        cuda-command-line-tools-10-0 \
        cuda-cublas-dev-10-0 \
        cuda-cudart-dev-10-0 \
        cuda-cufft-dev-10-0 \
        cuda-curand-dev-10-0 \
        cuda-cusolver-dev-10-0 \
        cuda-cusparse-dev-10-0 \
        libcudnn7=7.4.1.5-1+cuda10.0 \
        libcudnn7-dev=7.4.1.5-1+cuda10.0 \
        libcurl3-dev \
        libfreetype6-dev \
        libhdf5-serial-dev \
        libpng-dev \
        libzmq3-dev \
        pkg-config \
        rsync \
        software-properties-common \
        unzip \
        zip \
        zlib1g-dev \
        wget \
        curl \
        git \
        && \
    find /usr/local/cuda-10.0/lib64/ -type f -name 'lib*_static.a' -not -name 'libcudart_static.a' -delete && \
    rm /usr/lib/x86_64-linux-gnu/libcudnn_static_v7.a

RUN apt-get update && \
        apt-get install nvinfer-runtime-trt-repo-ubuntu1604-5.0.2-ga-cuda10.0 \
        && apt-get update \
        && apt-get install -y --no-install-recommends libnvinfer-dev=5.0.2-1+cuda10.0 \
        && rm -rf /var/lib/apt/lists/*

# Configure the build for our CUDA configuration.
ENV CI_BUILD_PYTHON python
ENV LD_LIBRARY_PATH /usr/local/cuda/extras/CUPTI/lib64:$LD_LIBRARY_PATH
ENV TF_NEED_CUDA 1
ENV TF_NEED_TENSORRT 1
ENV TF_CUDA_COMPUTE_CAPABILITIES=3.5,5.2,6.0,6.1,7.0
ENV TF_CUDA_VERSION=10.0
ENV TF_CUDNN_VERSION=7

# Check out TensorFlow source code if --build_arg CHECKOUT_TENSORFLOW=1
ARG CHECKOUT_TF_SRC=0
RUN test "${CHECKOUT_TF_SRC}" -eq 1 && git clone --branch=r1.13 --depth=1 https://github.com/tensorflow/tensorflow.git /tensorflow

ARG USE_PYTHON_3_NOT_2
ARG _PY_SUFFIX=${USE_PYTHON_3_NOT_2:+3.6}
ARG PYTHON=python${_PY_SUFFIX}
ARG PIP=pip${_PY_SUFFIX}

# See http://bugs.python.org/issue19846
ENV LANG C.UTF-8

# Install python 3.6
RUN add-apt-repository ppa:jonathonf/python-${_PY_SUFFIX}
RUN apt-get update && apt-get install -y \
    ${PYTHON} \
    python3-pip

RUN curl https://bootstrap.pypa.io/get-pip.py | ${PYTHON}
RUN ${PIP} --no-cache-dir install --upgrade \
    pip \
    setuptools

# Some TF tools expect a "python" binary
RUN ln -s $(which ${PYTHON}) /usr/local/bin/python 

RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    git \
    wget \
    openjdk-8-jdk \
    ${PYTHON}-dev \
    swig


RUN ${PIP} --no-cache-dir install \
    Pillow==5.3.0 \
    h5py==2.8.0 \
    keras_applications \
    keras_preprocessing \
    matplotlib==3.0.0 \
    mock \
    numpy==1.15.3 \
    scipy==1.1.0 \
    sklearn \
    pandas \
    && test "${USE_PYTHON_3_NOT_2}" -eq 1 && true || ${PIP} --no-cache-dir install \
    enum34

# Install bazel
ARG BAZEL_VERSION=0.19.2
RUN mkdir /bazel && \
    wget -O /bazel/installer.sh "https://github.com/bazelbuild/bazel/releases/download/${BAZEL_VERSION}/bazel-${BAZEL_VERSION}-installer-linux-x86_64.sh" && \
    wget -O /bazel/LICENSE.txt "https://raw.githubusercontent.com/bazelbuild/bazel/master/LICENSE" && \
    chmod +x /bazel/installer.sh && \
    /bazel/installer.sh && \
    rm -f /bazel/installer.sh

COPY bashrc /etc/bash.bashrc
RUN chmod a+rwx /etc/bash.bashrc

# from build-gpu.sh:
RUN echo ${PYTHON}
RUN ln -sfn $(which ${PYTHON}) /usr/local/bin/python 

RUN ln -s /usr/local/cuda/lib64/stubs/libcuda.so /usr/local/cuda/lib64/stubs/libcuda.so.1

ENV LD_LIBRARY_PATH /usr/local/cuda/lib64/stubs:$LD_LIBRARY_PATH

WORKDIR /tensorflow
# new Bazel expects .rc files in different locations:
# RUN rm .bazelrc
RUN mv tools/bazel.rc .bazelrc

# NCCL 2.x
ENV TF_NCCL_VERSION=2

RUN tensorflow/tools/ci_build/builds/configured GPU

#### apply patch, ref: https://github.com/tensorflow/tensorflow/issues/23853
COPY 0001-TF.patch /tmp/0001-TF.patch
RUN git config --global user.email "dev@easymile.com"
RUN git config --global user.name "EasyMile"
RUN git am /tmp/0001-TF.patch


# Compiling .whl as well as c api (avx2 did not show any speed improvements)
#RUN bazel build -c opt --copt=-mavx --copt=-mavx2 --copt=-mfma --copt=-mfpmath=both --copt=-msse4.2 --config=cuda \
RUN bazel build -c opt --copt=-mavx --config=cuda --output_filter=DONT_MATCH_ANYTHING \
    --cxxopt="-D_GLIBCXX_USE_CXX11_ABI=0" \
    tensorflow/tools/pip_package:build_pip_package \
    tensorflow:libtensorflow.so

RUN rm /usr/local/cuda/lib64/stubs/libcuda.so.1
RUN bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/pip

# Testing .whl installation
RUN ${PIP} --no-cache-dir install --upgrade /tmp/pip/tensorflow-*.whl

# opencv-python useful when running other scripts...
RUN ${PIP} --no-cache-dir install \
    imutils \
    opencv-python
