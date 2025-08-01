FROM python:3.11-slim

WORKDIR /app

# Sistem bağımlılıklarını yükle
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Poetry'yi yükle
RUN pip install poetry

# Poetry konfigürasyonu
RUN poetry config virtualenvs.create false

# Bağımlılık dosyalarını kopyala
COPY pyproject.toml poetry.lock* ./

# Bağımlılıkları yükle
RUN poetry install --no-dev

# Uygulama kodunu kopyala
COPY . .

# Port'u aç
EXPOSE 8001

# Uygulamayı başlat
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001"]
