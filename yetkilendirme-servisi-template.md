# Yetkilendirme Servisi - Modern Authentication Microservice

## 📁 Dizin Yapısı

```
hizmetler/yetkilendirme/
├── app/
│   ├── __init__.py
│   ├── eyay.py              # FastAPI app
│   ├── ayar.py            # Configuration
│   ├── veritabani.py          # Database connection
│   ├── modeller/
│   │   ├── __init__.py
│   │   ├── kullanici.py     # User model
│   │   └── token.py         # Token model
│   ├── semalar/
│   │   ├── __init__.py
│   │   ├── kullanici.py     # Pydantic schemas
│   │   └── yetkilendirme.py # Auth schemas
│   ├── api/
│   │   ├── __init__.py
│   │   ├── deps.py          # Dependencies
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── yetkilendirme.py  # Auth endpoints
│   │       └── kullanicilar.py   # User endpoints
│   ├── cekirdek/
│   │   ├── __init__.py
│   │   ├── guvenlik.py      # JWT, password hashing
│   │   └── ayar.py        # Core configuration
│   └── hizmetler/
│       ├── __init__.py
│       ├── yetkilendirme.py # Auth business logic
│       └── kullanici.py     # User business logic
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_yetkilendirme.py
│   └── test_kullanicilar.py
├── aktarim/
├── Dockerfile
├── pyproject.toml
└── README.md
```

## 🔧 Temel Dosyalar

### pyproject.toml
```toml
[tool.poetry]
name = "eyay-yetkilendirme-servisi"
version = "0.1.0"
description = "EyAy.OS Yetkilendirme Mikroservisi"
authors = ["Barut Şeref <barutseref@barutben.com>"]

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.104.0"
uvicorn = {extras = ["standard"], version = "^0.24.0"}
sqlalchemy = "^2.0.0"
alembic = "^1.12.0"
psycopg2-binary = "^2.9.0"
python-jose = {extras = ["cryptography"], version = "^3.3.0"}
passlib = {extras = ["bcrypt"], version = "^1.7.0"}
python-multipart = "^0.0.6"
pydantic = {extras = ["email"], version = "^2.4.0"}
pydantic-settings = "^2.0.0"
redis = "^5.0.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"
pytest-asyncio = "^0.21.0"
httpx = "^0.25.0"
black = "^23.7.0"
ruff = "^0.0.287"
mypy = "^1.5.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"

[tool.black]
line-length = 88
target-version = ['py311']

[tool.ruff]
target-version = "py311"
line-length = 88
select = ["E", "F", "I", "N", "W", "UP"]

[tool.mypy]
python_version = "3.11"
strict = true
```

### app/main.py
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import HTTPBearer

from app.api.v1 import yetkilendirme, kullanicilar
from app.core.config import settings
from app.database import engine
from app.models import Base

# Tabloları oluştur
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="EyAy.OS Yetkilendirme Servisi",
    description="EyAy.OS için kimlik doğrulama ve yetkilendirme mikroservisi",
    version="0.1.0",
    docs_url="/docs" if settings.DEBUG else None,
    redoc_url="/redoc" if settings.DEBUG else None,
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_HOSTS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Güvenlik
security = HTTPBearer()

# Router'ları dahil et
app.include_router(
    yetkilendirme.router, 
    prefix="/api/v1/yetkilendirme", 
    tags=["yetkilendirme"]
)
app.include_router(
    kullanicilar.router, 
    prefix="/api/v1/kullanicilar", 
    tags=["kullanicilar"]
)

@app.get("/saglik")
async def saglik_kontrolu():
    return {"durum": "saglikli", "servis": "yetkilendirme-servisi"}

@app.get("/")
async def ana_sayfa():
    return {"mesaj": "EyAy.OS Yetkilendirme Servisi", "surum": "0.1.0"}
```

### app/core/guvenlik.py
```python
from datetime import datetime, timedelta
from typing import Any, Union

from jose import jwt
from passlib.context import CryptContext

from app.core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

ALGORITHM = "HS256"

