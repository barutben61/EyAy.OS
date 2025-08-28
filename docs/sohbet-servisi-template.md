# Sohbet Servisi - Chat Management Service

## 🎯 Özellikler

- Gerçek zamanlı sohbet yönetimi
- WebSocket bağlantıları
- Çoklu kullanıcı desteği
- Sohbet odaları
- Mesaj geçmişi
- Dosya paylaşımı
- Bildirim sistemi

## 📁 Dizin Yapısı

```
hizmetler/sohbet/
├── app/
│   ├── __init__.py
│   ├── eyay.py              # FastAPI app
│   ├── ayar.py              # Configuration
│   ├── veritabani.py        # Database connection
│   ├── modeller/
│   │   ├── __init__.py
│   │   ├── sohbet_odasi.py  # Chat room model
│   │   ├── mesaj.py         # Message model
│   │   └── katilimci.py     # Participant model
│   ├── semalar/
│   │   ├── __init__.py
│   │   ├── sohbet.py        # Chat schemas
│   │   └── mesaj.py         # Message schemas
│   ├── api/
│   │   ├── __init__.py
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── sohbet.py    # Chat endpoints
│   │       ├── mesaj.py     # Message endpoints
│   │       └── websocket.py # WebSocket endpoints
│   ├── cekirdek/
│   │   ├── __init__.py
│   │   ├── sohbet_motoru.py # Chat engine
│   │   ├── websocket_yoneticisi.py # WebSocket manager
│   │   └── bildirim.py      # Notification system
│   └── hizmetler/
│       ├── __init__.py
│       ├── sohbet_hizmeti.py
│       ├── mesaj_hizmeti.py
│       └── bildirim_hizmeti.py
├── tests/
├── Dockerfile
├── pyproject.toml
└── README.md
```

## 🔧 Temel Dosyalar

### pyproject.toml
```toml
[tool.poetry]
name = "eyay-sohbet-servisi"
version = "0.1.0"
description = "EyAy.OS Sohbet Yönetimi Mikroservisi"
authors = ["Barut Şeref <barutseref@barutben.com>"]

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.104.0"
uvicorn = {extras = ["standard"], version = "^0.24.0"}
websockets = "^12.0"
sqlalchemy = "^2.0.0"
alembic = "^1.12.0"
psycopg2-binary = "^2.9.0"
redis = "^5.0.0"
pydantic = "^2.4.0"
python-socketio = "^5.10.0"
aiofiles = "^23.2.1"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"
pytest-asyncio = "^0.21.0"
httpx = "^0.25.0"
websocket-client = "^1.6.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### app/cekirdek/sohbet_motoru.py
```python
import logging
import asyncio
import json
from datetime import datetime
from typing import Dict, Any, List, Optional, Set
from collections import defaultdict

import redis
from sqlalchemy.orm import Session
from fastapi import WebSocket

from app.modeller.sohbet_odasi import SohbetOdasi
from app.modeller.mesaj import Mesaj
from app.veritabani import get_db

logger = logging.getLogger(__name__)

