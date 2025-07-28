# Hafıza Servisi - Memory Management Service

## 🎯 Özellikler

- Kısa vadeli hafıza (Session Memory)
- Uzun vadeli hafıza (Persistent Memory)
- Vektör tabanlı hafıza (Vector Memory)
- Konuşma geçmişi yönetimi
- Bağlamsal hafıza (Contextual Memory)
- Hafıza arama ve filtreleme
- Otomatik hafıza temizleme

## 📁 Dizin Yapısı

```
hizmetler/hafiza/
├── app/
│   ├── __init__.py
│   ├── eyay.py              # FastAPI app
│   ├── ayar.py              # Configuration
│   ├── veritabani.py        # Database connections
│   ├── modeller/
│   │   ├── __init__.py
│   │   ├── konusma.py       # Conversation model
│   │   ├── hafiza_ogesi.py  # Memory item model
│   │   └── vektor.py        # Vector model
│   ├── semalar/
│   │   ├── __init__.py
│   │   ├── hafiza.py        # Memory schemas
│   │   └── arama.py         # Search schemas
│   ├── api/
│   │   ├── __init__.py
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── konusma.py   # Conversation endpoints
│   │       ├── hafiza.py    # Memory endpoints
│   │       └── arama.py     # Search endpoints
│   ├── cekirdek/
│   │   ├── __init__.py
│   │   ├── hafiza_motoru.py # Memory engine
│   │   ├── vektor_db.py     # Vector database
│   │   └── baglamsal.py     # Contextual memory
│   └── hizmetler/
│       ├── __init__.py
│       ├── kisa_hafiza.py
│       ├── uzun_hafiza.py
│       ├── vektor_hafiza.py
│       └── arama_hizmeti.py
├── tests/
├── Dockerfile
├── pyproject.toml
└── README.md
```

## 🔧 Temel Dosyalar

### pyproject.toml
```toml
[tool.poetry]
name = "eyay-hafiza-servisi"
version = "0.1.0"
description = "EyAy.OS Hafıza Yönetimi Mikroservisi"
authors = ["Barut Şeref <barutseref@barutben.com>"]

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.104.0"
uvicorn = {extras = ["standard"], version = "^0.24.0"}
sqlalchemy = "^2.0.0"
alembic = "^1.12.0"
psycopg2-binary = "^2.9.0"
redis = "^5.0.0"
qdrant-client = "^1.6.0"
sentence-transformers = "^2.2.0"
numpy = "^1.24.0"
pandas = "^2.1.0"
python-dateutil = "^2.8.0"
pydantic = "^2.4.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"
pytest-asyncio = "^0.21.0"
httpx = "^0.25.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### app/cekirdek/hafiza_motoru.py
```python
import logging
import asyncio
from datetime import datetime, timedelta
from typing import Dict, Any, List, Optional, Union
from collections import deque
import json

import redis
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct
from sentence_transformers import SentenceTransformer
import numpy as np
from sqlalchemy.orm import Session

from app.modeller.konusma import Konusma
from app.modeller.hafiza_ogesi import HafizaOgesi
from app.veritabani import get_db

logger = logging.getLogger(__name__)

