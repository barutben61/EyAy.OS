# Dil İşleme Servisi - Turkish Natural Language Processing

## 🎯 Özellikler

- Türkçe doğal dil anlama
- Niyet sınıflandırması (Intent Classification)
- Duygu analizi (Sentiment Analysis)
- Varlık tanıma (Named Entity Recognition)
- Metin gömme (Text Embeddings)
- Konuşma bağlamı yönetimi
- Türkçe dilbilgisi analizi

## 📁 Dizin Yapısı

```
hizmetler/dil_isleme/
├── app/
│   ├── __init__.py
│   ├── eyay.py              # FastAPI app
│   ├── ayar.py              # Configuration
│   ├── modeller/
│   │   ├── __init__.py
│   │   ├── niyet.py         # Intent model
│   │   ├── duygu.py         # Sentiment model
│   │   └── varlik.py        # Entity model
│   ├── semalar/
│   │   ├── __init__.py
│   │   ├── dil_analiz.py    # NLP schemas
│   │   └── yanit.py         # Response schemas
│   ├── api/
│   │   ├── __init__.py
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── analiz.py    # Analysis endpoints
│   │       ├── niyet.py     # Intent endpoints
│   │       └── gomme.py     # Embeddings endpoints
│   ├── cekirdek/
│   │   ├── __init__.py
│   │   ├── dil_motoru.py    # NLP engine
│   │   └── turkce_modeller.py # Turkish models
│   └── hizmetler/
│       ├── __init__.py
│       ├── niyet_siniflandirici.py
│       ├── duygu_analizci.py
│       ├── varlik_cikartici.py
│       └── gomme_hizmeti.py
├── modeller/             # ML model files
│   ├── niyet/
│   ├── duygu/
│   └── varlik/
├── tests/
├── Dockerfile
├── pyproject.toml
└── README.md
```

## 🔧 Temel Dosyalar

### pyproject.toml
```toml
[tool.poetry]
name = "eyay-dil-isleme-servisi"
version = "0.1.0"
description = "EyAy.OS Türkçe Doğal Dil İşleme Mikroservisi"
authors = ["Barut Şeref <barutseref@barutben.com>"]

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.104.0"
uvicorn = {extras = ["standard"], version = "^0.24.0"}
transformers = "^4.35.0"
torch = "^2.1.0"
sentence-transformers = "^2.2.0"
spacy = "^3.7.0"
datasets = "^2.14.0"
scikit-learn = "^1.3.0"
numpy = "^1.24.0"
pandas = "^2.1.0"
langdetect = "^1.0.9"
textblob = "^0.17.1"
zemberek-python = "^0.1.0"
turkish-stemmer = "^1.0.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"
pytest-asyncio = "^0.21.0"
httpx = "^0.25.0"
jupyter = "^1.0.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### app/cekirdek/turkce_modeller.py
```python
from typing import Dict, List, Optional, Any
import torch
from transformers import (
    AutoTokenizer, 
    AutoModelForSequenceClassification,
    pipeline
)
from sentence_transformers import SentenceTransformer
import logging

logger = logging.getLogger(__name__)

