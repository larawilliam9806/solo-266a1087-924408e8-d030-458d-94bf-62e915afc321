# 水质批次修订发布台 · 一体化镜像
# 单容器即可运行；多容器共用数据卷时由 SQLite 写锁保证发布/恢复互斥。
FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1 \
    WQ_DATA_DIR=/app/data \
    WQ_DOWNSTREAM_DIR=/app/downstream

WORKDIR /app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY wq_publish ./wq_publish
COPY wsgi.py ./

EXPOSE 8000

# waitress 单进程多线程；多实例部署时共享 WQ_DATA_DIR / WQ_DOWNSTREAM_DIR 卷
CMD ["waitress-serve", "--host=0.0.0.0", "--port=8000", "--call", "wsgi:create_app"]
