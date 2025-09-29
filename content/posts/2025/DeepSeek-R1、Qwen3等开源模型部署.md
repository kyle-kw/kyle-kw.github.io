+++
title = 'DeepSeek-R1、Qwen3等开源模型部署'
date = 2025-09-29T10:00:00+08:00
tags = ["LLM", "VLLM", "Sglang", "DeepSeek-R1", "Qwen"]
+++


下面会放一些，部署在H200/A100上模型的启动配置。VLLM/Sglang是当时最新的版本，理论上最新的VLLM/Sglang也能部署，但要注意参数是否有变化。

部署环境：
- 操作系统: RedHat 9.4
- 内存: 2T
- 磁盘: 14T
- GPU: H200 * 8
- Nvidia Driver 570.86.10
- CUDA: 12.8


## 1. Deepseek-R1

```yaml
services:
  sglang-deepseek-r1-0528:
    image: lmsysorg/sglang:v0.4.6.post5-cu124
    container_name: sglang-deepseek-r1-0528
    restart: always
    runtime: nvidia
    shm_size: '30gb'
    volumes:
      - /data/models/DeepSeek-R1-0528:/data/DeepSeek-R1-0528
    environment:
      TZ: Asia/Shanghai
      LOG_LEVEL: INFO
    ports:
       - 30000:30000
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "10"
    entrypoint: >
      python3 -m sglang.launch_server 
      --model /data/DeepSeek-R1-0528
      --tp 8
      --host 0.0.0.0
      --post 30000
      --trust-remote-code
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

参考：
1. [DeepSeek-R1部署优化](https://evsio0n.com/archives/65/)

## 2. Qwen3-235B-A22B-Instruct

```yaml
services:
  sglang-qwen3-235b-a22b:
    image: lmsysorg/sglang:v0.5.1.post2-cu126
    container_name: sglang-qwen3-235b-a22b
    restart: always
    runtime: nvidia
    shm_size: '30gb'
    volumes:
      - /data/models/Qwen3-235B-A22B-Instruct-2507:/data/Qwen3-235B-A22B-Instruct-2507
    environment:
      TZ: Asia/Shanghai
      LOG_LEVEL: INFO
      CUDA_VISIBLE_DEVICES: 0,1,2,3
    ports:
       - 31000:31000
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "10"
    entrypoint: >
      python3 -m sglang.launch_server 
      --model /data/Qwen3-235B-A22B-Instruct-2507
      --tp 4
      --host 0.0.0.0
      --post 31000
      --context-length 262144
      --trust-remote-code
```

## 3. Qwen3-235B-A22B-Thinking

```yaml
services:
  sglang-qwen3-235b-a22b-thinking:
    image: lmsysorg/sglang:v0.5.1.post2-cu126
    container_name: sglang-qwen3-235b-a22b-thinking
    restart: always
    runtime: nvidia
    shm_size: '30gb'
    volumes:
      - /data/models/Qwen3-235B-A22B-Thinking-2507:/data/Qwen3-235B-A22B-Thinking-2507
    environment:
      TZ: Asia/Shanghai
      LOG_LEVEL: INFO
      CUDA_VISIBLE_DEVICES: 4,5,6,7
    ports:
       - 32000:32000
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "10"
    entrypoint: >
      python3 -m sglang.launch_server 
      --model /data/Qwen3-235B-A22B-Thinking-2507
      --tp 4
      --host 0.0.0.0
      --post 32000
      --context-length 262144
      --reasoning-parser deepseek-r1
      --trust-remote-code
```

## 4. Qwen2.5-VL-72B-Instruct

```yaml
services:
  vllm-qwen2.5-vl-72b:
    image: vllm/vllm-openai:v0.7.3
    container_name: vllm-qwen2.5-vl-72b
    restart: always
    runtime: nvidia
    shm_size: '20gb'
    volumes:
      - /data/models/Qwen2.5-VL-72B-Instruct:/data/Qwen2.5-VL-72B-Instruct
    environment:
      TZ: Asia/Shanghai
      LOG_LEVEL: INFO
      CUDA_VISIBLE_DEVICES: 0,1
    ports:
       - 31100:31100
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "10"

    entrypoint: >
      vllm serve /data/Qwen2.5-VL-72B-Instruct
      --host 0.0.0.0
      --port 31100
      --served-model-name qwen2.5-vl-72b
      --max-model-len 32768
      --gpu-memory-utilization 0.7
      --tensor-parallel-size 2
      --limit-mm-per-prompt image=16
      --enable-prefix-caching
      --trust-remote-code
```

## 5. Qwen2.5-72B-Instruct

```yaml
services:
  vllm-qwen2.5-72b:
    image: vllm/vllm-openai:v0.7.3
    container_name: vllm-qwen2.5-72b
    restart: always
    runtime: nvidia
    shm_size: '20gb'
    volumes:
      - /data/models/Qwen2.5-72B-Instruct:/data/Qwen2.5-72B-Instruct
    environment:
      TZ: Asia/Shanghai
      LOG_LEVEL: INFO
      CUDA_VISIBLE_DEVICES: 0,1
    ports:
       - 31200:31200
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "10"

    entrypoint: >
      vllm serve /data/Qwen2.5-72B-Instruct
      --host 0.0.0.0
      --port 31200
      --served-model-name qwen2.5-72b
      --max-model-len 32768
      --gpu-memory-utilization 0.7
      --tensor-parallel-size 2
      --enable-prefix-caching
      --trust-remote-code
      --enable-auto-tool-choice
      --tool-call-parser hermes