class TurkceNLPModelleri:
    """Türkçe NLP modelleri yöneticisi"""
    
    def __init__(self):
        self.modeller = {}
        self.tokenizer_lar = {}
        self.pipeline_lar = {}
        self._modelleri_yukle()
    
    def _modelleri_yukle(self):
        """Tüm Türkçe NLP modellerini yükle"""
        try:
            # Duygu Analizi Modeli
            self.modeller['duygu'] = AutoModelForSequenceClassification.from_pretrained(
                "savasy/bert-base-turkish-sentiment-cased"
            )
            self.tokenizer_lar['duygu'] = AutoTokenizer.from_pretrained(
                "savasy/bert-base-turkish-sentiment-cased"
            )
            
            # Niyet Sınıflandırma Modeli (özel eğitilmiş)
            # Bu model özel olarak Türkçe intent'ler için eğitilecek
            try:
                self.modeller['niyet'] = AutoModelForSequenceClassification.from_pretrained(
                    "./modeller/niyet/turkce-intent-model"
                )
                self.tokenizer_lar['niyet'] = AutoTokenizer.from_pretrained(
                    "./modeller/niyet/turkce-intent-model"
                )
            except:
                logger.warning("Özel niyet modeli bulunamadı, varsayılan model kullanılacak")
                self.modeller['niyet'] = self.modeller['duygu']  # Geçici çözüm
                self.tokenizer_lar['niyet'] = self.tokenizer_lar['duygu']
            
            # Gömme Modeli
            self.modeller['gomme'] = SentenceTransformer(
                'emrecan/bert-base-turkish-cased-mean-nli-stsb-tr'
            )
            
            # Varlık Tanıma Pipeline
            self.pipeline_lar['varlik'] = pipeline(
                "ner",
                model="savasy/bert-base-turkish-ner-cased",
                tokenizer="savasy/bert-base-turkish-ner-cased",
                aggregation_strategy="simple"
            )
            
            # Metin Sınıflandırma Pipeline
            self.pipeline_lar['siniflandirma'] = pipeline(
                "text-classification",
                model="savasy/bert-base-turkish-sentiment-cased",
                tokenizer="savasy/bert-base-turkish-sentiment-cased"
            )
            
            logger.info("✅ Türkçe NLP modelleri başarıyla yüklendi")
            
        except Exception as e:
            logger.error(f"❌ Model yükleme hatası: {e}")
            raise
    
    def duygu_analiz_et(self, metin: str) -> Dict[str, Any]:
        """Türkçe metnin duygusunu analiz et"""
        try:
            sonuc = self.pipeline_lar['siniflandirma'](metin)
            
            # Türkçe etiketlere çevir
            duygu_map = {
                'POSITIVE': 'pozitif',
                'NEGATIVE': 'negatif',
                'NEUTRAL': 'notr'
            }
            
            return {
                'duygu': duygu_map.get(sonuc[0]['label'], sonuc[0]['label']),
                'guven': round(sonuc[0]['score'], 3),
                'metin': metin,
                'detay': {
                    'orijinal_etiket': sonuc[0]['label'],
                    'skor': sonuc[0]['score']
                }
            }
        except Exception as e:
            logger.error(f"Duygu analizi hatası: {e}")
            return {'hata': str(e)}
    
    def varliklari_cikart(self, metin: str) -> List[Dict[str, Any]]:
        """Türkçe metinden varlıkları çıkart"""
        try:
            varliklar = self.pipeline_lar['varlik'](metin)
            
            # Türkçe etiketlere çevir
            etiket_map = {
                'PER': 'kisi',
                'LOC': 'yer',
                'ORG': 'kurum',
                'MISC': 'diger'
            }
            
            sonuc = []
            for varlik in varliklar:
                sonuc.append({
                    'metin': varlik['word'],
                    'etiket': etiket_map.get(varlik['entity_group'], varlik['entity_group']),
                    'guven': round(varlik['score'], 3),
                    'baslangic': varlik.get('start', 0),
                    'bitis': varlik.get('end', 0)
                })
            
            return sonuc
            
        except Exception as e:
            logger.error(f"Varlık çıkarma hatası: {e}")
            return []
    
    def niyet_siniflandir(self, metin: str) -> Dict[str, Any]:
        """Metnin niyetini sınıflandır"""
        try:
            # Basit kural tabanlı niyet tespiti (geliştirilecek)
            metin_kucuk = metin.lower().strip()
            
            # Türkçe niyet kategorileri
            niyet_kurallari = {
                'selamlama': ['merhaba', 'selam', 'günaydın', 'iyi akşamlar', 'hey', 'sa'],
                'vedalaşma': ['görüşürüz', 'bay bay', 'hoşça kal', 'güle güle', 'as'],
                'soru': ['ne', 'nasıl', 'nerede', 'ne zaman', 'kim', 'neden', 'kaç'],
                'komut': ['aç', 'kapat', 'çal', 'durdur', 'başlat', 'yap', 'getir'],
                'teşekkür': ['teşekkür', 'sağol', 'mersi', 'thanks'],
                'şikayet': ['kötü', 'berbat', 'çalışmıyor', 'hata', 'problem'],
                'övgü': ['harika', 'mükemmel', 'süper', 'çok iyi', 'bravo']
            }
            
            # Soru işareti kontrolü
            if '?' in metin:
                return {
                    'niyet': 'soru',
                    'guven': 0.9,
                    'metin': metin,
                    'detay': 'Soru işareti tespit edildi'
                }
            
            # Kural tabanlı kontrol
            for niyet, kelimeler in niyet_kurallari.items():
                for kelime in kelimeler:
                    if kelime in metin_kucuk:
                        return {
                            'niyet': niyet,
                            'guven': 0.8,
                            'metin': metin,
                            'detay': f'"{kelime}" kelimesi tespit edildi'
                        }
            
            # Varsayılan
            return {
                'niyet': 'genel',
                'guven': 0.5,
                'metin': metin,
                'detay': 'Belirli bir niyet tespit edilemedi'
            }
            
        except Exception as e:
            logger.error(f"Niyet sınıflandırma hatası: {e}")
            return {'hata': str(e)}
    
    def metin_gomme_olustur(self, metin: str) -> List[float]:
        """Metin için gömme vektörü oluştur"""
        try:
            gomme = self.modeller['gomme'].encode(metin)
            return gomme.tolist()
        except Exception as e:
            logger.error(f"Gömme oluşturma hatası: {e}")
            return []
    
    def benzerlik_hesapla(self, metin1: str, metin2: str) -> float:
        """İki metin arasındaki benzerliği hesapla"""
        try:
            gomme1 = self.modeller['gomme'].encode([metin1])
            gomme2 = self.modeller['gomme'].encode([metin2])
            
            from sklearn.metrics.pairwise import cosine_similarity
            benzerlik = cosine_similarity(gomme1, gomme2)[0][0]
            
            return float(benzerlik)
        except Exception as e:
            logger.error(f"Benzerlik hesaplama hatası: {e}")
            return 0.0
    
    def kapsamli_analiz(self, metin: str) -> Dict[str, Any]:
        """Metnin kapsamlı analizi"""
        try:
            return {
                'metin': metin,
                'duygu': self.duygu_analiz_et(metin),
                'niyet': self.niyet_siniflandir(metin),
                'varliklar': self.varliklari_cikart(metin),
                'istatistikler': {
                    'karakter_sayisi': len(metin),
                    'kelime_sayisi': len(metin.split()),
                    'cumle_sayisi': metin.count('.') + metin.count('!') + metin.count('?'),
                    'dil': 'tr'
                }
            }
        except Exception as e:
            logger.error(f"Kapsamlı analiz hatası: {e}")
            return {'hata': str(e)}
