FROM python:3.9-slim

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

WORKDIR /app

RUN apt update && apt install -y --no-install-recommends \
    gcc \
    python3-dev \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ /app/src/

WORKDIR /app/src

EXPOSE 8000

CMD ["gunicorn", "webapp.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4", "--threads", "2"]