# Slim Python base image matching the app's runtime requirements
FROM python:3.12-slim

WORKDIR /app

# build-essential + libpq-dev are needed to compile Postgres client bindings
# during pip install; apt-get update is required before install so the
# package index is current in this fresh image layer
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends build-essential libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Copy only the requirements first so Docker can cache this layer
# and skip reinstalling dependencies when only app code changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5153

CMD ["python", "app.py"]
