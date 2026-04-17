FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY indexer.py .

RUN useradd -r -u 1000 appuser
USER appuser

CMD ["python", "-u", "indexer.py"]