class SohbetMotoru:
    """Ana sohbet yönetim motoru"""
    
    def __init__(self, ayarlar: Dict[str, Any]):
        self.ayarlar = ayarlar
        self.redis_client = None
        self.aktif_baglanti_lar: Dict[str, WebSocket] = {}
        self.oda_katilimci_lari: Dict[str, Set[str]] = defaultdict(set)
        self.kullanici_oda_lari: Dict[str, str] = {}
        self._redis_baglanti_kur()
    
    def _redis_baglanti_kur(self):
        """Redis bağlantısını kur"""
        try:
            self.redis_client = redis.Redis(
                host=self.ayarlar.get('redis_host', 'localhost'),
                port=self.ayarlar.get('redis_port', 6379),
                db=self.ayarlar.get('redis_db', 1),
                decode_responses=True
            )
            logger.info("✅ Sohbet motoru Redis bağlantısı kuruldu")
        except Exception as e:
            logger.error(f"❌ Redis bağlantı hatası: {e}")
            raise
    
    async def kullanici_baglan(
        self, 
        kullanici_id: str, 
        websocket: WebSocket
    ) -> Dict[str, Any]:
        """Kullanıcıyı sohbet sistemine bağla"""
        try:
            await websocket.accept()
            
            # Aktif bağlantıları güncelle
            self.aktif_baglanti_lar[kullanici_id] = websocket
            
            # Redis'e kullanıcı durumunu kaydet
            await self._kullanici_durumu_guncelle(kullanici_id, "online")
            
            # Hoş geldin mesajı gönder
            await self._mesaj_gonder(kullanici_id, {
                "tip": "sistem",
                "mesaj": "Sohbet sistemine bağlandınız",
                "zaman": datetime.now().isoformat()
            })
            
            logger.info(f"Kullanıcı bağlandı: {kullanici_id}")
            
            return {
                "durum": "basarili",
                "kullanici_id": kullanici_id,
                "baglanti_zamani": datetime.now().isoformat()
            }
            
        except Exception as e:
            logger.error(f"Kullanıcı bağlantı hatası: {e}")
            return {"durum": "hata", "mesaj": str(e)}
    
    async def kullanici_ayril(self, kullanici_id: str) -> Dict[str, Any]:
        """Kullanıcıyı sohbet sisteminden ayır"""
        try:
            # Aktif bağlantılardan çıkar
            if kullanici_id in self.aktif_baglanti_lar:
                del self.aktif_baglanti_lar[kullanici_id]
            
            # Odadan çıkar
            if kullanici_id in self.kullanici_oda_lari:
                oda_id = self.kullanici_oda_lari[kullanici_id]
                await self.odadan_ayril(kullanici_id, oda_id)
            
            # Redis'te durumu güncelle
            await self._kullanici_durumu_guncelle(kullanici_id, "offline")
            
            logger.info(f"Kullanıcı ayrıldı: {kullanici_id}")
            
            return {
                "durum": "basarili",
                "kullanici_id": kullanici_id,
                "ayrilma_zamani": datetime.now().isoformat()
            }
            
        except Exception as e:
            logger.error(f"Kullanıcı ayrılma hatası: {e}")
            return {"durum": "hata", "mesaj": str(e)}
    
    async def odaya_katil(
        self, 
        kullanici_id: str, 
        oda_id: str
    ) -> Dict[str, Any]:
        """Kullanıcıyı sohbet odasına ekle"""
        try:
            # Önceki odadan çıkar
            if kullanici_id in self.kullanici_oda_lari:
                eski_oda = self.kullanici_oda_lari[kullanici_id]
                await self.odadan_ayril(kullanici_id, eski_oda)
            
            # Yeni odaya ekle
            self.oda_katilimci_lari[oda_id].add(kullanici_id)
            self.kullanici_oda_lari[kullanici_id] = oda_id
            
            # Redis'e kaydet
            await self._oda_katilimci_guncelle(oda_id, kullanici_id, "katil")
            
            # Odadaki diğer kullanıcılara bildir
            await self._oda_mesaji_gonder(oda_id, {
                "tip": "katilim",
                "kullanici_id": kullanici_id,
                "mesaj": f"Kullanıcı odaya katıldı",
                "zaman": datetime.now().isoformat()
            }, haric_kullanici=kullanici_id)
            
            # Kullanıcıya onay mesajı
            await self._mesaj_gonder(kullanici_id, {
                "tip": "oda_katilim",
                "oda_id": oda_id,
                "mesaj": f"'{oda_id}' odasına katıldınız",
                "katilimci_sayisi": len(self.oda_katilimci_lari[oda_id]),
                "zaman": datetime.now().isoformat()
            })
            
            logger.info(f"Kullanıcı odaya katıldı: {kullanici_id} -> {oda_id}")
            
            return {
                "durum": "basarili",
                "kullanici_id": kullanici_id,
                "oda_id": oda_id,
                "katilimci_sayisi": len(self.oda_katilimci_lari[oda_id])
            }
            
        except Exception as e:
            logger.error(f"Odaya katılma hatası: {e}")
            return {"durum": "hata", "mesaj": str(e)}
    
    async def odadan_ayril(
        self, 
        kullanici_id: str, 
        oda_id: str
    ) -> Dict[str, Any]:
        """Kullanıcıyı sohbet odasından çıkar"""
        try:
            # Odadan çıkar
            if oda_id in self.oda_katilimci_lari:
                self.oda_katilimci_lari[oda_id].discard(kullanici_id)
                
                # Oda boşsa sil
                if not self.oda_katilimci_lari[oda_id]:
                    del self.oda_katilimci_lari[oda_id]
            
            # Kullanıcı oda kaydını sil
            if kullanici_id in self.kullanici_oda_lari:
                del self.kullanici_oda_lari[kullanici_id]
            
            # Redis'i güncelle
            await self._oda_katilimci_guncelle(oda_id, kullanici_id, "ayril")
            
            # Odadaki diğer kullanıcılara bildir
            await self._oda_mesaji_gonder(oda_id, {
                "tip": "ayrilma",
                "kullanici_id": kullanici_id,
                "mesaj": f"Kullanıcı odadan ayrıldı",
                "zaman": datetime.now().isoformat()
            })
            
            logger.info(f"Kullanıcı odadan ayrıldı: {kullanici_id} <- {oda_id}")
            
            return {
                "durum": "basarili",
                "kullanici_id": kullanici_id,
                "oda_id": oda_id
            }
            
        except Exception as e:
            logger.error(f"Odadan ayrılma hatası: {e}")
            return {"durum": "hata", "mesaj": str(e)}
    
    async def mesaj_gonder(
        self,
        gonderen_id: str,
        oda_id: str,
        mesaj_icerigi: str,
        mesaj_tipi: str = "metin"
    ) -> Dict[str, Any]:
        """Sohbet odasına mesaj gönder"""
        try:
            # Kullanıcının odada olup olmadığını kontrol et
            if (kullanici_id not in self.kullanici_oda_lari or 
                self.kullanici_oda_lari[gonderen_id] != oda_id):
                return {
                    "durum": "hata",
                    "mesaj": "Kullanıcı bu odada değil"
                }
            
            # Mesaj objesi oluştur
            mesaj = {
                "id": f"{gonderen_id}_{datetime.now().timestamp()}",
                "gonderen_id": gonderen_id,
                "oda_id": oda_id,
                "icerik": mesaj_icerigi,
                "tip": mesaj_tipi,
                "zaman": datetime.now().isoformat()
            }
            
            # Veritabanına kaydet
            await self._mesaj_veritabani_kaydet(mesaj)
            
            # Odadaki tüm kullanıcılara gönder
            await self._oda_mesaji_gonder(oda_id, {
                "tip": "mesaj",
                "mesaj": mesaj
            })
            
            logger.debug(f"Mesaj gönderildi: {gonderen_id} -> {oda_id}")
            
            return {
                "durum": "basarili",
                "mesaj_id": mesaj["id"],
                "alici_sayisi": len(self.oda_katilimci_lari.get(oda_id, set()))
            }
            
        except Exception as e:
            logger.error(f"Mesaj gönderme hatası: {e}")
            return {"durum": "hata", "mesaj": str(e)}
    
    async def _mesaj_gonder(
        self, 
        kullanici_id: str, 
        mesaj: Dict[str, Any]
    ):
        """Belirli kullanıcıya mesaj gönder"""
        try:
            if kullanici_id in self.aktif_baglanti_lar:
                websocket = self.aktif_baglanti_lar[kullanici_id]
                await websocket.send_text(json.dumps(mesaj))
        except Exception as e:
            logger.error(f"Mesaj gönderme hatası: {e}")
            # Bağlantı kopmuşsa temizle
            if kullanici_id in self.aktif_baglanti_lar:
                del self.aktif_baglanti_lar[kullanici_id]
    
    async def _oda_mesaji_gonder(
        self, 
        oda_id: str, 
        mesaj: Dict[str, Any],
        haric_kullanici: Optional[str] = None
    ):
        """Odadaki tüm kullanıcılara mesaj gönder"""
        try:
            katilimcilar = self.oda_katilimci_lari.get(oda_id, set())
            
            for kullanici_id in katilimcilar:
                if haric_kullanici and kullanici_id == haric_kullanici:
                    continue
                
                await self._mesaj_gonder(kullanici_id, mesaj)
                
        except Exception as e:
            logger.error(f"Oda mesajı gönderme hatası: {e}")
    
    async def _mesaj_veritabani_kaydet(self, mesaj: Dict[str, Any]):
        """Mesajı veritabanına kaydet"""
        try:
            db = next(get_db())
            
            db_mesaj = Mesaj(
                id=mesaj["id"],
                gonderen_id=int(mesaj["gonderen_id"]),
                oda_id=mesaj["oda_id"],
                icerik=mesaj["icerik"],
                tip=mesaj["tip"],
                olusturulma_tarihi=datetime.fromisoformat(mesaj["zaman"])
            )
            
            db.add(db_mesaj)
            db.commit()
            
        except Exception as e:
            logger.error(f"Mesaj veritabanı kaydetme hatası: {e}")
    
    async def _kullanici_durumu_guncelle(
        self, 
        kullanici_id: str, 
        durum: str
    ):
        """Kullanıcı durumunu Redis'te güncelle"""
        try:
            self.redis_client.hset(
                "kullanici_durumlari",
                kullanici_id,
                json.dumps({
                    "durum": durum,
                    "son_guncelleme": datetime.now().isoformat()
                })
            )
        except Exception as e:
            logger.error(f"Kullanıcı durumu güncelleme hatası: {e}")
    
    async def _oda_katilimci_guncelle(
        self, 
        oda_id: str, 
        kullanici_id: str, 
        islem: str
    ):
        """Oda katılımcılarını Redis'te güncelle"""
        try:
            redis_key = f"oda_katilimcilari:{oda_id}"
            
            if islem == "katil":
                self.redis_client.sadd(redis_key, kullanici_id)
            elif islem == "ayril":
                self.redis_client.srem(redis_key, kullanici_id)
                
                # Oda boşsa Redis'ten de sil
                if self.redis_client.scard(redis_key) == 0:
                    self.redis_client.delete(redis_key)
                    
        except Exception as e:
            logger.error(f"Oda katılımcı güncelleme hatası: {e}")
    
    async def oda_gecmisi_getir(
        self, 
        oda_id: str, 
        limit: int = 50
    ) -> List[Dict[str, Any]]:
        """Oda mesaj geçmişini getir"""
        try:
            db = next(get_db())
            
            mesajlar = db.query(Mesaj).filter(
                Mesaj.oda_id == oda_id
            ).order_by(
                Mesaj.olusturulma_tarihi.desc()
            ).limit(limit).all()
            
            sonuc = []
            for mesaj in reversed(mesajlar):  # Eski -> yeni sıralama
                sonuc.append({
                    "id": mesaj.id,
                    "gonderen_id": mesaj.gonderen_id,
                    "icerik": mesaj.icerik,
                    "tip": mesaj.tip,
                    "zaman": mesaj.olusturulma_tarihi.isoformat()
                })
            
            return sonuc
            
        except Exception as e:
            logger.error(f"Oda geçmişi getirme hatası: {e}")
            return []
    
    async def aktif_odalar_listesi(self) -> List[Dict[str, Any]]:
        """Aktif sohbet odalarını listele"""
        try:
            odalar = []
            
            for oda_id, katilimcilar in self.oda_katilimci_lari.items():
                odalar.append({
                    "oda_id": oda_id,
                    "katilimci_sayisi": len(katilimcilar),
                    "katilimcilar": list(katilimcilar),
                    "son_aktivite": datetime.now().isoformat()
                })
            
            return odalar
            
        except Exception as e:
            logger.error(f"Aktif odalar listesi hatası: {e}")
            return []
    
    def durum_bilgisi(self) -> Dict[str, Any]:
        """Sohbet motoru durum bilgisi"""
        try:
            return {
                "aktif_kullanici_sayisi": len(self.aktif_baglanti_lar),
                "aktif_oda_sayisi": len(self.oda_katilimci_lari),
                "toplam_katilimci": sum(len(k) for k in self.oda_katilimci_lari.values()),
                "redis_baglantisi": bool(self.redis_client),
                "son_guncelleme": datetime.now().isoformat()
            }
        except Exception as e:
            logger.error(f"Durum bilgisi hatası: {e}")
            return {"aktif": False, "hata": str(e)}
