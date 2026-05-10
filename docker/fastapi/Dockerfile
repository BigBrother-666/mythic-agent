# ============== Stage 1: builder ==============
FROM python:3.12-slim AS builder

ENV PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /build

# torch wheel 源：
#   CPU: https://download.pytorch.org/whl/cpu          (默认)
#   GPU: https://download.pytorch.org/whl/cuxxx        (xxx对应cuda版本)
ARG TORCH_INDEX_URL=https://download.pytorch.org/whl/cpu

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential git curl && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

# pymilvus 2.5 仍 import pkg_resources；显式装 setuptools<81（81+ 已移除 pkg_resources）
RUN pip install --upgrade pip && \
    pip install "setuptools>=75,<81" && \
    pip install --extra-index-url ${TORCH_INDEX_URL} -r requirements.txt

# ============== Stage 2: runtime ==============
FROM python:3.12-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    HF_HOME=/home/app/.cache/huggingface \
    TRANSFORMERS_OFFLINE=0

# 创建非 root 用户
RUN groupadd -g 1000 app && useradd -m -u 1000 -g app app

# 复制依赖
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

WORKDIR /app

# 复制源码（.dockerignore 过滤掉 wiki 与缓存）
COPY app ./app
COPY scripts ./scripts
COPY pyproject.toml ./

# 提前创建好 HF 缓存目录并交给 app；named volume 首次挂载会沿用此所有权，
# 避免 root 拥有挂载点导致非 root 进程写入 Permission denied。
RUN mkdir -p /home/app/.cache/huggingface && \
    chown -R app:app /home/app/.cache /app

USER app

EXPOSE 8000

# uvloop 显式启用
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--loop", "uvloop", "--http", "httptools"]
