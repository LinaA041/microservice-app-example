FROM python:3.10-slim

WORKDIR /app


COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt


COPY main.py .


ENV REDIS_HOST=redis-service \
    REDIS_PORT=6379 \
    REDIS_CHANNEL=log_channel \
    PYTHONUNBUFFERED=1



CMD ["python3", "main.py"]