```

### app/api/v1/websocket.py
```python
from fastapi import APIRouter, WebSocket, WebSocketDisconnect, Depends
import json
import logging

from app.cekirdek.sohbet_motoru import SohbetMotoru
from app.ayar import ayarlar

router = APIRouter()
logger = logging.getLogger(__name__)

# Global sohbet motoru instance
sohbet_motoru = SohbetMotoru(ayarlar.SOHBET_AYARLARI)

@router.websocket("/ws/{kullanici_id}")
async def websocket_endpoint(websocket: WebSocket, kullanici_id: str):
    """WebSocket bağlantı endpoint'i"""
    try:
        # Kullanıcıyı bağla
        await sohbet_motoru.kullanici_baglan(kullanici_id, websocket)
        
        while True:
            # Mesaj bekle
            data = await websocket.receive_text()
            mesaj_data = json.loads(data)
            
            # Mesaj tipine göre işle
            await mesaj_isle(kullanici_id, mesaj_data)
            
    except WebSocketDisconnect:
        # Kullanıcı bağlantısı koptu
        await sohbet_motoru.kullanici_ayril(kullanici_id)
        logger.info(f"WebSocket bağlantısı koptu: {kullanici_id}")
        
    except Exception as e:
        logger.error(f"WebSocket hatası: {e}")
        await sohbet_motoru.kullanici_ayril(kullanici_id)

