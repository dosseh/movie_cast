FROM python:3.8-slim

WORKDIR /app

COPY ./requirements.txt /app/requirements.txt

RUN apt-get update \
    && apt-get install -y gcc libpq-dev python3-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN pip install -r /app/requirements.txt \
    && rm -rf /root/.cache/pip

COPY . /app/
