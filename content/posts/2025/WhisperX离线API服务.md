+++
title = 'Whisperx离线API服务'
date = 2025-05-24T11:23:17+08:00
tags = ["Python", "Whisper", "WhisperX", "Docker"]
+++


## 一、Whisper

[Whisper](https://github.com/openai/whisper)是OpenAI开源的通用语音识别模型，支持多种语言，当然也包括中文。它可以进行语音识别、语音翻译和语音识别。



## 二、WhisperX项目

[WhisperX](https://github.com/m-bain/whisperX)在Whisper的基础上增加了单词级别的时间戳和说话人分类。还添加了优化，比如预处理音频活动检测，减少幻觉和批量处理。

其中使用了

1. `Phoneme-Based ASR`: 自动语音识别语音中的最小单位。
2. `Forced Alignment`: 将文本和音频进行对其，实现生成音素级别的分割。
3. `Voice Activity Detection (VAD)`: 语音活动检测。
4. `Speaker Diarization`: 将每个发言人的音频进行划分。



## 三、打包WhisperX镜像

WhisperX官方提供了安装环境的流程，但由于主机环境各不相同，且考虑有离线使用WhisperX的需求，所以尝试将WhisperX打包成docker镜像。

官方没有提供docker镜像，但在[issues](https://github.com/m-bain/whisperX/issues/909)中可以发现有人使用github的action构建了多种版本的Whisperx镜像。

项目是：[docker-whisperX](https://github.com/jim60105/docker-whisperX)，这位大神写的docker非常好，环境都打包到了镜像，对于不同语音的whsiper模型，有专门的镜像去拉对应模型，比较遗憾的是由于`pyannote/speaker-diarization-3.1`许可证的问题，需要在huggingface同意相关限制，使用huggingface token进行下载。

由于之前说的想要离线使用，所以要在docker-whisperX提供的镜像上加上`pyannote/speaker-diarization-3.1`缓存。

Dockerfile如下：

```dockerfile
# Define optional arguments that indicate the OpenAI Whisper model size and language to use
ARG WHISPER_MODEL=large-v3
ARG LANG=zh

# Get the base WhisperX Docker image (https://github.com/jim60105/docker-whisperX)
FROM ghcr.io/jim60105/whisperx:${WHISPER_MODEL}-${LANG}

# Define the required argument for the huggingface.co token used by Pyannote (diarization/speaker-recognition library)
ARG HUGGING_FACE_TOKEN

# Output argument value for debugging/inspecting
RUN echo "Huggingface.co token: ${HUGGING_FACE_TOKEN}"

# Ensure the required argument was supplied
# (test -n "") Returns false if the string is zero length
RUN test -n "$HUGGING_FACE_TOKEN" || (echo "HUGGING_FACE_TOKEN argument is required" && false)

# Preload and cache the Pyannote models so that the image can run offline
RUN python3 -c 'from pyannote.audio import Pipeline; pipeline = Pipeline.from_pretrained("pyannote/speaker-diarization-3.1", use_auth_token="'${HUGGING_FACE_TOKEN}'")'
```

打包时注意添加个人的HUGGING_FACE_TOKEN，且确保对应huggingface账户已经同意相关模型的许可。

之后就可以在离线环境下使用了。

效果截图：



## 四、使用API调用

最后一个问题，环境和模型都搭建好了，该怎么进行API调用呢？

由于docker-whisperX的作者说明不会添加API调用到镜像中，指路：[Docker image with Whisperx ASR API endpoint](https://github.com/jim60105/docker-whisperX/issues/39)

所以要自己写一个api服务，下面是使用FastAPI写了一个基本的API：

```python
import os
import tempfile
from contextlib import asynccontextmanager
from typing import Optional

import whisperx
from fastapi import FastAPI, File, UploadFile
from fastapi.responses import JSONResponse


class WhisperProcessor:
    # Class variables for models
    model = None
    model_a = None
    metadata = None
    diarize_model = None
    
    # Configuration
    DEVICE = "cuda"
    BATCH_SIZE = 16
    COMPUTE_TYPE = "float16"
    MODEL_NAME = "large-v3"
    HF_TOKEN = os.getenv("HF_TOKEN", default="hf_xxx")
    DEFAULT_LANGUAGE = "zh"

    @classmethod
    def initialize(cls):
        """Initialize all models when the application starts"""
        # Initialize main model
        if cls.model is None:
            cls.model = whisperx.load_model(cls.MODEL_NAME, cls.DEVICE, compute_type=cls.COMPUTE_TYPE)
        
        # Initialize alignment model with default language
        if cls.model_a is None or cls.metadata is None:
            cls.model_a, cls.metadata = whisperx.load_align_model(
                language_code=cls.DEFAULT_LANGUAGE, 
                device=cls.DEVICE
            )
        
        # Initialize diarization model
        if cls.diarize_model is None:
            cls.diarize_model = whisperx.diarize.DiarizationPipeline(
                use_auth_token=cls.HF_TOKEN, 
                device=cls.DEVICE
            )

    @classmethod
    def process_audio(cls, audio_path: str, min_speakers: Optional[int] = None, max_speakers: Optional[int] = None):
        """Process audio file through the complete pipeline"""
        # 1. Transcribe with whisper
        audio = whisperx.load_audio(audio_path)
        result = cls.model.transcribe(audio, batch_size=cls.BATCH_SIZE)

        # 2. Align whisper output
        result = whisperx.align(
            result["segments"], 
            cls.model_a, 
            cls.metadata, 
            audio, 
            cls.DEVICE, 
            return_char_alignments=False
        )

        # 3. Assign speaker labels
        if min_speakers is not None and max_speakers is not None:
            diarize_segments = cls.diarize_model(
                audio, 
                min_speakers=min_speakers, 
                max_speakers=max_speakers
            )
        else:
            diarize_segments = cls.diarize_model(audio)

        result = whisperx.assign_word_speakers(diarize_segments, result)
        return result

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    WhisperProcessor.initialize()
    yield
    # Shutdown
    pass

# Initialize FastAPI app
app = FastAPI(
    title="Whisper API", 
    description="Audio transcription and speaker diarization API",
    lifespan=lifespan
)

@app.post("/transcribe")
async def transcribe_audio(
    audio_file: UploadFile = File(...),
    min_speakers: Optional[int] = None,
    max_speakers: Optional[int] = None
):
    try:
        # Save uploaded file temporarily
        with tempfile.NamedTemporaryFile(delete=False, suffix=os.path.splitext(audio_file.filename)[1]) as temp_file:
            content = await audio_file.read()
            temp_file.write(content)
            temp_file_path = temp_file.name

        # Process the audio file
        result = WhisperProcessor.process_audio(
            temp_file_path,
            min_speakers=min_speakers,
            max_speakers=max_speakers
        )

        # Clean up temporary file
        os.unlink(temp_file_path)

        return JSONResponse(content=result)

    except Exception as e:
        return JSONResponse(
            status_code=500,
            content={"error": str(e)}
        )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000) 
```

将其命名为`api.py`，后面我在原来的基础上继续打包，注意原作者为了减少镜像体积，安装包之后就把pip删除了，这里重新把pip安装回来，为了添加FastAPI相关依赖。

```dockerfile
# Define optional arguments that indicate the OpenAI Whisper model size and language to use
ARG WHISPER_MODEL=large-v3
ARG LANG=zh

# Get the base WhisperX Docker image (https://github.com/jim60105/docker-whisperX)
FROM ghcr.io/jim60105/whisperx:${WHISPER_MODEL}-${LANG}

# Define the required argument for the huggingface.co token used by Pyannote (diarization/speaker-recognition library)
ARG HUGGING_FACE_TOKEN=

# Output argument value for debugging/inspecting
RUN echo "Huggingface.co token: ${HUGGING_FACE_TOKEN}"

# Ensure the required argument was supplied
# (test -n "") Returns false if the string is zero length
RUN test -n "$HUGGING_FACE_TOKEN" || (echo "HUGGING_FACE_TOKEN argument is required" && false)

# Preload and cache the Pyannote models so that the image can run offline
RUN python3 -c 'from pyannote.audio import Pipeline; pipeline = Pipeline.from_pretrained("pyannote/speaker-diarization-3.1", use_auth_token="'${HUGGING_FACE_TOKEN}'")'

# Add api service
USER root

ADD https://bootstrap.pypa.io/get-pip.py /app/get-pip.py
RUN python3 /app/get-pip.py && rm /app/get-pip.py

RUN pip install fastapi uvicorn python-multipart -i https://mirrors.aliyun.com/pypi/simple

USER 1001

COPY api.py api.py

ENTRYPOINT ["uvicorn", "api:app", "--host", "0.0.0.0", "--port", "8000"]
```

使用docker-compose配置启动服务，调用结果如下：

```yaml
services:
  whisperx-speaker-api:
    image: whisperx-speaker:api
    container_name: whisperx-api
    ports:
      - "8000:8000"
    user: "1001"
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

![WhisperX API测试](/images/WhisperX测试.png)