async def mesaj_isle(kullanici_id: str, mesaj_data: dict):
    """Gelen mesajı işle"""
    try:
        mesaj_tipi = mesaj_data.get("tip")
        
        if mesaj_tipi == "oda_katil":
            oda_id = mesaj_data.get("oda_id")
            await sohbet_motoru.odaya_katil(kullanici_id, oda_id)
            
        elif mesaj_tipi == "oda_ayril":
            oda_id = mesaj_data.get("oda_id")
            await sohbet_motoru.odadan_ayril(kullanici_id, oda_id)
            
        elif mesaj_tipi == "mesaj_gonder":
            oda_id = mesaj_data.get("oda_id")
            icerik = mesaj_data.get("icerik")
            await sohbet_motoru.mesaj_gonder(kullanici_id, oda_id, icerik)
            
        elif mesaj_tipi == "oda_gecmisi":
            oda_id = mesaj_data.get("oda_id")
            limit = mesaj_data.get("limit", 50)
            gecmis = await sohbet_motoru.oda_gecmisi_getir(oda_id, limit)
            
            # Geçmişi kullanıcıya gönder
            await sohbet_motoru._mesaj_gonder(kullanici_id, {
                "tip": "oda_gecmisi",
                "oda_id": oda_id,
                "mesajlar": gecmis
            })
            
    except Exception as e:
        logger.error(f"Mesaj işleme hatası: {e}")
