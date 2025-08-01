# 🚀 EyAy.OS 2.0 - Kapsamlı Kurulum ve Geliştirme Rehberi

## 📋 İçindekiler

1. [Proje Genel Bakış](#proje-genel-bakış)
2. [Sistem Gereksinimleri](#sistem-gereksinimleri)
3. [Hızlı Başlangıç](#hızlı-başlangıç)
4. [Mikroservis Mimarisi](#mikroservis-mimarisi)
5. [Adım Adım Kurulum](#adım-adım-kurulum)
6. [Geliştirme Ortamı](#geliştirme-ortamı)
7. [Production Deployment](#production-deployment)

## 🎯 Proje Genel Bakış

EyAy.OS 2.0, modern mikroservis mimarisi ile geliştirilmiş Türkçe AI asistan platformudur.

### Temel Özellikler
- **Mikroservis Mimarisi**: 6 bağımsız servis
- **Türkçe AI Desteği**: Özel Türkçe NLP modelleri
- **Gerçek Zamanlı İletişim**: WebSocket desteği
- **Ölçeklenebilir Altyapı**: Docker + Kubernetes
- **Modern Tech Stack**: FastAPI, React, PostgreSQL, Redis

### Servis Listesi
1. **Yetkilendirme Servisi** (Port: 8001) - JWT tabanlı kimlik doğrulama
2. **Dil İşleme Servisi** (Port: 8002) - Türkçe NLP ve AI
3. **Konuşma Servisi** (Port: 8003) - Ses tanıma ve sentezi
4. **Hafıza Servisi** (Port: 8004) - Konuşma hafızası ve vektör DB
5. **Sohbet Servisi** (Port: 8005) - Gerçek zamanlı sohbet
6. **Analiz Servisi** (Port: 8006) - Metrikler ve dashboard

## 💻 Sistem Gereksinimleri

### Minimum Gereksinimler
- **CPU**: 4 core (Intel i5 veya AMD Ryzen 5)
- **RAM**: 8 GB
- **Disk**: 50 GB SSD
- **OS**: Ubuntu 20.04+, macOS 12+, Windows 11

### Önerilen Gereksinimler
- **CPU**: 8 core (Intel i7 veya AMD Ryzen 7)
- **RAM**: 16 GB
- **Disk**: 100 GB NVMe SSD
- **GPU**: NVIDIA GTX 1060+ (AI modelleri için)

### Yazılım Gereksinimleri
- **Python**: 3.11+
- **Node.js**: 18+
- **Docker**: 24.0+
- **Docker Compose**: 2.20+
- **Git**: 2.30+

## ⚡ Hızlı Başlangıç

### 1. Repository Klonlama
```bash
git clone https://github.com/barutseref/EyAy.OS.git
cd EyAy.OS
```

### 2. Geliştirme Ortamını Başlatma
```bash
# Docker Compose ile tüm servisleri başlat
docker-compose up -d

# Servislerin durumunu kontrol et
docker-compose ps

# Logları izle
docker-compose logs -f
```

### 3. Frontend Başlatma
```bash
cd frontend/web-app
npm install
npm run dev
```

### 4. Test Etme
```bash
# Sağlık kontrolü
curl http://localhost:8001/saglik  # Yetkilendirme
curl http://localhost:8002/saglik  # Dil İşleme
curl http://localhost:8003/saglik  # Konuşma
curl http://localhost:8004/saglik  # Hafıza
curl http://localhost:8005/saglik  # Sohbet
curl http://localhost:8006/saglik  # Analiz
```

## 🏗️ Mikroservis Mimarisi

```
┌─────────────────────────────────────────────────────────────┐
│                    EyAy.OS 2.0 Architecture                 │
├─────────────────────────────────────────────────────────────┤
│  Frontend Layer                                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│  │   Web App   │ │ Mobile App  │ │Desktop App  │            │
│  │ (React/Next)│ │(React Native│ │ (Electron)  │            │
│  └─────────────┘ └─────────────┘ └─────────────┘            │
├─────────────────────────────────────────────────────────────┤
│  API Gateway (Kong/Traefik)                                 │
├─────────────────────────────────────────────────────────────┤
│  Microservices Layer                                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│  │Yetkilendirme│ │Dil İşleme   │ │  Konuşma    │            │
│  │   :8001     │ │   :8002     │ │   :8003     │            │
│  └─────────────┘ └─────────────┘ └─────────────┘            │ 
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│  │   Hafıza    │ │   Sohbet    │ │   Analiz    │            │
│  │   :8004     │ │   :8005     │ │   :8006     │            │
│  └─────────────┘ └─────────────┘ └─────────────┘            │
├─────────────────────────────────────────────────────────────┤
│  Data Layer                                                 │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│  │ PostgreSQL  │ │    Redis    │ │   Qdrant    │            │
│  │ (Ana Veri)  │ │  (Cache)    │ │ (Vector DB) │            │
│  └─────────────┘ └─────────────┘ └─────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

## 📝 Adım Adım Kurulum

### Adım 1: Proje Yapısını Oluşturma

```bash
# Ana dizini oluştur
mkdir EyAy.OS
cd EyAy.OS

# Git repository başlat
git init
git branch -M main

# Proje yapısını oluştur
mkdir -p .github/workflows
mkdir -p hizmetler/{yetkilendirme,dil_isleme,konusma,hafiza,sohbet,analiz}
mkdir -p frontend/{web-app,mobile-app,desktop-app}
mkdir -p infrastructure/{docker,kubernetes,helm}
mkdir -p shared/{schemas,utils,types}
mkdir -p docs tests scripts
```

### Adım 2: Her Servisi Kurma

#### Yetkilendirme Servisi
```bash
cd hizmetler/yetkilendirme

# Poetry ile bağımlılıkları yükle
poetry install

# Alembic ile veritabanı migration
poetry run alembic init migrations
poetry run alembic revision --autogenerate -m "Initial migration"
poetry run alembic upgrade head

# Test et
poetry run uvicorn app.main:app --reload --port 8001
```

## 🔧 Geliştirme Ortamı

### VS Code Konfigürasyonu

```json
// .vscode/settings.json
{
    "python.defaultInterpreterPath": "./hizmetler/yetkilendirme/.venv/bin/python",
    "python.linting.enabled": true,
    "python.linting.pylintEnabled": false,
    "python.linting.flake8Enabled": true,
    "python.formatting.provider": "black",
    "editor.formatOnSave": true,
    "files.exclude": {
        "**/__pycache__": true,
        "**/.pytest_cache": true,
        "**/node_modules": true
    }
}
```

## 🚀 Production Deployment

### Kubernetes Deployment

```yaml
# infrastructure/kubernetes/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: eyayos
```

### Helm Chart

```yaml
# infrastructure/helm/eyayos/Chart.yaml
apiVersion: v2
name: eyayos
description: EyAy.OS Helm Chart
version: 0.1.0
appVersion: "0.1.0"
```

## 📊 İzleme ve Bakım

### Sağlık Kontrolleri

```bash
#!/bin/bash
# scripts/health-check.sh

services=("yetkilendirme:8001" "dil-isleme:8002" "konusma:8003" "hafiza:8004" "sohbet:8005" "analiz:8006")

echo "🔍 EyAy.OS Sağlık Kontrolü"
echo "=========================="

for service in "${services[@]}"; do
    name=$(echo $service | cut -d: -f1)
    port=$(echo $service | cut -d: -f2)
    
    if curl -s "http://localhost:$port/saglik" > /dev/null; then
        echo "✅ $name servisi çalışıyor"
    else
        echo "❌ $name servisi çalışmıyor"
    fi
done
```

## 🎯 Sonraki Adımlar

### Faz 1: Temel Kurulum (1-2 Hafta)
- [x] Proje yapısı oluşturma
- [x] Docker Compose konfigürasyonu
- [x] Temel servis template'leri
- [ ] CI/CD pipeline kurulumu

### Faz 2: Servis Geliştirme (3-4 Hafta)
- [ ] Yetkilendirme servisi tamamlama
- [ ] Dil işleme servisi AI entegrasyonu
- [ ] Konuşma servisi Whisper entegrasyonu
- [ ] Hafıza servisi vektör DB entegrasyonu

### Faz 3: Frontend Geliştirme (2-3 Hafta)
- [ ] React/Next.js web uygulaması
- [ ] WebSocket entegrasyonu
- [ ] Responsive tasarım
- [ ] PWA özellikleri

### Faz 4: Production Hazırlık (1-2 Hafta)
- [ ] Kubernetes deployment
- [ ] Monitoring ve logging
- [ ] Security hardening
- [ ] Performance optimization

## 📞 Destek ve Katkı

### Geliştirici Topluluğu
- **GitHub**: https://github.com/barutseref/EyAy.OS
- **Discord**: EyAy.OS Geliştirici Sunucusu
- **Email**: barutseref@barutben.com

### Katkıda Bulunma
1. Fork yapın
2. Feature branch oluşturun (`git checkout -b feature/yeni-ozellik`)
3. Değişikliklerinizi commit edin (`git commit -am 'Yeni özellik eklendi'`)
4. Branch'inizi push edin (`git push origin feature/yeni-ozellik`)
5. Pull Request oluşturun
