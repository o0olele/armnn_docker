FROM ubuntu:16.04
ENV TERM linux
ENV DEBIAN_FRONTEND noninteractive

# apt
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends locales ca-certificates &&  rm -rf /var/lib/apt/lists/*

# 设置
RUN locale-gen en_US.UTF-8 && update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LC_ALL en_US.UTF-8

# 关闭权限验证
RUN echo "check_certificate = off" >> ~/.wgetrc
RUN echo "[global] \n\
trusted-host = pypi.python.org \n \
\t               pypi.org \n \
\t              files.pythonhosted.org" >> /etc/pip.conf

# 安装基础包
RUN apt-get update && apt-get install -y \
    software-properties-common \
    apparmor \
    aufs-tools \
    automake \
    bash-completion \
    btrfs-tools \
    build-essential \
    cmake \
    createrepo \
    curl \
    dpkg-sig \
    g++ \
    gcc \
    git \
    iptables \
    jq \
    libapparmor-dev \
    libc6-dev \
    libcap-dev \
    libsystemd-dev \
    libyaml-dev \
    mercurial \
    net-tools \
    parallel \
    pkg-config \
    python-dev \
    python-mock \
    python-pip \
    python-setuptools \
    python-websocket \
    golang-go \
    iproute2 \
    iputils-ping \
    vim-common \
    vim \
    wget \
    libtool \
    unzip \
    scons \
    curl \
    autoconf \
    libtool \
    build-essential \
    g++ \ 
    cmake && rm -rf /var/lib/apt/lists/*

# 安装交叉编译工具链
RUN apt-get update && apt-get install -y crossbuild-essential-arm64

# 创建工作目录
RUN cd $HOME && mkdir armnn-tflite

# 安装scons
RUN cd $HOME && \
    wget https://sourceforge.net/projects/scons/files/scons/2.4.1/scons-2.4.1.tar.gz && \
    tar xf scons-2.4.1.tar.gz && \
    cd scons-2.4.1 && \
    python setup.py install

# 获取指定版本的aarch-linux-gnu
# RUN cd $HOME && \
#     wget https://releases.linaro.org/components/toolchain/binaries/6.3-2017.05/aarch64-linux-gnu/gcc-linaro-6.3.1-2017.05-x86_64_aarch64-linux-gnu.tar.xz && \
#     tar -xf gcc-linaro-6.3.1-2017.05-x86_64_aarch64-linux-gnu.tar.xz && \
#     mv ./gcc-linaro-6.3.1-2017.05-x86_64_aarch64-linux-gnu linaro-gcc && \
#     export PATH=$HOME/linaro-gcc/bin:$PATH

# 建立软链接(这一步是在构建flatc的时候，出现类似"/lib/ld-linux-aarch64.so.1: No such file or directory"的错误的解决)
# RUN ln -s $HOME/linaro-gcc/aarch64-linux-gnu/libc/lib/ld-linux-aarch64.so.1 /lib/ld-linux-aarch64.so.1 && \
#     ln -s $HOME/linaro-gcc/aarch64-linux-gnu/libc/lib/libstdc++.so.6 /lib/libstdc++.so.6 && \
#     ln -s $HOME/linaro-gcc/aarch64-linux-gnu/libc/lib/libm.so.6 /lib/libm.so.6 && \
#     ln -s $HOME/linaro-gcc/aarch64-linux-gnu/libc/lib/libgcc_s.so.1 /lib/libgcc_s.so.1 && \
#     ln -s $HOME/linaro-gcc/aarch64-linux-gnu/libc/lib/libc.so.6 /lib/libc.so.6

# 安装ComputeLibrary
# scons编译参数可以自己按需调整
# 仓库地址可换成 https://github.com/ARM-software/ComputeLibrary.git
# RUN export PATH=$HOME/linaro-gcc/bin:$PATH && \
RUN cd $HOME && \
    git clone https://gitee.com/rick_liang/ComputeLibrary.git && \
    cd $HOME/ComputeLibrary && \
    git checkout -b 20_02 v20.02 && \
    scons arch=arm64-v8a validation_tests=0 Werror=1 debug=0 asserts=0 neon=1 opencl=1 embed_kernels=1 benchmark_tests=0 extra_cxx_flags="-fPIC" -j4 internal_only=0

# 安装Boost库
# RUN export PATH=$HOME/linaro-gcc/bin:$PATH && \
RUN cd $HOME && \
    wget https://dl.bintray.com/boostorg/release/1.64.0/source/boost_1_64_0.tar.bz2 && \
    tar xf boost_1_64_0.tar.bz2 && \
    cd $HOME/boost_1_64_0 && \
    echo "using gcc : arm : aarch64-linux-gnu-g++ ;" > user_config.jam && \
    ./bootstrap.sh --prefix=$HOME/armnn-tflite/boost_arm64_install && \
    ./b2 install toolset=gcc-arm link=static cxxflags=-fPIC --with-filesystem --with-test --with-log --with-program_options -j4 --user-config=user_config.jam 

# 安装Google protobuf
# 这一步如果卡在了某个编译的地方，可能是docker的内存不足，可以将make install -j8降低为-j4等
# 仓库地址可换成 https://gitee.com/rick_liang/protobuf.git 
# RUN export PATH=$HOME/linaro-gcc/bin:$PATH && \
RUN cd $HOME && \
    git clone -b v3.5.2 https://github.com/google/protobuf.git && \
    cd protobuf && \
    ./autogen.sh && \
    mkdir x86_64_build && \
    cd x86_64_build && \
    ../configure --prefix=$HOME/armnn-tflite/google/x86_64_pb_install && \
    make install -j8

# 安装Google protobuf arm版本
# RUN export PATH=$HOME/linaro-gcc/bin:$PATH && \
RUN cd $HOME/protobuf && \
    mkdir arm64_build && \
    cd arm64_build && \
    export CC=aarch64-linux-gnu-gcc && \
    export CXX=aarch64-linux-gnu-g++ && \
    ../configure --host=aarch64-linux \
    --prefix=$HOME/armnn-tflite/google/arm64_pb_install \
    --with-protoc=$HOME/armnn-tflite/google/x86_64_pb_install/bin/protoc && \
    make install -j8

# 首先下载下armnn
# 替换 https://github.com/ARM-software/armnn.git
RUN cd $HOME && \
    git clone https://gitee.com/rick_liang/armnn.git && \
    cd armnn && \
    git checkout -b armnn_21_02 origin/branches/armnn_20_02

# 安装tensorflow protobuf
RUN cd $HOME && \
    wget https://github.com/tensorflow/tensorflow/archive/v2.0.0.tar.gz && \
    tar -xf v2.0.0.tar.gz && \
    cd tensorflow-2.0.0 && \
    ../armnn/scripts/generate_tensorflow_protobuf.sh ../tensorflow-protobuf ../armnn-tflite/google/x86_64_pb_install/

# 安装flatbuffers native
# RUN export PATH=$HOME/linaro-gcc/bin:$PATH && \
RUN cd $HOME && \
    wget -O flatbuffers-1.12.0.tar.gz https://github.com/google/flatbuffers/archive/v1.12.0.tar.gz && \
    tar -xf flatbuffers-1.12.0.tar.gz && \
    cd flatbuffers-1.12.0 && \
    mkdir build && \
    cd build && \
    cmake .. -DFLATBUFFERS_BUILD_FLATC=1 \
     -DCMAKE_INSTALL_PREFIX:PATH=$HOME/armnn-tflite/flatbuffers \
     -DFLATBUFFERS_BUILD_TESTS=0 && \
    make all install

# 安装flatbuffers arm
# RUN export PATH=$HOME/linaro-gcc/bin:$PATH && \
RUN cd $HOME/flatbuffers-1.12.0 && \
    mkdir arm64_build && \
    cd arm64_build && \
    CXXFLAGS="-fPIC" cmake .. -DCMAKE_C_COMPILER=/usr/bin/aarch64-linux-gnu-gcc \
     -DCMAKE_CXX_COMPILER=/usr/bin/aarch64-linux-gnu-g++ \
     -DFLATBUFFERS_BUILD_FLATC=1 \
     -DCMAKE_INSTALL_PREFIX:PATH=$HOME/armnn-tflite/flatbuffers-arm64 \
     -DFLATBUFFERS_BUILD_TESTS=0 && \
    make all install

# 安装onnx
RUN git clone https://gitee.com/gitmirror/onnx.git && \
    cd onnx && \
    git fetch https://gitee.com/gitmirror/onnx.git 553df22c67bee5f0fe6599cff60f1afc6748c635 && \
    git checkout FETCH_HEAD && \
    export LD_LIBRARY_PATH=$HOME/armnn-tflite/google/x86_64_pb_install/lib/:$LD_LIBRARY_PATH && \
    $HOME/armnn-tflite/google/x86_64_pb_install/bin/protoc onnx/onnx.proto --proto_path=. --proto_path=../armnn-tflite/google/x86_64_pb_install/include --cpp_out $HOME/armnn-tflite

# 安装tflite
RUN cd $HOME/armnn-tflite && \
    mkdir tflite && \
    cd tflite && \
    cp ../../tensorflow-2.0.0/tensorflow/lite/schema/schema.fbs . && \
    ../../flatbuffers-1.12.0/build/flatc -c --gen-object-api --reflect-types --reflect-names schema.fbs

# RUN export PATH=$HOME/linaro-gcc/bin:$PATH && \
RUN cd $HOME/armnn && \
    mkdir build && \
    cd build && \
    CC=aarch64-linux-gnu-gcc CXX=aarch64-linux-gnu-g++ \ 
    cmake .. \
    -DARMCOMPUTE_ROOT=$HOME/ComputeLibrary \
    -DARMCOMPUTE_BUILD_DIR=$HOME/ComputeLibrary/build/ \
    -DBOOST_ROOT=$HOME/armnn-tflite/boost_arm64_install \
    -DTF_GENERATED_SOURCES=$HOME/tensorflow-protobuf/ \
    -DBUILD_TF_PARSER=1 \
    -DBUILD_TF_LITE_PARSER=1 \
    -DBUILD_TESTS=1 \
    -DONNX_GENERATED_SOURCES=$HOME/armnn-tflite \
    -DBUILD_ONNX_PARSER=1 \
    -DARMCOMPUTENEON=1 \
    -DARMCOMPUTECL=1 \
    -DTF_LITE_GENERATED_PATH=$HOME/tensorflow-2.0.0/tensorflow/lite/schema \
    -DPROTOBUF_ROOT=$HOME/armnn-tflite/google/x86_64_pb_install/ \
    -DPROTOBUF_LIBRARY_DEBUG=$HOME/armnn-tflite/google/arm64_pb_install/lib/libprotobuf.so \
    -DPROTOBUF_LIBRARY_RELEASE=$HOME/armnn-tflite/google/arm64_pb_install/lib/libprotobuf.so \
    -DARMNNREF=1 \
    -DFLATBUFFERS_ROOT=$HOME/armnn-tflite/flatbuffers-arm64 \
    -DFLATBUFFERS_LIBRARY=$HOME/armnn-tflite/flatbuffers-arm64/lib/libflatbuffers.a \
    -DBUILD_ARMNN_SERIALIZER=1 \
    -DBUILD_ARMNN_QUANTIZER=1 \
    -DFLATC_DIR=$HOME/flatbuffers-1.12.0/build && \
    make -j4