```

### app/semalar/dil_analiz.py
```python
from typing import List, Optional, Dict, Any
from pydantic import BaseModel

class DuyguAnalizi(BaseModel):
    duygu: str
    guven: float
    metin: str
    detay: Optional[Dict[str, Any]] = None

class NiyetSiniflandirma(BaseModel):
    niyet: str
    guven: float
    metin: str
    detay: Optional[str] = None

class VarlikTanima(BaseModel):
    metin: str
    etiket: str
    guven: float
    baslangic: int
    bitis: int

class MetinIstatistikleri(BaseModel):
    karakter_sayisi: int
    kelime_sayisi: int
    cumle_sayisi: int
    dil: str

class KapsamliAnaliz(BaseModel):
    metin: str
    duygu: DuyguAnalizi
    niyet: NiyetSiniflandirma
    varliklar: List[VarlikTanima]
    istatistikler: MetinIstatistikleri

class MetinGirdi(BaseModel):
    metin: str
    dil: Optional[str] = "tr"

class BenzerlikGirdi(BaseModel):
    metin1: str
    metin2: str

class BenzerlikSonucu(BaseModel):
    benzerlik: float
    metin1: str
    metin2: str
```

### app/api/v1/analiz.py
```python
from typing import Any, List
from fastapi import APIRouter, HTTPException, Depends
from app.semalar.dil_analiz import (
    MetinGirdi, 
    KapsamliAnaliz, 
    DuyguAnalizi,
    NiyetSiniflandirma,
    BenzerlikGirdi,
    BenzerlikSonucu
)
from app.cekirdek.turkce_modeller import TurkceNLPModelleri

