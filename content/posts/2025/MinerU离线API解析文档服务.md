+++
title = 'MinerU离线API解析文档服务'
date = 2025-05-24T14:31:00+08:00
tags = ["Python", "MinerU", "Gradio"]
+++

## 一、MinerU

[MinerU](https://github.com/opendatalab/MinerU)是一个可以将PDF转换为结构化数据的项目，它是由[书生-浦语](https://github.com/InternLM/InternLM)团队开发维护的。

MinerU结合了多个项目，对于解析PDF，特别是扫描件有较好的效果。最主要的是官方提供了详细的快速开始文档和第三方的项目用于补充web和API能力。

主要功能有：

1. 删除页眉、页脚、脚注、页码等元素，确保语义连贯
2. 输出符合人类阅读顺序的文本，适用于单栏、多栏及复杂排版
3. 保留原文档的结构，包括标题、段落、列表等
4. 提取图像、图片描述、表格、表格标题及脚注
5. 自动识别并转换文档中的公式为LaTeX格式
6. 自动识别并转换文档中的表格为HTML格式
7. 自动检测扫描版PDF和乱码PDF，并启用OCR功能
8. 等等



## 二、打包MinerU镜像

怎么使用MinerU呢？官网提供了[Dockerfile](https://github.com/opendatalab/MinerU/blob/master/docker/china/Dockerfile)直接进行打包就可以在docker中使用了。



## 三、使用Web和API调用

由于官网的镜像只能在命令环境使用，所以可以添加下面两个项目到镜像中：

1. [Gradio APP](https://github.com/opendatalab/MinerU/tree/master/projects/gradio_app)： 使用Gradio将MinerU服务封装成web服务。

2. [API Service](https://github.com/opendatalab/MinerU/tree/master/projects/web_api)： 使用FastAPI把MinerU封装成API服务。

下面是使用Dockerfile

```dockerfile
# Use the official Ubuntu base image
FROM ubuntu:22.04

# Set environment variables to non-interactive to avoid prompts during installation
ENV DEBIAN_FRONTEND=noninteractive

# Update the package list and install necessary packages
RUN apt-get update && \
    apt-get install -y \
        software-properties-common && \
    add-apt-repository ppa:deadsnakes/ppa && \
    apt-get update && \
    apt-get install -y \
        python3.10 \
        python3.10-venv \
        python3.10-distutils \
        python3-pip \
        wget \
        git \
        libgl1 \
        libreoffice \
        fonts-noto-cjk \
        fonts-wqy-zenhei \
        fonts-wqy-microhei \
        ttf-mscorefonts-installer \
        fontconfig \
        libglib2.0-0 \
        libxrender1 \
        libsm6 \
        libxext6 \
        poppler-utils \
        && rm -rf /var/lib/apt/lists/*

# Set Python 3.10 as the default python3
RUN update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 1

# Create a virtual environment for MinerU
RUN python3 -m venv /opt/mineru_venv

# Copy the configuration file template and install magic-pdf latest
RUN /bin/bash -c "wget https://gcore.jsdelivr.net/gh/opendatalab/MinerU@master/magic-pdf.template.json && \
    cp magic-pdf.template.json /root/magic-pdf.json && \
    source /opt/mineru_venv/bin/activate && \
    pip3 install --upgrade pip -i https://mirrors.aliyun.com/pypi/simple && \
    pip3 install -U magic-pdf[full] -i https://mirrors.aliyun.com/pypi/simple"

# Download models and update the configuration file
RUN /bin/bash -c "pip3 install modelscope && \
    wget https://gcore.jsdelivr.net/gh/opendatalab/MinerU@master/scripts/download_models.py -O download_models.py && \
    python3 download_models.py && \
    sed -i 's|cpu|cuda|g' /root/magic-pdf.json"

# 增加 gradio 和 api 
RUN mkdir -p /app/examples

WORKDIR /app

RUN /bin/bash -c "\
wget https://gcore.jsdelivr.net/gh/opendatalab/MinerU@master/projects/gradio_app/app.py -O gra_app.py && \
wget https://gcore.jsdelivr.net/gh/opendatalab/MinerU@master/projects/gradio_app/header.html -O header.html && \
wget https://gcore.jsdelivr.net/gh/opendatalab/MinerU@master/projects/web_api/app.py -O app.py && \
source /opt/mineru_venv/bin/activate && \
pip3 install fastapi uvicorn python-multipart gradio gradio-pdf -i https://mirrors.aliyun.com/pypi/simple"
```



启动镜像的docker compose配置如下：
```yaml
services:
  mineru-gradio:
    # 如果不想自己打包，可以我打包好的镜像
    image: registry.cn-shanghai.aliyuncs.com/0xkyle/mineru:latest
    container_name: mineru-gradio
    # 启动gradio应用
    # command: /bin/bash -c "source /opt/mineru_venv/bin/activate && exec python gra_app.py"
    # 启动api服务
    command: /bin/bash -c "source /opt/mineru_venv/bin/activate && exec uvicorn app:app --host 0.0.0.0 --port 8000"
    ports:
      - "7860:7860"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

![MinerU](/images/MinerU测试.png)