class HafizaMotoru:
    """Ana hafıza yönetim motoru"""
    
    def __init__(self, ayarlar: Dict[str, Any]):
        self.ayarlar = ayarlar
        self.redis_client = None
        self.qdrant_client = None
        self.embedding_model = None
        self.kisa_hafiza = deque(maxlen=ayarlar.get('kisa_hafiza_limit', 100))
        self._baglanti_kur()
    
    def _baglanti_kur(self):
        """Veritabanı bağlantılarını kur"""
        try:
            # Redis bağlantısı (kısa vadeli hafıza için)
            self.redis_client = redis.Redis(
                host=self.ayarlar.get('redis_host', 'localhost'),
                port=self.ayarlar.get('redis_port', 6379),
                db=self.ayarlar.get('redis_db', 0),
                decode_responses=True
            )
            
            # Qdrant bağlantısı (vektör hafıza için)
            self.qdrant_client = QdrantClient(
                host=self.ayarlar.get('qdrant_host', 'localhost'),
                port=self.ayarlar.get('qdrant_port', 6333)
            )
            
            # Embedding modeli
            self.embedding_model = SentenceTransformer(
                'emrecan/bert-base-turkish-cased-mean-nli-stsb-tr'
            )
            
            # Qdrant koleksiyonunu oluştur
            self._qdrant_koleksiyon_olustur()
            
            logger.info("✅ Hafıza motoru bağlantıları kuruldu")
            
        except Exception as e:
            logger.error(f"❌ Hafıza motoru bağlantı hatası: {e}")
            raise
    
    def _qdrant_koleksiyon_olustur(self):
        """Qdrant koleksiyonunu oluştur"""
        try:
            koleksiyon_adi = "eyay_hafiza"
            
            # Koleksiyon var mı kontrol et
            koleksiyonlar = self.qdrant_client.get_collections()
            mevcut_koleksiyonlar = [k.name for k in koleksiyonlar.collections]
            
            if koleksiyon_adi not in mevcut_koleksiyonlar:
                self.qdrant_client.create_collection(
                    collection_name=koleksiyon_adi,
                    vectors_config=VectorParams(
                        size=768,  # Turkish BERT embedding boyutu
                        distance=Distance.COSINE
                    )
                )
                logger.info(f"Qdrant koleksiyonu oluşturuldu: {koleksiyon_adi}")
            
        except Exception as e:
            logger.error(f"Qdrant koleksiyon oluşturma hatası: {e}")
    
    async def kisa_hafiza_ekle(
        self, 
        kullanici_id: int, 
        mesaj: str, 
        kaynak: str = "kullanici",
        metadata: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        """Kısa vadeli hafızaya mesaj ekle"""
        try:
            hafiza_ogesi = {
                "id": f"{kullanici_id}_{datetime.now().timestamp()}",
                "kullanici_id": kullanici_id,
                "mesaj": mesaj,
                "kaynak": kaynak,
                "zaman": datetime.now().isoformat(),
                "metadata": metadata or {}
            }
            
            # Yerel hafızaya ekle
            self.kisa_hafiza.append(hafiza_ogesi)
            
            # Redis'e ekle (TTL ile)
            redis_key = f"kisa_hafiza:{kullanici_id}"
            self.redis_client.lpush(redis_key, json.dumps(hafiza_ogesi))
            self.redis_client.expire(redis_key, 3600)  # 1 saat TTL
            
            # Hafıza limitini kontrol et
            hafiza_uzunlugu = self.redis_client.llen(redis_key)
            if hafiza_uzunlugu > self.ayarlar.get('kisa_hafiza_limit', 100):
                self.redis_client.rpop(redis_key)
            
            logger.debug(f"Kısa hafızaya eklendi: {mesaj[:50]}...")
            
            return {
                "durum": "basarili",
                "hafiza_id": hafiza_ogesi["id"],
                "hafiza_uzunlugu": hafiza_uzunlugu
            }
            
        except Exception as e:
            logger.error(f"Kısa hafıza ekleme hatası: {e}")
            return {"durum": "hata", "mesaj": str(e)}
    
    async def uzun_hafiza_kaydet(
        self,
        kullanici_id: int,
        kullanici_mesaj: str,
        eyay_yanit: str,
        metadata: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        """Uzun vadeli hafızaya konuşma kaydet"""
        try:
            # PostgreSQL'e kaydet
            db = next(get_db())
            
            konusma = Konusma(
                kullanici_id=kullanici_id,
                kullanici_mesaj=kullanici_mesaj,
                eyay_yanit=eyay_yanit,
                metadata=json.dumps(metadata or {}),
                olusturulma_tarihi=datetime.now()
            )
            
            db.add(konusma)
            db.commit()
            db.refresh(konusma)
            
            # Vektör hafızasına da ekle
            await self._vektor_hafiza_ekle(
                konusma.id,
                kullanici_mesaj,
                eyay_yanit,
                kullanici_id
            )
            
            logger.debug(f"Uzun hafızaya kaydedildi: ID {konusma.id}")
            
            return {
                "durum": "basarili",
                "konusma_id": konusma.id
            }
            
        except Exception as e:
            logger.error(f"Uzun hafıza kaydetme hatası: {e}")
            return {"durum": "hata", "mesaj": str(e)}
    
    async def _vektor_hafiza_ekle(
        self,
        konusma_id: int,
        kullanici_mesaj: str,
        eyay_yanit: str,
        kullanici_id: int
    ):
        """Vektör hafızasına konuşma ekle"""
        try:
            # Mesajları birleştir
            tam_metin = f"{kullanici_mesaj} {eyay_yanit}"
            
            # Embedding oluştur
            embedding = self.embedding_model.encode(tam_metin)
            
            # Qdrant'a ekle
            point = PointStruct(
                id=konusma_id,
                vector=embedding.tolist(),
                payload={
                    "kullanici_id": kullanici_id,
                    "kullanici_mesaj": kullanici_mesaj,
                    "eyay_yanit": eyay_yanit,
                    "zaman": datetime.now().isoformat(),
                    "metin_uzunlugu": len(tam_metin)
                }
            )
            
            self.qdrant_client.upsert(
                collection_name="eyay_hafiza",
                points=[point]
            )
            
            logger.debug(f"Vektör hafızasına eklendi: ID {konusma_id}")
            
        except Exception as e:
            logger.error(f"Vektör hafıza ekleme hatası: {e}")
    
    async def kisa_hafiza_getir(
        self, 
        kullanici_id: int, 
        limit: int = 20
    ) -> List[Dict[str, Any]]:
        """Kısa vadeli hafızayı getir"""
        try:
            redis_key = f"kisa_hafiza:{kullanici_id}"
            hafiza_listesi = self.redis_client.lrange(redis_key, 0, limit - 1)
            
            sonuc = []
            for item in hafiza_listesi:
                try:
                    hafiza_ogesi = json.loads(item)
                    sonuc.append(hafiza_ogesi)
                except json.JSONDecodeError:
                    continue
            
            return sonuc
            
        except Exception as e:
            logger.error(f"Kısa hafıza getirme hatası: {e}")
            return []
    
    async def uzun_hafiza_ara(
        self,
        kullanici_id: int,
        arama_terimi: str,
        limit: int = 10
    ) -> List[Dict[str, Any]]:
        """Uzun vadeli hafızada arama yap"""
        try:
            # Arama teriminin embedding'ini oluştur
            arama_embedding = self.embedding_model.encode(arama_terimi)
            
            # Qdrant'ta arama yap
            arama_sonucu = self.qdrant_client.search(
                collection_name="eyay_hafiza",
                query_vector=arama_embedding.tolist(),
                query_filter={
                    "must": [
                        {"key": "kullanici_id", "match": {"value": kullanici_id}}
                    ]
                },
                limit=limit
            )
            
            sonuc = []
            for hit in arama_sonucu:
                sonuc.append({
                    "skor": hit.score,
                    "konusma_id": hit.id,
                    "kullanici_mesaj": hit.payload["kullanici_mesaj"],
                    "eyay_yanit": hit.payload["eyay_yanit"],
                    "zaman": hit.payload["zaman"]
                })
            
            return sonuc
            
        except Exception as e:
            logger.error(f"Uzun hafıza arama hatası: {e}")
            return []
    
    async def benzer_konusmalar_bul(
        self,
        kullanici_id: int,
        referans_mesaj: str,
        limit: int = 5
    ) -> List[Dict[str, Any]]:
        """Benzer konuşmaları bul"""
        try:
            # Referans mesajın embedding'ini oluştur
            referans_embedding = self.embedding_model.encode(referans_mesaj)
            
            # Benzerlik araması yap
            arama_sonucu = self.qdrant_client.search(
                collection_name="eyay_hafiza",
                query_vector=referans_embedding.tolist(),
                query_filter={
                    "must": [
                        {"key": "kullanici_id", "match": {"value": kullanici_id}}
                    ]
                },
                limit=limit,
                score_threshold=0.7  # Minimum benzerlik skoru
            )
            
            sonuc = []
            for hit in arama_sonucu:
                sonuc.append({
                    "benzerlik_skoru": hit.score,
                    "konusma_id": hit.id,
                    "kullanici_mesaj": hit.payload["kullanici_mesaj"],
                    "eyay_yanit": hit.payload["eyay_yanit"],
                    "zaman": hit.payload["zaman"]
                })
            
            return sonuc
            
        except Exception as e:
            logger.error(f"Benzer konuşma bulma hatası: {e}")
            return []
    
    async def baglamsal_hafiza_olustur(
        self,
        kullanici_id: int,
        mevcut_mesaj: str,
        gecmis_limit: int = 5
    ) -> Dict[str, Any]:
        """Mevcut mesaj için bağlamsal hafıza oluştur"""
        try:
            # Son konuşmaları getir
            kisa_hafiza = await self.kisa_hafiza_getir(kullanici_id, gecmis_limit)
            
            # Benzer konuşmaları bul
            benzer_konusmalar = await self.benzer_konusmalar_bul(
                kullanici_id, mevcut_mesaj, 3
            )
            
            # Bağlamsal özet oluştur
            baglam = {
                "mevcut_mesaj": mevcut_mesaj,
                "son_konusmalar": kisa_hafiza,
                "benzer_konusmalar": benzer_konusmalar,
                "kullanici_id": kullanici_id,
                "olusturulma_zamani": datetime.now().isoformat()
            }
            
            return baglam
            
        except Exception as e:
            logger.error(f"Bağlamsal hafıza oluşturma hatası: {e}")
            return {}
    
    async def hafiza_temizle(
        self,
        kullanici_id: int,
        hafiza_tipi: str = "kisa"
    ) -> Dict[str, Any]:
        """Hafızayı temizle"""
        try:
            if hafiza_tipi == "kisa":
                # Redis'ten temizle
                redis_key = f"kisa_hafiza:{kullanici_id}"
                silinen_sayisi = self.redis_client.delete(redis_key)
                
                return {
                    "durum": "basarili",
                    "silinen_mesaj_sayisi": silinen_sayisi,
                    "hafiza_tipi": "kisa"
                }
            
            elif hafiza_tipi == "uzun":
                # PostgreSQL'den temizle (dikkatli!)
                db = next(get_db())
                silinen_sayisi = db.query(Konusma).filter(
                    Konusma.kullanici_id == kullanici_id
                ).delete()
                db.commit()
                
                # Qdrant'tan da temizle
                self.qdrant_client.delete(
                    collection_name="eyay_hafiza",
                    points_selector={
                        "filter": {
                            "must": [
                                {"key": "kullanici_id", "match": {"value": kullanici_id}}
                            ]
                        }
                    }
                )
                
                return {
                    "durum": "basarili",
                    "silinen_konusma_sayisi": silinen_sayisi,
                    "hafiza_tipi": "uzun"
                }
            
            else:
                return {
                    "durum": "hata",
                    "mesaj": "Geçersiz hafıza tipi"
                }
                
        except Exception as e:
            logger.error(f"Hafıza temizleme hatası: {e}")
            return {"durum": "hata", "mesaj": str(e)}
    
    async def hafiza_istatistikleri(
        self,
        kullanici_id: int
    ) -> Dict[str, Any]:
        """Hafıza istatistiklerini getir"""
        try:
            # Kısa hafıza istatistikleri
            redis_key = f"kisa_hafiza:{kullanici_id}"
            kisa_hafiza_sayisi = self.redis_client.llen(redis_key)
            
            # Uzun hafıza istatistikleri
            db = next(get_db())
            uzun_hafiza_sayisi = db.query(Konusma).filter(
                Konusma.kullanici_id == kullanici_id
            ).count()
            
            # Vektör hafıza istatistikleri
            vektor_sayisi = self.qdrant_client.count(
                collection_name="eyay_hafiza",
                count_filter={
                    "must": [
                        {"key": "kullanici_id", "match": {"value": kullanici_id}}
                    ]
                }
            ).count
            
            return {
                "kullanici_id": kullanici_id,
                "kisa_hafiza_mesaj_sayisi": kisa_hafiza_sayisi,
                "uzun_hafiza_konusma_sayisi": uzun_hafiza_sayisi,
                "vektor_hafiza_nokta_sayisi": vektor_sayisi,
                "toplam_hafiza_boyutu": kisa_hafiza_sayisi + uzun_hafiza_sayisi,
                "son_guncelleme": datetime.now().isoformat()
            }
            
        except Exception as e:
            logger.error(f"Hafıza istatistikleri hatası: {e}")
            return {}
    
    def durum_bilgisi(self) -> Dict[str, Any]:
        """Hafıza motoru durum bilgisi"""
        try:
            redis_durum = self.redis_client.ping() if self.redis_client else False
            qdrant_durum = bool(self.qdrant_client)
            
            return {
                "redis_baglantisi": redis_durum,
                "qdrant_baglantisi": qdrant_durum,
                "embedding_model": bool(self.embedding_model),
                "kisa_hafiza_limit": self.ayarlar.get('kisa_hafiza_limit', 100),
                "aktif": redis_durum and qdrant_durum
            }
            
        except Exception as e:
            logger.error(f"Durum bilgisi hatası: {e}")
            return {"aktif": False, "hata": str(e)}
```

### app/semalar/hafiza.py
```python
from typing import List, Optional, Dict, Any
from datetime import datetime
from pydantic import BaseModel, Field

class HafizaOgesi(BaseModel):
    id: str
    kullanici_id: int
    mesaj: str
    kaynak: str
    zaman: datetime
    metadata: Optional[Dict[str, Any]] = None

class KonusmaKaydi(BaseModel):
    kullanici_mesaj: str = Field(..., min_length=1, max_length=1000)
    eyay_yanit: str = Field(..., min_length=1, max_length=2000)
    metadata: Optional[Dict[str, Any]] = None

class HafizaArama(BaseModel):
    arama_terimi: str = Field(..., min_length=1, max_length=200)
    limit: Optional[int] = Field(10, ge=1, le=50)

class BenzerlikArama(BaseModel):
    referans_mesaj: str = Field(..., min_length=1, max_length=500)
    limit: Optional[int] = Field(5, ge=1, le=20)
    minimum_skor: Optional[float] = Field(0.7, ge=0.0, le=1.0)

class HafizaTemizleme(BaseModel):
    hafiza_tipi: str = Field(..., regex="^(kisa|uzun|tumu)$")
    onay: bool = Field(..., description="Temizleme işlemini onaylıyorum")

class BaglamselHafiza(BaseModel):
    mevcut_mesaj: str
    son_konusmalar: List[HafizaOgesi]
    benzer_konusmalar: List[Dict[str, Any]]
    kullanici_id: int
    olusturulma_zamani: datetime

class HafizaIstatistikleri(BaseModel):
    kullanici_id: int
    kisa_hafiza_mesaj_sayisi: int
    uzun_hafiza_konusma_sayisi: int
    vektor_hafiza_nokta_sayisi: int
    toplam_hafiza_boyutu: int
    son_guncelleme: datetime
```

### app/api/v1/hafiza.py
```python
from typing import Any, List
from fastapi import APIRouter, HTTPException, Depends, Query
from sqlalchemy.orm import Session

from app.cekirdek.hafiza_motoru import HafizaMotoru
from app.semalar.hafiza import (
    KonusmaKaydi,
    HafizaArama,
    BenzerlikArama,
    HafizaTemizleme,
    HafizaIstatistikleri
)
from app.ayar import ayarlar
from app.api.deps import get_current_user

router = APIRouter()

# Global hafıza motoru instance
hafiza_motoru = HafizaMotoru(ayarlar.HAFIZA_AYARLARI)

@router.post("/kisa-hafiza/ekle")
async def kisa_hafiza_ekle(
    mesaj: str,
    kaynak: str = "kullanici",
    mevcut_kullanici = Depends(get_current_user)
) -> Any:
    """Kısa vadeli hafızaya mesaj ekle"""
    try:
        if not mesaj.strip():
            raise HTTPException(status_code=400, detail="Mesaj boş olamaz")
        
        sonuc = await hafiza_motoru.kisa_hafiza_ekle(
            kullanici_id=mevcut_kullanici.id,
            mesaj=mesaj,
            kaynak=kaynak
        )
        
        if sonuc.get("durum") == "hata":
            raise HTTPException(status_code=500, detail=sonuc.get("mesaj"))
        
        return sonuc
        
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/kisa-hafiza")
async def kisa_hafiza_getir(
    limit: int = Query(20, ge=1, le=100),
    mevcut_kullanici = Depends(get_current_user)
) -> Any:
    """Kısa vadeli hafızayı getir"""
    try:
        hafiza = await hafiza_motoru.kisa_hafiza_getir(
            kullanici_id=mevcut_kullanici.id,
            limit=limit
        )
        
        return {
            "kullanici_id": mevcut_kullanici.id,
            "hafiza": hafiza,
            "toplam": len(hafiza)
        }
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/uzun-hafiza/kaydet")
async def uzun_hafiza_kaydet(
    konusma: KonusmaKaydi,
    mevcut_kullanici = Depends(get_current_user)
) -> Any:
    """Uzun vadeli hafızaya konuşma kaydet"""
    try:
        sonuc = await hafiza_motoru.uzun_hafiza_kaydet(
            kullanici_id=mevcut_kullanici.id,
            kullanici_mesaj=konusma.kullanici_mesaj,
            eyay_yanit=konusma.eyay_yanit,
            metadata=konusma.metadata
        )
        
        if sonuc.get("durum") == "hata":
            raise HTTPException(status_code=500, detail=sonuc.get("mesaj"))
        
        return sonuc
        
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/ara")
async def hafiza_ara(
    arama: HafizaArama,
    mevcut_kullanici = Depends(get_current_user)
) -> Any:
    """Hafızada arama yap"""
    try:
        sonuclar = await hafiza_motoru.uzun_hafiza_ara(
            kullanici_id=mevcut_kullanici.id,
            arama_terimi=arama.arama_terimi,
            limit=arama.limit
        )
        
        return {
            "arama_terimi": arama.arama_terimi,
            "sonuclar": sonuclar,
            "toplam": len(sonuclar)
        }
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/benzer-konusmalar")
async def benzer_konusmalar_bul(
    benzerlik: BenzerlikArama,
    mevcut_kullanici = Depends(get_current_user)
) -> Any:
    """Benzer konuşmaları bul"""
    try:
        sonuclar = await hafiza_motoru.benzer_konusmalar_bul(
            kullanici_id=mevcut_kullanici.id,
            referans_mesaj=benzerlik.referans_mesaj,
            limit=benzerlik.limit
        )
        
        # Minimum skora göre filtrele
        filtrelenmis_sonuclar = [
            s for s in sonuclar 
            if s.get("benzerlik_skoru", 0) >= benzerlik.minimum_skor
        ]
        
        return {
            "referans_mesaj": benzerlik.referans_mesaj,
            "benzer_konusmalar": filtrelenmis_sonuclar,
            "toplam": len(filtrelenmis_sonuclar)
        }
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/baglamsal-hafiza")
async def baglamsal_hafiza_olustur(
    mevcut_mesaj: str,
    gecmis_limit: int = Query(5, ge=1, le=20),
    mevcut_kullanici = Depends(get_current_user)
) -> Any:
    """Bağlamsal hafıza oluştur"""
    try:
        if not mevcut_mesaj.strip():
            raise HTTPException(status_code=400, detail="Mesaj boş olamaz")
        
        baglam = await hafiza_motoru.baglamsal_hafiza_olustur(
            kullanici_id=mevcut_kullanici.id,
            mevcut_mesaj=mevcut_mesaj,
            gecmis_limit=gecmis_limit
        )
        
        return baglam
        
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.delete("/temizle")
async def hafiza_temizle(
    temizleme: Hafiz