router = APIRouter()

# Global model instance
nlp_modelleri = TurkceNLPModelleri()

@router.post("/kapsamli", response_model=KapsamliAnaliz)
async def kapsamli_analiz(girdi: MetinGirdi) -> Any:
    """Metnin kapsamlı NLP analizi"""
    try:
        if not girdi.metin.strip():
            raise HTTPException(status_code=400, detail="Metin boş olamaz")
        
        sonuc = nlp_modelleri.kapsamli_analiz(girdi.metin)
        
        if 'hata' in sonuc:
            raise HTTPException(status_code=500, detail=sonuc['hata'])
        
        return sonuc
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/duygu", response_model=DuyguAnalizi)
async def duygu_analizi(girdi: MetinGirdi) -> Any:
    """Sadece duygu analizi"""
    try:
        if not girdi.metin.strip():
            raise HTTPException(status_code=400, detail="Metin boş olamaz")
        
        sonuc = nlp_modelleri.duygu_analiz_et(girdi.metin)
        
        if 'hata' in sonuc:
            raise HTTPException(status_code=500, detail=sonuc['hata'])
        
        return sonuc
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/niyet", response_model=NiyetSiniflandirma)
async def niyet_siniflandirma(girdi: MetinGirdi) -> Any:
    """Sadece niyet sınıflandırması"""
    try:
        if not girdi.metin.strip():
            raise HTTPException(status_code=400, detail="Metin boş olamaz")
        
        sonuc = nlp_modelleri.niyet_siniflandir(girdi.metin)
        
        if 'hata' in sonuc:
            raise HTTPException(status_code=500, detail=sonuc['hata'])
        
        return sonuc
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/varliklar")
async def varlik_tanima(girdi: MetinGirdi) -> Any:
    """Varlık tanıma"""
    try:
        if not girdi.metin.strip():
            raise HTTPException(status_code=400, detail="Metin boş olamaz")
        
        varliklar = nlp_modelleri.varliklari_cikart(girdi.metin)
        
        return {
            "metin": girdi.metin,
            "varliklar": varliklar,
            "toplam": len(varliklar)
        }
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/benzerlik", response_model=BenzerlikSonucu)
async def benzerlik_hesapla(girdi: BenzerlikGirdi) -> Any:
    """İki metin arasındaki benzerlik"""
    try:
        if not girdi.metin1.strip() or not girdi.metin2.strip():
            raise HTTPException(status_code=400, detail="Metinler boş olamaz")
        
        benzerlik = nlp_modelleri.benzerlik_hesapla(girdi.metin1, girdi.metin2)
        
        return {
            "benzerlik": benzerlik,
            "metin1": girdi.metin1,
            "metin2": girdi.metin2
        }
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/gomme")
async def metin_gomme(girdi: MetinGirdi) -> Any:
    """Metin gömme vektörü"""
    try:
        if not girdi.metin.strip():
            raise HTTPException(status_code=400, detail="Metin boş olamaz")
        
        gomme = nlp_modelleri.metin_gomme_olustur(girdi.metin)
        
        return {
            "metin": girdi.metin,
            "gomme": gomme,
            "boyut": len(gomme)
        }
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### app/eyay.py
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.api.v1 import analiz
from app.ayar import ayarlar

