# Konuşma Servisi - Speech Processing Service

## 🎯 Özellikler

- Türkçe ses tanıma (Speech-to-Text)
- Türkçe ses sentezi (Text-to-Speech)
- Gerçek zamanlı ses işleme
- Çoklu ses motoru desteği
- Ses kalitesi optimizasyonu
- Gürültü filtreleme
- Ses komut tanıma

## 📁 Dizin Yapısı

```
hizmetler/konusma/
├── app/
│   ├── __init__.py
│   ├── eyay.py              # FastAPI app
│   ├── ayar.py              # Configuration
│   ├── modeller/
│   │   ├── __init__.py
│   │   ├── ses_tanima.py    # STT model
│   │   └── ses_sentez.py    # TTS model
│   ├── semalar/
│   │   ├── __init__.py
│   │   ├── ses_girdi.py     # Audio input schemas
│   │   └── ses_cikti.py     # Audio output schemas
│   ├── api/
│   │   ├── __init__.py
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── tanima.py    # STT endpoints
│   │       ├── sentez.py    # TTS endpoints
│   │       └── akis.py      # Streaming endpoints
│   ├── cekirdek/
│   │   ├── __init__.py
│   │   ├── ses_motoru.py    # Speech engine
│   │   ├── ses_isleme.py    # Audio processing
│   │   └── whisper_entegre.py # Whisper integration
│   └── hizmetler/
│       ├── __init__.py
│       ├── ses_tanima_hizmeti.py
│       ├── ses_sentez_hizmeti.py
│       └── ses_isleme_hizmeti.py
├── modeller/             # Speech model files
│   ├── whisper/
│   ├── tts/
│   └── ses_komutlar/
├── tests/
├── Dockerfile
├── pyproject.toml
└── README.md
```

## 🔧 Temel Dosyalar

### pyproject.toml
```toml
[tool.poetry]
name = "eyay-konusma-servisi"
version = "0.1.0"
description = "EyAy.OS Konuşma İşleme Mikroservisi"
authors = ["Barut Şeref <barutseref@barutben.com>"]

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.104.0"
uvicorn = {extras = ["standard"], version = "^0.24.0"}
openai-whisper = "^20231117"
torch = "^2.1.0"
torchaudio = "^2.1.0"
pydub = "^0.25.1"
soundfile = "^0.12.1"
librosa = "^0.10.1"
numpy = "^1.24.0"
scipy = "^1.11.0"
webrtcvad = "^2.0.10"
pyaudio = "^0.2.11"
TTS = "^0.20.0"
gTTS = "^2.4.0"
pyttsx3 = "^2.90"
azure-cognitiveservices-speech = "^1.34.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"
pytest-asyncio = "^0.21.0"
httpx = "^0.25.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### app/cekirdek/ses_motoru.py
```python
import logging
import asyncio
import io
import tempfile
import os
from typing import Dict, Any, Optional, List, Union
from pathlib import Path

import whisper
import torch
import torchaudio
import soundfile as sf
import numpy as np
from pydub import AudioSegment
from TTS.api import TTS
from gtts import gTTS
import pyttsx3

logger = logging.getLogger(__name__)