```

### app/eyay.py
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.api.v1 import sohbet, mesaj, websocket
from app.ayar import ayarlar

app = FastAPI(
    title="EyAy.OS Sohbet Servisi",
    description="Gerçek zamanlı sohbet yönetimi mikroservisi",
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
    sohbet.router, 
    prefix="/api/v1/sohbet", 
    tags=["sohbet"]
)
app.include_router(
    mesaj.router, 
    prefix="/api/v1/mesaj", 
    tags=["mesaj"]
)
app.include_router(
    websocket.router, 
    prefix="/api/v1", 
    tags=["websocket"]
)

@app.get("/saglik")
async def saglik_kontrolu():
    return {"durum": "saglikli", "servis": "sohbet-servisi"}

@app.get("/")
async def ana_sayfa():
    return {
        "mesaj": "EyAy.OS Sohbet Servisi", 
        "surum": "0.1.0",
        "websocket_endpoint": "/api/v1/ws/{kullanici_id}"
    }

@app.get("/yetenekler")
async def yetenekler():
    return {
        "websocket_destegi": True,
        "coklu_oda": True,
        "mesaj_gecmisi": True,
        "gercek_zamanli": True,
        "bildirim_sistemi": True
    }
```

## 🚀 Çalıştırma

```bash
# Bağımlılıkları yükle
cd hizmetler/sohbet
poetry install

# Development server başlat
poetry run uvicorn app.eyay:app --reload --port 8004

# WebSocket test
wscat -c ws://localhost:8004/api/v1/ws/kullanici123
```

## 📋 API Endpoints

### REST API
- `GET /api/v1/sohbet/odalar` - Aktif odalar listesi
- `POST /api/v1/sohbet/oda-olustur` - Yeni oda oluştur
- `GET /api/v1/mesaj/gecmis/{oda_id}` - Mesaj geçmişi
- `GET /saglik` - Sağlık kontrolü

### WebSocket
- `WS /api/v1/ws/{kullanici_id}` - WebSocket bağlantısı

### WebSocket Mesaj Tipleri
```json
{
  "tip": "oda_katil",
  "oda_id": "genel"
}

{
  "tip": "mesaj_gonder",
  "oda_id": "genel",
  "icerik": "Merhaba!"
}

{
  "tip": "oda_ayril",
  "oda_id": "genel"
}
