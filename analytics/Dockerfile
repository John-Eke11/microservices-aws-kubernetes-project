FROM python:3.12-slim

WORKDIR /app

# build-essential and libpq-dev are needed to compile psycopg2's C extension
# against the Postgres client library
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends build-essential libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Copy only the requirements file first so this layer is cached and only
# rebuilds when dependencies actually change, not on every code edit
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5153

CMD ["python", "app.py"]