app = FastAPI(
    title="EyAy.OS Dil İşleme Servisi",
    description="Türkçe doğal dil işleme mikroservisi",
    version="0.1.0",
    docs_url="/docs" if ayarlar.DEBUG else None,
    redoc_url="/redoc" if ayarlar.DEBUG else None,
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=ayarlar.ALLOWED_HOSTS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Router'ları dahil et
app.include_router(
    analiz.router, 
    prefix="/api/v1/analiz", 
    tags=["dil-analizi"]
)

@app.get("/saglik")
async def saglik_kontrolu():
    return {"durum": "saglikli", "servis": "dil-isleme-servisi"}

@app.get("/")
async def ana_sayfa():
    return {
        "mesaj": "EyAy.OS Dil İşleme Servisi", 
        "surum": "0.1.0",
        "desteklenen_dil": "Türkçe"
    }

@app.get("/yetenekler")
async def yetenekler():
    return {
        "duygu_analizi": True,
        "niyet_siniflandirma": True,
        "varlik_tanima": True,
        "metin_gomme": True,
        "benzerlik_hesaplama": True,
        "desteklenen_dil": "tr"
    }
```

### Dockerfile
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Sistem bağımlılıklarını yükle
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    wget \
    && rm -rf /var/lib/apt/lists/*

# Poetry yükle
RUN pip install poetry

# Poetry dosyalarını kopyala
COPY pyproject.toml poetry.lock* ./

# Poetry yapılandır
RUN poetry config virtualenvs.create false

# Bağımlılıkları yükle
RUN poetry install --no-dev

# Modelleri önceden indir (opsiyonel)
RUN python -c "from transformers import AutoTokenizer, AutoModelForSequenceClassification; AutoTokenizer.from_pretrained('savasy/bert-base-turkish-sentiment-cased'); AutoModelForSequenceClassification.from_pretrained('savasy/bert-base-turkish-sentiment-cased')"

# Uygulamayı kopyala
COPY ./app /app/app
COPY ./modeller /app/modeller

# Port aç
EXPOSE 8002

# Uygulamayı çalıştır
CMD ["uvicorn", "app.eyay:app", "--host", "0.0.0.0", "--port", "8002"]
```

## 🧪 Test Örnekleri

### tests/test_dil_analiz.py
```python
import pytest
from httpx import AsyncClient
from app.eyay import app

@pytest.mark.asyncio
async def test_duygu_analizi():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.post(
            "/api/v1/analiz/duygu",
            json={"metin": "Bu harika bir gün!"}
        )
    assert response.status_code == 200
    data = response.json()
    assert "duygu" in data
    assert "guven" in data

@pytest.mark.asyncio
async def test_niyet_siniflandirma():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.post(
            "/api/v1/analiz/niyet",
            json={"metin": "Merhaba, nasılsın?"}
        )
    assert response.status_code == 200
    data = response.json()
    assert data["niyet"] == "selamlama"

@pytest.mark.asyncio
async def test_kapsamli_analiz():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.post(
            "/api/v1/analiz/kapsamli",
            json={"metin": "Merhaba Ahmet, bugün İstanbul'da hava çok güzel!"}
        )
    assert response.status_code == 200
    data = response.json()
    assert "duygu" in data
    assert "niyet" in data
    assert "varliklar" in data
    assert "istatistikler" in data
```

## 🚀 Çalıştırma

```bash
# Bağımlılıkları yükle
cd hizmetler/dil_isleme
poetry install

# Development server başlat
poetry run uvicorn app.eyay:app --reload --port 8002

# Testleri çalıştır
poetry run pytest
```

## 📋 API Endpoints

- `POST /api/v1/analiz/kapsamli` - Kapsamlı metin analizi
- `POST /api/v1/analiz/duygu` - Duygu analizi
- `POST /api/v1/analiz/niyet` - Niyet sınıflandırması
- `POST /api/v1/analiz/varliklar` - Varlık tanıma
- `POST /api/v1/analiz/benzerlik` - Metin benzerliği
- `POST /api/v1/analiz/gomme` - Metin gömme vektörü
- `GET /yetenekler` - Servis yetenekleri
- `GET /saglik` - Sağlık kontrolü

## 🎯 Türkçe Özelleştirmeler

- Türkçe karakter desteği
- Türkçe dilbilgisi kuralları
- Türkçe duygu ifadeleri
- Türkçe varlık tipleri
- Türkçe niyet kategorileri