def erisim_token_olustur(
    konu: Union[str, Any], gecerlilik_suresi: timedelta = None
) -> str:
    """Erişim token'ı oluştur"""
    if gecerlilik_suresi:
        son_gecerlilik = datetime.utcnow() + gecerlilik_suresi
    else:
        son_gecerlilik = datetime.utcnow() + timedelta(
            minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
        )
    
    kodlanacak_veri = {"exp": son_gecerlilik, "sub": str(konu)}
    kodlanmis_jwt = jwt.encode(kodlanacak_veri, settings.SECRET_KEY, algorithm=ALGORITHM)
    return kodlanmis_jwt

def sifre_dogrula(duz_sifre: str, hash_sifre: str) -> bool:
    """Şifre doğrulaması yap"""
    return pwd_context.verify(duz_sifre, hash_sifre)

def sifre_hash_olustur(sifre: str) -> str:
    """Şifre hash'i oluştur"""
    return pwd_context.hash(sifre)

def token_dogrula(token: str) -> Union[str, None]:
    """Token doğrulaması yap"""
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[ALGORITHM])
        kullanici_id: str = payload.get("sub")
        if kullanici_id is None:
            return None
        return kullanici_id
    except jwt.JWTError:
        return None
```

### app/models/kullanici.py
```python
from sqlalchemy import Boolean, Column, Integer, String, DateTime
from sqlalchemy.sql import func

from app.database import Base

class Kullanici(Base):
    __tablename__ = "kullanicilar"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    kullanici_adi = Column(String, unique=True, index=True, nullable=False)
    tam_ad = Column(String, nullable=True)
    hash_sifre = Column(String, nullable=False)
    aktif_mi = Column(Boolean, default=True)
    super_kullanici_mi = Column(Boolean, default=False)
    olusturulma_tarihi = Column(DateTime(timezone=True), server_default=func.now())
    guncelleme_tarihi = Column(DateTime(timezone=True), onupdate=func.now())
    son_giris = Column(DateTime(timezone=True), nullable=True)
    dil_tercihi = Column(String, default="tr")
    tema_tercihi = Column(String, default="acik")
```

### app/schemas/kullanici.py
```python
from typing import Optional
from datetime import datetime
from pydantic import BaseModel, EmailStr

class KullaniciBase(BaseModel):
    email: EmailStr
    kullanici_adi: str
    tam_ad: Optional[str] = None
    aktif_mi: bool = True
    dil_tercihi: str = "tr"
    tema_tercihi: str = "acik"

class KullaniciOlustur(KullaniciBase):
    sifre: str

class KullaniciGuncelle(BaseModel):
    email: Optional[EmailStr] = None
    tam_ad: Optional[str] = None
    sifre: Optional[str] = None
    dil_tercihi: Optional[str] = None
    tema_tercihi: Optional[str] = None

class KullaniciVeritabaniBase(KullaniciBase):
    id: int
    olusturulma_tarihi: datetime
    son_giris: Optional[datetime] = None
    
    class Config:
        from_attributes = True

class Kullanici(KullaniciVeritabaniBase):
    pass

class KullaniciVeritabaninda(KullaniciVeritabaniBase):
    hash_sifre: str
```

### app/schemas/yetkilendirme.py
```python
from pydantic import BaseModel

class GirisYap(BaseModel):
    kullanici_adi: str
    sifre: str

class Token(BaseModel):
    erisim_token: str
    token_tipi: str = "bearer"
    gecerlilik_suresi: int  # dakika cinsinden

class TokenVerisi(BaseModel):
    kullanici_id: Optional[int] = None
```

### app/api/v1/yetkilendirme.py
```python
from datetime import timedelta
from typing import Any

from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session

from app import schemas
from app.api import deps
from app.core import guvenlik
from app.core.config import settings
from app.services import kullanici as kullanici_servisi

router = APIRouter()

@router.post("/kayit-ol", response_model=schemas.Kullanici, status_code=201)
def kayit_ol(
    *,
    db: Session = Depends(deps.get_db),
    kullanici_verisi: schemas.KullaniciOlustur,
) -> Any:
    """Yeni kullanıcı kaydı"""
    # Email kontrolü
    kullanici = kullanici_servisi.email_ile_kullanici_getir(
        db, email=kullanici_verisi.email
    )
    if kullanici:
        raise HTTPException(
            status_code=400,
            detail="Bu email adresi zaten kayıtlı"
        )
    
    # Kullanıcı adı kontrolü
    kullanici = kullanici_servisi.kullanici_adi_ile_kullanici_getir(
        db, kullanici_adi=kullanici_verisi.kullanici_adi
    )
    if kullanici:
        raise HTTPException(
            status_code=400,
            detail="Bu kullanıcı adı zaten alınmış"
        )
    
    kullanici = kullanici_servisi.kullanici_olustur(db, obj_in=kullanici_verisi)
    return kullanici