```

## 6. bge-m3

```yaml
services:
  vllm-bge-m3:
    image: vllm/vllm-openai:v0.8.1
    container_name: vllm-bge-m3
    restart: always
    runtime: nvidia
    shm_size: '10gb'
    volumes:
      - /data/models/bge-m3:/data/bge-m3
    environment:
      TZ: Asia/Shanghai
      LOG_LEVEL: INFO
      CUDA_VISIBLE_DEVICES: 7
    ports:
       - 33000:33000
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "10"

    entrypoint: >
      vllm serve /data/bge-m3
      --host 0.0.0.0
      --port 33000
      --served-model-name bge-m3
      --max-model-len 8192
      --gpu-memory-utilization 0.05
      --trust-remote-code
```

## 7. bge-reranker-v2-m3

```yaml
services:
  vllm-bge-reranker-v2-m3:
    image: vllm/vllm-openai:v0.7.2
    container_name: vllm-bge-reranker-v2-m3
    restart: always
    runtime: nvidia
    shm_size: '10gb'
    volumes:
      - /data/models/bge-reranker-v2-m3:/data/bge-reranker-v2-m3
    environment:
      TZ: Asia/Shanghai
      LOG_LEVEL: INFO
      CUDA_VISIBLE_DEVICES: 7
    ports:
       - 34000:34000
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "10"

    entrypoint: >
      vllm serve /data/bge-m3
      --host 0.0.0.0
      --port 34000
      --served-model-name bge-reranker-v2-m3
      --task score
      --max-model-len 8192
      --gpu-memory-utilization 0.05
      --trust-remote-code
```


## 8. gpt-oss-120b

```yaml
services:
  vllm-gpt-oss-120b:
    image: vllm/vllm-openai:gptoss
    container_name: vllm-gpt-oss-120b
    restart: always
    runtime: nvidia
    shm_size: '20gb'
    volumes:
      - /data/models/gpt-oss-120b:/data/gpt-oss-120b
      - ./harmony:/data/harmony
    environment:
      TZ: Asia/Shanghai
      LOG_LEVEL: INFO
      CUDA_VISIBLE_DEVICES: 0
      VLLM_ATTENTION_BACKEND: TRITON_ATTN_VLLM_V1
      TIKTOKEN_RS_CACHE_DIR: /data/harmony
    ports:
       - 32100:32100
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "10"

    entrypoint: >
      vllm serve /data/gpt-oss-120b
      --host 0.0.0.0
      --port 32100
      --served-model-name gpt-oss-120b
      --max-model-len 32768
      --gpu-memory-utilization 0.9
      --tensor-parallel-size 1
      --enable-prefix-caching
      --trust-remote-code
      --async-scheduling
```

harmony 文件参考：[openai_harmony.HarmonyError: error downloading or loading vocab file: failed to download or load vocab file](https://huggingface.co/openai/gpt-oss-120b/discussions/39#6892d6cc421a56a8deb417a7)

## 9. GLM-4.5V

```yaml
services:
  sglang-glm-4.5v:
    image: lmsysorg/sglang:v0.5.2
    container_name: sglang-glm-4.5v
    restart: always
    runtime: nvidia
    shm_size: '30gb'
    volumes:
      - /data/models/glm-4.5v:/data/glm-4.5v
    environment:
      TZ: Asia/Shanghai
      LOG_LEVEL: INFO
      CUDA_VISIBLE_DEVICES: 0,1,2,3
    ports:
       - 31000:31000
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "10"
    entrypoint: >
      python3 -m sglang.launch_server 
      --model /data/glm-4.5v
      --served-model-name glm-4.5v
      --tp 4
      --host 0.0.0.0
      --post 31000
      --context-length 65536
      --reasoning-parser glm45
      --tool-call-parser glm45
      --limit-mm-per-prompt image=16
      --trust-remote-code
```


## 10. Qwen3-VL-235B-A22B-Thinking

```dockerfile
FROM lmsysorg/sglang:dev
# sha256:0318b32b3b38cf75dd5b9a4e6abdece3048ddc38720299d60f2fe4ae5ab8c642

RUN pip3 install git+https://github.com/huggingface/transformers

# docker build . -t lmsysorg/sglang:dev-v1
```

由于目前（2025年9月29日）transformers还没有发布4.57.0，所以需要手动更新镜像中的transformers。

```yaml
services:
  sglang-qwen3-vl-235b-a22b-thinking:
    image: lmsysorg/sglang:dev-v1
    container_name: sglang-qwen3-vl-235b-a22b-thinking
    restart: always
    runtime: nvidia
    shm_size: '30gb'
    volumes:
      - /data/models/Qwen3-VL-235B-A22B-Thinking:/Qwen3-VL-235B-A22B-Thinking
      - ./template_qwen3_vl_thinking.jinja:/sgl-workspace/sglang/template_qwen3_vl_thinking.jinja
    environment:
      TZ: Asia/Shanghai
      LOG_LEVEL: INFO
    ports:
       - 30000:30000
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "10"
    entrypoint: >
      python3 -m sglang.launch_server 
      --model /data/Qwen3-VL-235B-A22B-Thinking
      --served-model-name qwen3-vl-235b-a22b-thinking
      --chat-template template_qwen3_vl_thinking.jinja
      --tp 8
      --host 0.0.0.0
      --post 30000
      --trust-remote-code
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

目前（2025年9月29日）sglang中没有适配thinking的chat template，会导致读取不到图片，因此需要手动指定，参考依据： [sglang pr](https://github.com/sgl-project/sglang/pull/10946#issue-3456278834)

等过段时间transformers发布了4.57.0，sglang也会发布新版本，到时直接替换镜像名字即可部署。
