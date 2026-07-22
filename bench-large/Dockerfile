FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    build-essential \
    cmake \
    llvm \
    clang \
    libboost-all-dev \
    python3 \
    python3-pip \
    nodejs \
    npm \
    libssl-dev \
    zlib1g-dev \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*

RUN pip3 install numpy scipy pandas

CMD ["echo", "large benchmark image built"]