class SesMotoru:
    """Ana ses işleme motoru"""
    
    def __init__(self, ayarlar: Dict[str, Any]):
        self.ayarlar = ayarlar
        self.whisper_model = None
        self.tts_model = None
        self.pyttsx3_engine = None
        self._motorlari_baslat()
    
    def _motorlari_baslat(self):
        """Ses motorlarını başlat"""
        try:
            # Whisper modelini yükle
            model_boyutu = self.ayarlar.get('whisper_model', 'base')
            logger.info(f"Whisper modeli yükleniyor: {model_boyutu}")
            self.whisper_model = whisper.load_model(model_boyutu)
            
            # TTS modelini yükle
            try:
                self.tts_model = TTS(model_name="tts_models/tr/common-voice/glow-tts")
                logger.info("Coqui TTS modeli yüklendi")
            except Exception as e:
                logger.warning(f"Coqui TTS yüklenemedi: {e}")
            
            # pyttsx3 motorunu başlat
            try:
                self.pyttsx3_engine = pyttsx3.init()
                # Türkçe ses ayarları
                voices = self.pyttsx3_engine.getProperty('voices')
                for voice in voices:
                    if 'turkish' in voice.name.lower() or 'tr' in voice.id.lower():
                        self.pyttsx3_engine.setProperty('voice', voice.id)
                        break
                
                # Konuşma hızı ve ses seviyesi
                self.pyttsx3_engine.setProperty('rate', 150)
                self.pyttsx3_engine.setProperty('volume', 0.8)
                logger.info("pyttsx3 motoru başlatıldı")
            except Exception as e:
                logger.warning(f"pyttsx3 başlatılamadı: {e}")
            
            logger.info("✅ Ses motorları başarıyla başlatıldı")
            
        except Exception as e:
            logger.error(f"❌ Ses motoru başlatma hatası: {e}")
            raise
    
    async def ses_tanima_yap(self, ses_verisi: bytes, format: str = "wav") -> Dict[str, Any]:
        """Ses verisini metne çevir"""
        try:
            # Geçici dosya oluştur
            with tempfile.NamedTemporaryFile(suffix=f".{format}", delete=False) as tmp_file:
                tmp_file.write(ses_verisi)
                tmp_file_path = tmp_file.name
            
            try:
                # Whisper ile tanıma
                sonuc = self.whisper_model.transcribe(
                    tmp_file_path,
                    language="tr",
                    task="transcribe"
                )
                
                return {
                    "metin": sonuc["text"].strip(),
                    "dil": sonuc.get("language", "tr"),
                    "guven": self._ortalama_guven_hesapla(sonuc.get("segments", [])),
                    "segmentler": sonuc.get("segments", []),
                    "durum": "basarili"
                }
                
            finally:
                # Geçici dosyayı sil
                os.unlink(tmp_file_path)
                
        except Exception as e:
            logger.error(f"Ses tanıma hatası: {e}")
            return {
                "hata": str(e),
                "durum": "hata"
            }
    
    def _ortalama_guven_hesapla(self, segmentler: List[Dict]) -> float:
        """Segmentlerden ortalama güven skoru hesapla"""
        if not segmentler:
            return 0.0
        
        toplam_guven = 0.0
        segment_sayisi = 0
        
        for segment in segmentler:
            if 'avg_logprob' in segment:
                # Log probability'yi güven skoruna çevir
                guven = np.exp(segment['avg_logprob'])
                toplam_guven += guven
                segment_sayisi += 1
        
        return toplam_guven / segment_sayisi if segment_sayisi > 0 else 0.0
    
    async def ses_sentez_yap(self, metin: str, motor: str = "auto") -> Dict[str, Any]:
        """Metni sese çevir"""
        try:
            if motor == "auto":
                motor = self._en_iyi_tts_motoru_sec()
            
            if motor == "coqui" and self.tts_model:
                return await self._coqui_tts(metin)
            elif motor == "gtts":
                return await self._gtts_sentez(metin)
            elif motor == "pyttsx3" and self.pyttsx3_engine:
                return await self._pyttsx3_sentez(metin)
            else:
                # Varsayılan olarak gTTS kullan
                return await self._gtts_sentez(metin)
                
        except Exception as e:
            logger.error(f"Ses sentezi hatası: {e}")
            return {
                "hata": str(e),
                "durum": "hata"
            }
    
    def _en_iyi_tts_motoru_sec(self) -> str:
        """Mevcut motorlar arasından en iyisini seç"""
        if self.tts_model:
            return "coqui"
        elif self.pyttsx3_engine:
            return "pyttsx3"
        else:
            return "gtts"
    
    async def _coqui_tts(self, metin: str) -> Dict[str, Any]:
        """Coqui TTS ile ses sentezi"""
        try:
            with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as tmp_file:
                self.tts_model.tts_to_file(text=metin, file_path=tmp_file.name)
                
                # Ses dosyasını oku
                with open(tmp_file.name, 'rb') as f:
                    ses_verisi = f.read()
                
                os.unlink(tmp_file.name)
                
                return {
                    "ses_verisi": ses_verisi,
                    "format": "wav",
                    "motor": "coqui",
                    "metin": metin,
                    "durum": "basarili"
                }
                
        except Exception as e:
            logger.error(f"Coqui TTS hatası: {e}")
            raise
    
    async def _gtts_sentez(self, metin: str) -> Dict[str, Any]:
        """Google TTS ile ses sentezi"""
        try:
            tts = gTTS(text=metin, lang='tr', slow=False)
            
            with tempfile.NamedTemporaryFile(suffix=".mp3", delete=False) as tmp_file:
                tts.save(tmp_file.name)
                
                # MP3'ü WAV'a çevir
                audio = AudioSegment.from_mp3(tmp_file.name)
                wav_data = io.BytesIO()
                audio.export(wav_data, format="wav")
                
                os.unlink(tmp_file.name)
                
                return {
                    "ses_verisi": wav_data.getvalue(),
                    "format": "wav",
                    "motor": "gtts",
                    "metin": metin,
                    "durum": "basarili"
                }
                
        except Exception as e:
            logger.error(f"gTTS hatası: {e}")
            raise
    
    async def _pyttsx3_sentez(self, metin: str) -> Dict[str, Any]:
        """pyttsx3 ile ses sentezi"""
        try:
            with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as tmp_file:
                self.pyttsx3_engine.save_to_file(metin, tmp_file.name)
                self.pyttsx3_engine.runAndWait()
                
                # Ses dosyasını oku
                with open(tmp_file.name, 'rb') as f:
                    ses_verisi = f.read()
                
                os.unlink(tmp_file.name)
                
                return {
                    "ses_verisi": ses_verisi,
                    "format": "wav",
                    "motor": "pyttsx3",
                    "metin": metin,
                    "durum": "basarili"
                }
                
        except Exception as e:
            logger.error(f"pyttsx3 hatası: {e}")
            raise
    
    def ses_kalitesi_analiz_et(self, ses_verisi: bytes) -> Dict[str, Any]:
        """Ses kalitesini analiz et"""
        try:
            # Geçici dosya oluştur
            with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as tmp_file:
                tmp_file.write(ses_verisi)
                tmp_file_path = tmp_file.name
            
            try:
                # Ses dosyasını yükle
                y, sr = sf.read(tmp_file_path)
                
                # Temel ses analizi
                rms = np.sqrt(np.mean(y**2))  # RMS (ses seviyesi)
                max_amplitude = np.max(np.abs(y))
                zero_crossing_rate = np.mean(np.abs(np.diff(np.sign(y))))
                
                # Spektral analiz
                fft = np.fft.fft(y)
                freqs = np.fft.fftfreq(len(fft), 1/sr)
                magnitude = np.abs(fft)
                
                # Dominant frekans
                dominant_freq = freqs[np.argmax(magnitude[:len(magnitude)//2])]
                
                return {
                    "ses_seviyesi": float(rms),
                    "maksimum_genlik": float(max_amplitude),
                    "sifir_gecis_orani": float(zero_crossing_rate),
                    "dominant_frekans": float(dominant_freq),
                    "ornekleme_hizi": sr,
                    "sure": len(y) / sr,
                    "kalite_skoru": self._kalite_skoru_hesapla(rms, max_amplitude),
                    "durum": "basarili"
                }
                
            finally:
                os.unlink(tmp_file_path)
                
        except Exception as e:
            logger.error(f"Ses kalitesi analizi hatası: {e}")
            return {
                "hata": str(e),
                "durum": "hata"
            }
    
    def _kalite_skoru_hesapla(self, rms: float, max_amplitude: float) -> float:
        """Ses kalitesi skoru hesapla (0-1 arası)"""
        # Basit kalite skoru hesaplama
        if max_amplitude == 0:
            return 0.0
        
        # RMS/Max oranı (dinamik aralık göstergesi)
        dynamic_range = rms / max_amplitude
        
        # Optimal aralık: 0.1 - 0.3
        if 0.1 <= dynamic_range <= 0.3:
            return 1.0
        elif dynamic_range < 0.1:
            return dynamic_range / 0.1
        else:
            return max(0.0, 1.0 - (dynamic_range - 0.3) / 0.7)
    
    def desteklenen_formatlar(self) -> Dict[str, List[str]]:
        """Desteklenen ses formatları"""
        return {
            "girdi": ["wav", "mp3", "flac", "m4a", "ogg"],
            "cikti": ["wav", "mp3"],
            "ornekleme_hizlari": [8000, 16000, 22050, 44100, 48000],
            "bit_derinlikleri": [16, 24, 32]
        }
    
    def motor_durumu(self) -> Dict[str, Any]:
        """Motor durumu bilgisi"""
        return {
            "whisper": {
                "aktif": self.whisper_model is not None,
                "model": self.ayarlar.get('whisper_model', 'base')
            },
            "coqui_tts": {
                "aktif": self.tts_model is not None
            },
            "pyttsx3": {
                "aktif": self.pyttsx3_engine is not None
            },
            "gtts": {
                "aktif": True  # Her zaman mevcut
            }
        }
```

### app/semalar/ses_girdi.py
```python
from typing import Optional, List
from pydantic import BaseModel, Field
from fastapi import UploadFile

class SesGirdi(BaseModel):
    format: Optional[str] = "wav"
    dil: Optional[str] = "tr"
    kalite: Optional[str] = "yuksek"  # yuksek, orta, dusuk

class SesSentezGirdi(BaseModel):
    metin: str = Field(..., min_length=1, max_length=1000)
    motor: Optional[str] = "auto"  # auto, coqui, gtts, pyttsx3
    hiz: Optional[float] = Field(1.0, ge=0.5, le=2.0)
    ses_tonu: Optional[str] = "normal"  # normal, resmi, samimi

class SesKaliteAnalizi(BaseModel):
    ses_seviyesi: float
    maksimum_genlik: float
    sifir_gecis_orani: float
    dominant_frekans: float
    ornekleme_hizi: int
    sure: float
    kalite_skoru: float
    durum: str

class SesMotorDurumu(BaseModel):
    whisper: dict
    coqui_tts: dict
    pyttsx3: dict
    gtts: dict
```

### app/api/v1/tanima.py
```python
from typing import Any
from fastapi import APIRouter, UploadFile, File, HTTPException, Form
from fastapi.responses import JSONResponse

from app.cekirdek.ses_motoru import SesMotoru
from app.semalar.ses_girdi import SesGirdi, SesKaliteAnalizi
from app.ayar import ayarlar

router = APIRouter()

# Global ses motoru instance
ses_motoru = SesMotoru(ayarlar.SES_AYARLARI)

@router.post("/dosya")
async def ses_dosyasi_tanima(
    dosya: UploadFile = File(...),
    dil: str = Form("tr"),
    kalite: str = Form("yuksek")
) -> Any:
    """Ses dosyasını metne çevir"""
    try:
        # Dosya formatı kontrolü
        if not dosya.filename.lower().endswith(('.wav', '.mp3', '.flac', '.m4a', '.ogg')):
            raise HTTPException(
                status_code=400, 
                detail="Desteklenmeyen dosya formatı"
            )
        
        # Dosya boyutu kontrolü (10MB limit)
        dosya_verisi = await dosya.read()
        if len(dosya_verisi) > 10 * 1024 * 1024:
            raise HTTPException(
                status_code=400,
                detail="Dosya boyutu çok büyük (max 10MB)"
            )
        
        # Ses tanıma
        sonuc = await ses_motoru.ses_tanima_yap(
            dosya_verisi, 
            format=dosya.filename.split('.')[-1]
        )
        
        if sonuc.get("durum") == "hata":
            raise HTTPException(status_code=500, detail=sonuc.get("hata"))
        
        return {
            "dosya_adi": dosya.filename,
            "boyut": len(dosya_verisi),
            "tanima_sonucu": sonuc
        }
        
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/kalite-analizi", response_model=SesKaliteAnalizi)
async def ses_kalitesi_analizi(
    dosya: UploadFile = File(...)
) -> Any:
    """Ses kalitesini analiz et"""
    try:
        if not dosya.filename.lower().endswith(('.wav', '.mp3', '.flac')):
            raise HTTPException(
                status_code=400,
                detail="Desteklenmeyen dosya formatı"
            )
        
        dosya_verisi = await dosya.read()
        
        analiz = ses_motoru.ses_kalitesi_analiz_et(dosya_verisi)
        
        if analiz.get("durum") == "hata":
            raise HTTPException(status_code=500, detail=analiz.get("hata"))
        
        return analiz
        
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/desteklenen-formatlar")
async def desteklenen_formatlar() -> Any:
    """Desteklenen ses formatları"""
    return ses_motoru.desteklenen_formatlar()

@router.get("/motor-durumu")
async def motor_durumu() -> Any:
    """Ses motoru durum bilgisi"""
    return ses_motoru.motor_durumu()
```

### app/api/v1/sentez.py
```python
from typing import Any
from fastapi import APIRouter, HTTPException
from fastapi.responses import Response

from app.cekirdek.ses_motoru import SesMotoru
from app.semalar.ses_girdi import SesSentezGirdi
from app.ayar import ayarlar

router = APIRouter()

# Global ses motoru instance
ses_motoru = SesMotoru(ayarlar.SES_AYARLARI)

@router.post("/metin-to-ses")
async def metin_to_ses(girdi: SesSentezGirdi) -> Any:
    """Metni sese çevir"""
    try:
        if not girdi.metin.strip():
            raise HTTPException(status_code=400, detail="Metin boş olamaz")
        
        # Ses sentezi
        sonuc = await ses_motoru.ses_sentez_yap(
            girdi.metin,
            motor=girdi.motor
        )
        
        if sonuc.get("durum") == "hata":
            raise HTTPException(status_code=500, detail=sonuc.get("hata"))
        
        # Ses verisini binary response olarak döndür
        return Response(
            content=sonuc["ses_verisi"],
            media_type=f"audio/{sonuc['format']}",
            headers={
                "Content-Disposition": f"attachment; filename=sentez.{sonuc['format']}",
                "X-Motor": sonuc["motor"],
                "X-Metin": girdi.metin[:100]  # İlk 100 karakter
            }
        )
        
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/metin-to-ses-json")
async def metin_to_ses_json(girdi: SesSentezGirdi) -> Any:
    """Metni sese çevir (JSON response)"""
    try:
        if not girdi.metin.strip():
            raise HTTPException(status_code=400, detail="Metin boş olamaz")
        
        sonuc = await ses_motoru.ses_sentez_yap(
            girdi.metin,
            motor=girdi.motor
        )
        
        if sonuc.get("durum") == "hata":
            raise HTTPException(status_code=500, detail=sonuc.get("hata"))
        
        # Base64 encode ses verisi
        import base64
        ses_base64 = base64.b64encode(sonuc["ses_verisi"]).decode('utf-8')
        
        return {
            "metin": girdi.metin,
            "motor": sonuc["motor"],
            "format": sonuc["format"],
            "ses_verisi_base64": ses_base64,
            "boyut": len(sonuc["ses_verisi"]),
            "durum": "basarili"
        }
        
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/mevcut-motorlar")
async def mevcut_motorlar() -> Any:
    """Mevcut TTS motorları"""
    durum = ses_motoru.motor_durumu()
    
    motorlar = []
    for motor, bilgi in durum.items():
        if motor != "whisper" and bilgi["aktif"]:
            motorlar.append({
                "ad": motor,
                "aktif": bilgi["aktif"],
                "aciklama": {
                    "coqui_tts": "Yüksek kaliteli yerel TTS",
                    "pyttsx3": "Sistem TTS motoru",
                    "gtts": "Google Text-to-Speech"
                }.get(motor, "Bilinmeyen motor")
            })
    
    return {"motorlar": motorlar}
```

### app/eyay.py
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.api.v1 import tanima, sentez
from app.ayar import ayarlar

app = FastAPI(
    title="EyAy.OS Konuşma Servisi",
    description="Türkçe ses tanıma ve sentezi mikroservisi",
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
    tanima.router, 
    prefix="/api/v1/tanima", 
    tags=["ses-tanima"]
)
app.include_router(
    sentez.router, 
    prefix="/api/v1/sentez", 
    tags=["ses-sentezi"]
)

@app.get("/saglik")
async def saglik_kontrolu():
    return {"durum": "saglikli", "servis": "konusma-servisi"}

@app.get("/")
async def ana_sayfa():
    return {
        "mesaj": "EyAy.OS Konuşma Servisi", 
        "surum": "0.1.0",
        "desteklenen_dil": "Türkçe"
    }

@app.get("/yetenekler")
async def yetenekler():
    return {
        "ses_tanima": True,
        "ses_sentezi": True,
        "kalite_analizi": True,
        "coklu_motor": True,
        "desteklenen_dil": "tr",
        "whisper_model": "base"
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
    ffmpeg \
    portaudio19-dev \
    espeak \
    espeak-data \
    libespeak1 \
    libespeak-dev \
    && rm -rf /var/lib/apt/lists/*

# Poetry yükle
RUN pip install poetry

# Poetry dosyalarını kopyala
COPY pyproject.toml poetry.lock* ./

# Poetry yapılandır
RUN poetry config virtualenvs.create false

# Bağımlılıkları yükle
RUN poetry install --no-dev

# Whisper modelini önceden indir
RUN python -c "import whisper; whisper.load_model('base')"

# Uygulamayı kopyala
COPY ./app /app/app
COPY ./modeller /app/modeller

# Port aç
EXPOSE 8003

# Uygulamayı çalıştır
CMD ["uvicorn", "app.eyay:app", "--host", "0.0.0.0", "--port", "8003"]
```

## 🧪 Test Örnekleri

### tests/test_konusma.py
```python
import pytest
from httpx import AsyncClient
from app.eyay import app
import io

@pytest.mark.asyncio
async def test_ses_sentezi():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.post(
            "/api/v1/sentez/metin-to-ses-json",
            json={
                "metin": "Merhaba, bu bir test mesajıdır.",
                "motor": "gtts"
            }
        )
    assert response.status_code == 200
    data = response.json()
    assert "ses_verisi_base64" in data
    assert data["durum"] == "basarili"

@pytest.mark.asyncio
async def test_motor_durumu():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/api/v1/tanima/motor-durumu")
    assert response.status_code == 200
    data = response.json()
    assert "whisper" in data
    assert "gtts" in data
```

## 🚀 Çalıştırma

```bash
# Bağımlılıkları yükle
cd hizmetler/konusma
poetry install

# Development server başlat
poetry run uvicorn app.eyay:app --reload --port 8003

# Testleri çalıştır
poetry run pytest
```

## 📋 API Endpoints

### Ses Tanıma
- `POST /api/v1/tanima/dosya` - Ses dosyasını metne çevir
- `POST /api/v1/tanima/kalite-analizi` - Ses kalitesi analizi
- `