@router.post("/giris-yap", response_model=schemas.Token)
def giris_yap(
    db: Session = Depends(deps.get_db),
    form_data: OAuth2PasswordRequestForm = Depends()
) -> Any:
    """Kullanıcı girişi"""
    kullanici = kullanici_servisi.kimlik_dogrula(
        db, kullanici_adi=form_data.username, sifre=form_data.password
    )
    if not kullanici:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Kullanıcı adı veya şifre hatalı"
        )
    elif not kullanici_servisi.aktif_mi(kullanici):
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Hesap aktif değil"
        )
    
    erisim_token_gecerlilik = timedelta(
        minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
    )
    
    return {
        "erisim_token": guvenlik.erisim_token_olustur(
            kullanici.id, gecerlilik_suresi=erisim_token_gecerlilik
        ),
        "token_tipi": "bearer",
        "gecerlilik_suresi": settings.ACCESS_TOKEN_EXPIRE_MINUTES
    }

@router.post("/token-dogrula")
def token_dogrula(
    mevcut_kullanici: schemas.Kullanici = Depends(deps.get_current_user)
) -> Any:
    """Token doğrulaması"""
    return {
        "gecerli": True,
        "kullanici_id": mevcut_kullanici.id,
        "kullanici_adi": mevcut_kullanici.kullanici_adi
    }
```

### Dockerfile
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Sistem bağımlılıklarını yükle
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Poetry yükle
RUN pip install poetry

# Poetry dosyalarını kopyala
COPY pyproject.toml poetry.lock* ./

# Poetry yapılandır
RUN poetry config virtualenvs.create false

# Bağımlılıkları yükle
RUN poetry install --no-dev

# Uygulamayı kopyala
COPY ./app /app/app

# Port aç
EXPOSE 8001

# Uygulamayı çalıştır
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001"]
```

## 🧪 Test Örnekleri

### tests/test_yetkilendirme.py
```python
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_kullanici_kayit():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.post(
            "/api/v1/yetkilendirme/kayit-ol",
            json={
                "email": "test@example.com",
                "kullanici_adi": "testkullanici",
                "sifre": "testsifre123",
                "tam_ad": "Test Kullanıcı"
            }
        )
    assert response.status_code == 201
    assert response.json()["email"] == "test@example.com"

@pytest.mark.asyncio
async def test_kullanici_giris():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        # Önce kayıt ol
        await ac.post(
            "/api/v1/yetkilendirme/kayit-ol",
            json={
                "email": "giris@example.com",
                "kullanici_adi": "giriskullanici",
                "sifre": "girissifre123"
            }
        )
        
        # Sonra giriş yap
        response = await ac.post(
            "/api/v1/yetkilendirme/giris-yap",
            data={
                "username": "giriskullanici",
                "password": "girissifre123"
            }
        )
    assert response.status_code == 200
    assert "erisim_token" in response.json()
```

## 🚀 Çalıştırma

```bash
# Bağımlılıkları yükle
cd hizmetler/yetkilendirme
poetry install

# Veritabanı migration
poetry run alembic upgrade head

# Development server başlat
poetry run uvicorn app.main:app --reload --port 8001

# Testleri çalıştır
poetry run pytest
```

## 📋 API Endpoints

- `POST /api/v1/yetkilendirme/kayit-ol` - Kullanıcı kaydı
- `POST /api/v1/yetkilendirme/giris-yap` - Giriş yapma
- `POST /api/v1/yetkilendirme/token-dogrula` - Token doğrulama
- `GET /api/v1/kullanicilar/ben` - Mevcut kullanıcı bilgisi
- `PUT /api/v1/kullanicilar/ben` - Kullanıcı bilgisi güncelleme
- `GET /saglik` - Sağlık kontrolü

## 🔐 Güvenlik Özellikleri

- JWT tabanlı authentication
- Bcrypt ile şifre hashleme
- Rate limiting
- CORS koruması
- SQL injection koruması
- XSS koruması
