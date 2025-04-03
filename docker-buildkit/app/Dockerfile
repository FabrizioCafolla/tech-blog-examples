# syntax=docker/dockerfile:1

FROM python:3.12-slim AS builder

ARG LABEL_TEST
LABEL test=$LABEL_TEST

WORKDIR /app

COPY requirements.txt .

RUN python -m venv .venv && . .venv/bin/activate && pip install --no-cache-dir -r requirements.txt

COPY requirements2.txt .

RUN python -m venv .venv2 && . .venv2/bin/activate && pip install --no-cache-dir -r requirements2.txt

RUN dd if=/dev/urandom of=test-file bs=1G count=1

FROM python:3.12-slim

WORKDIR /app

COPY . .

COPY --from=builder /app/.venv ./.venv

CMD ["python"]
