# FreeSWITCH Sesli Asistan Projesi

Bu proje, FreeSWITCH telefon sistemi ile AI destekli sesli asistanı entegre eden gerçek zamanlı bir çözümdür. Sistem, Whisper ASR (GPU sunucusu), harici chatbot servisi (canlı URL üzerinden) ve ElevenLabs TTS servislerini birleştirerek tam otomatik bir sesli asistan deneyimi sunar.

## 🏗️ Sistem Mimarisi

```
┌──────────────┐
│ FreeSWITCH   │ (Telefon sistemi)
│              │
│ audio_fork → │ (WebSocket üzerinden ses akışı)
└──────┬───────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ main.py (WebSocket Server - Port 8765)                      │
│ - UUID yönetimi                                             │
│ - Ses verisi koordinasyonu                                  │
│ - STT → Chatbot → TTS akışı yönetimi                        │
└──────┬──────────────────────────────────────────────────────┘
       │
       ├─► audio_utils.py (VAD + Ses İşleme)
       │   └─► Konuşma segmentleri → WAV dosyaları
       │
       ├─► stt.py (Speech-to-Text)
       │   └─► WAV → GPU Sunucusu (Whisper via ngrok)
       │       └─► Transkript
       │
       ├─► chatbot.py (AI Chatbot)
       │   └─► Transkript → Chatbot API/WebSocket (canlı URL)
       │       └─► Yanıt metni
       │
       ├─► tts.py (Text-to-Speech)
       │   └─► Metin → ElevenLabs API
       │       └─► MP3 → FFmpeg → PCM16LE WAV
       │
       └─► freeswitch.py
           └─► WAV → FreeSWITCH (uuid_broadcast)
               └─► Telefon hattında çalma
```

## 📁 Proje Dosyaları ve İşlevleri

### 1. **main.py** - Ana WebSocket Sunucusu ve Orchestrator

Bu dosya **ana koordinatör** görevi görür ve tüm sistemi yönetir:

- **WebSocket Sunucusu**: Port 8765 üzerinde FreeSWITCH'ten gelen ses verilerini dinler
- **UUID Yönetimi**: Her çağrı için benzersiz UUID alır ve saklar
- **Ses Verisi Koordinasyonu**: Gelen ham ses verilerini `audio_utils.py`'ye yönlendirir
- **Akış Yönetimi**: STT → Chatbot → TTS → FreeSWITCH döngüsünü yönetir
- **Barge-in Desteği**: Kullanıcı konuştuğunda TTS'i keser
- **Session Yönetimi**: Her çağrı için ayrı session takibi

**Önemli Özellikler:**
- Async/await tabanlı yapı (gerçek zamanlı işleme)
- Çoklu bağlantı desteği
- Hata yönetimi ve retry mekanizmaları

### 2. **audio_utils.py** - Ses İşleme ve VAD (Voice Activity Detection)

Bu dosya **ses analizi ve işleme** merkezidir:

- **VAD (Voice Activity Detection)**: WebRTC VAD kütüphanesi ile konuşma tespiti
- **RMS (Root Mean Square) Analizi**: Ses seviyesi analizi için
- **Dinamik Eşik Değeri**: Ortam gürültüsüne göre otomatik ayarlama
- **Peak Detection**: Ses piklerini tespit ederek konuşma başlangıcını yakalar
- **Noise Detection**: Gürültü filtreleme algoritmaları
- **Otomatik Kayıt**: Konuşma segmentlerini otomatik olarak WAV dosyasına kaydeder
- **Sessizlik Tespiti**: Belirlenen süre sessizlik sonrası konuşma bitişini tespit eder

**Parametreler:**
- `SILENCE_DURATION`: 1.5 saniye (konuşma bitişi için)
- `MIN_VOLUME_THRESHOLD`: 500 (ses tespiti için minimum eşik)
- `VAD_AGGRESSIVENESS`: 3 (0-3 arası, en agresif)

### 3. **stt.py** - Speech-to-Text (Konuşmayı Metne Çevirme)

Bu dosya **konuşma tanıma** işlemini yapar:

- **GPU Sunucusu Entegrasyonu**: Harici bir GPU sunucusunda çalışan Whisper modeline bağlanır
- **Ngrok Tünelleme**: GPU sunucusu ngrok ile canlıya alınmıştır (`GPU_SERVER_URL`)
- **WAV Upload**: Kaydedilen WAV dosyalarını HTTP POST ile GPU sunucusuna gönderir
- **Retry Mekanizması**: Bağlantı hatalarında otomatik yeniden deneme (3 kez)
- **Süre Hesaplama**: Ses dosyası süresini hesaplar ve loglar
- **Streaming ASR Desteği**: `StreamingWhisperASR` sınıfı ile gerçek zamanlı transkripsiyon

**GPU Sunucusu Formatı:**
- Endpoint: `POST /transcribe`
- Input: WAV dosyası (multipart/form-data)
- Output: `{"transcript": "..."}`

**Ngrok Yapılandırması:**
```python
GPU_SERVER_URL = "https://[ngrok-url].ngrok-free.app/transcribe"
```

### 4. **tts.py** - Text-to-Speech (Metni Sese Çevirme)

Bu dosya **metin okuma** işlemini yapar:

- **ElevenLabs API**: Yüksek kaliteli TTS için ElevenLabs API kullanır
- **Model**: `eleven_multilingual_v2` (çok dilli destek)
- **Format Dönüşümü**: 
  - ElevenLabs → MP3
  - FFmpeg → PCM16LE 16kHz mono WAV (FreeSWITCH uyumlu)
- **Metin Bölme**: Uzun metinleri 450 karakterlik parçalara böler
- **Parça Birleştirme**: Birden fazla parça varsa sessizlik ekleyerek birleştirir
- **Rate Limit Handling**: 429 hatası durumunda otomatik bekleme
- **Temizlik**: Geçici MP3 ve WAV dosyalarını otomatik siler

**Ses Parametreleri:**
- Format: PCM16LE
- Sample Rate: 16kHz
- Channels: Mono (1)
- Çıktı: `/var/lib/freeswitch/recordings/output.wav`

### 5. **chatbot.py** - AI Chatbot Entegrasyonu

Bu dosya **harici chatbot servisi** ile iletişim kurar:

- **Canlı URL Bağlantısı**: Başka bir projeden canlıya alınmış chatbot servisine bağlanır
- **Dual Protokol**: 
  - WebSocket (öncelikli, gerçek zamanlı)
  - HTTP POST (fallback)
- **Ngrok Tünelleme**: Chatbot servisi ngrok ile canlıya alınmıştır
- **Session Yönetimi**: Her çağrı için oturum kimliği gönderir
- **Reconnect Mekanizması**: Bağlantı koparsa otomatik yeniden bağlanır
- **Typing Indicator Filtreleme**: WebSocket'ten gelen typing mesajlarını filtreler
- **Timeout Yönetimi**: 30 saniyelik maksimum yanıt bekleme süresi

**Chatbot API Formatı:**

**WebSocket:**
```json
// Gönderilen:
{
  "type": "user_message",
  "text": "kullanıcı mesajı",
  "session_id": "freeswitch_session"
}

// Alınan:
{
  "messages": ["chatbot yanıtı"],
  // veya
  "message": "chatbot yanıtı",
  // veya
  "text": "chatbot yanıtı"
}
```

**HTTP POST:**
- Endpoint: `/api/chat`
- Method: POST
- Body: `{"text": "...", "session_id": "..."}`

**Ngrok Yapılandırması:**
```python
CHATBOT_API_URL = "https://[ngrok-url].ngrok-free.dev/api/chat"
CHATBOT_WS_URL = "wss://[ngrok-url].ngrok-free.dev/ws/chat"
```

### 6. **freeswitch.py** - FreeSWITCH Entegrasyonu

Bu dosya **FreeSWITCH iletişimi** sağlar:

- **fs_cli Komutları**: FreeSWITCH komut satırı arayüzünü kullanır
- **uuid_broadcast**: TTS çıktısını çağrı üzerinde çalar
- **uuid_break**: Barge-in için çalan sesi keser (media/all seçenekleri)
- **Hata Yönetimi**: Dosya varlığı ve boyut kontrolü
- **Loglama**: Tüm FreeSWITCH komutlarını loglar

**Önemli Fonksiyonlar:**
- `wav2call(uuid, wav_path)`: WAV dosyasını çağrıya çalar
- `break_tts(uuid)`: Çalan TTS'i durdurur (barge-in)
- `fs_uuid_broadcast()`: Doğrudan broadcast komutu
- `fs_uuid_break()`: Doğrudan break komutu

### 7. **config.py** - Yapılandırma ve Yardımcı Fonksiyonlar

Bu dosya **merkezi konfigürasyon** dosyasıdır:

- **API Anahtarları**: ElevenLabs API key ve Voice ID
- **Ses Parametreleri**: 
  - Sample rate: 16000 Hz
  - Channels: 1 (mono)
  - Sample width: 2 bytes (16-bit)
  - Chunk duration: 20ms
- **URL Yapılandırması**:
  - GPU sunucusu URL'i (ngrok)
  - Chatbot API/WebSocket URL'leri (ngrok)
- **IPv4 Zorlaması**: Bağlantı sorunları için socket ayarları
- **Loglama**: Merkezi log fonksiyonu (timestamp + dosyaya yazma)

**Yapılandırma Değişkenleri:**
```python
# ElevenLabs
API_KEY = "..."
VOICE_ID = "..."

# GPU Sunucusu (ngrok)
GPU_SERVER_URL = "https://[ngrok-url].ngrok-free.app/transcribe"

# Chatbot (ngrok)
CHATBOT_API_URL = "https://[ngrok-url].ngrok-free.dev/api/chat"
CHATBOT_WS_URL = "wss://[ngrok-url].ngrok-free.dev/ws/chat"

# WebSocket Server
WS_HOST = "0.0.0.0"
WS_PORT = 8765
```

### 8. **send_uuid_ws.py** - UUID Gönderme Scripti

Bu dosya **yardımcı script**tir:

- Komut satırından UUID alır
- WebSocket sunucusuna (port 8765) UUID gönderir
- FreeSWITCH'ten gelen çağrı UUID'sini sisteme bildirir

## 🔄 Detaylı Sistem Akışı

### Adım 1: FreeSWITCH → WebSocket
```
FreeSWITCH (audio_fork) 
  → WebSocket (ws://localhost:8765)
  → main.py (handle_connection)
```

### Adım 2: Ses İşleme ve VAD
```
main.py 
  → audio_utils.py (process_chunk)
  → VAD analizi (konuşma tespiti)
  → Konuşma segmentleri → WAV dosyası
```

### Adım 3: Speech-to-Text (Whisper)
```
audio_utils.py → WAV dosyası
  → stt.py (speech2text)
  → GPU Sunucusu (ngrok URL üzerinden)
  → Whisper ASR modeli
  → Transkript (metin)
```

### Adım 4: Chatbot İşleme
```
stt.py → Transkript
  → chatbot.py (send_to_chatbot)
  → Chatbot WebSocket/API (canlı URL üzerinden)
  → AI yanıtı (metin)
```

### Adım 5: Text-to-Speech
```
chatbot.py → Yanıt metni
  → tts.py (text2speech)
  → ElevenLabs API
  → MP3 → FFmpeg → PCM16LE WAV
```

### Adım 6: FreeSWITCH'e Geri Gönderme
```
tts.py → WAV dosyası
  → freeswitch.py (wav2call)
  → FreeSWITCH (uuid_broadcast)
  → Telefon hattında çalma
```

### Barge-in (Kesinti) Mekanizması

Kullanıcı TTS çalarken konuşmaya başlarsa:
1. `audio_utils.py` konuşmayı tespit eder
2. `main.py` durumu algılar
3. `freeswitch.py` → `break_tts()` çağrılır
4. TTS durdurulur, yeni konuşma işlenir

## 🚀 Kurulum ve Kullanım

### Gereksinimler

**Sistem Gereksinimleri:**
- Python 3.7+
- FreeSWITCH (kurulu ve çalışır durumda)
- FFmpeg (ses format dönüşümü için)
- Internet bağlantısı (API'lere erişim için)

**Harici Servisler:**
- **GPU Sunucusu**: Whisper ASR modeli çalışan sunucu (ngrok ile tünellenmiş)
- **Chatbot Servisi**: Canlı URL üzerinden erişilebilir chatbot (ngrok ile tünellenmiş)
- **ElevenLabs API**: TTS servisi için API anahtarı

### Bağımlılıklar

```bash
pip install websockets numpy webrtcvad pydub requests asyncio
```

**Bağımlılık Açıklamaları:**
- `websockets`: WebSocket sunucu ve istemci desteği
- `numpy`: Ses verisi işleme (RMS, peak detection)
- `webrtcvad`: Voice Activity Detection (konuşma tespiti)
- `pydub`: Ses dosyası format dönüşümleri
- `requests`: HTTP istekleri (STT ve TTS API'leri)
- `asyncio`: Asenkron işlemler (Python standart kütüphanesi)

### Yapılandırma

1. **config.py dosyasını düzenle:**
   ```python
   # ElevenLabs API anahtarı
   API_KEY = "your_elevenlabs_api_key"
   VOICE_ID = "your_voice_id"
   
   # GPU Sunucusu URL'i (ngrok)
   GPU_SERVER_URL = "https://your-ngrok-url.ngrok-free.app/transcribe"
   
   # Chatbot URL'leri (ngrok)
   CHATBOT_API_URL = "https://your-chatbot-url.ngrok-free.dev/api/chat"
   CHATBOT_WS_URL = "wss://your-chatbot-url.ngrok-free.dev/ws/chat"
   ```

2. **Ngrok Tünelleme:**

   **GPU Sunucusu için:**
   ```bash
   # GPU sunucusunda
   ngrok http 8000  # veya GPU sunucunun portu
   # Alınan URL'i config.py'de GPU_SERVER_URL'e yaz
   ```


   **Chatbot Servisi için:**
   ```bash
   # Chatbot sunucusunda
   ngrok http 5000  # veya chatbot'un portu
   # Alınan URL'leri config.py'de CHATBOT_API_URL ve CHATBOT_WS_URL'e yaz
   ```

### Çalıştırma

```bash
cd project_modules
python main.py
```

**Beklenen Çıktı:**
```
[WS] Audio handler listening on ws://0.0.0.0:8765
[WS] Ready for FreeSWITCH connections
```

### FreeSWITCH Konfigürasyonu

FreeSWITCH'te WebSocket'e ses akışı göndermek için `audio_fork` kullanın:

```xml
<!-- Dialplan örneği -->
<action application="set" data="api_hangup_hook=uuid_audio_fork ${uuid} start mute plain ws://localhost:8765"/>
```

## 🔧 Teknik Detaylar

### Ses Formatı

- **Gelen Ses**: PCM16LE, 16kHz, Mono
- **Chunk Boyutu**: 320 byte (20ms @ 16kHz)
- **VAD Eşikleri**: Dinamik (ortam gürültüsüne göre)

### Retry Mekanizmaları

- **STT**: 3 deneme, 0.5 saniye arayla
- **TTS**: 2 deneme, rate limit için bekleme
- **Chatbot**: 3 deneme, WebSocket bağlantı hatası durumunda HTTP fallback

### Hata Yönetimi

- Tüm modüllerde try-except blokları
- Detaylı loglama (`project_log.txt`)
- Graceful degradation (bir servis çökerse diğerleri çalışmaya devam eder)

### Performans Optimizasyonları

- Async/await ile non-blocking I/O
- Ses buffer'ları için deque kullanımı
- Geçici dosya otomatik temizliği
- Streaming ASR desteği (partial transcript)

## 📝 Loglama

Tüm işlemler `project_modules/project_log.txt` dosyasına loglanır:

```
2024-01-24 10:30:15 | [WS] Audio handler listening on ws://0.0.0.0:8765
2024-01-24 10:30:20 | Transkript (speech_20240124_103020.wav - 2.5s): (1.2sn) Merhaba nasılsın
2024-01-24 10:30:21 | [CHATBOT] WS send: Merhaba nasılsın
2024-01-24 10:30:22 | [CHATBOT] WebSocket response: Merhaba, ben iyiyim teşekkür ederim
```

## 🐛 Sorun Giderme

### WebSocket Bağlantı Sorunları
- Port 8765'in açık olduğundan emin olun
- FreeSWITCH'ten `ws://localhost:8765` adresine erişilebildiğini kontrol edin

### GPU Sunucusu Bağlantı Hataları
- Ngrok URL'inin güncel olduğunu kontrol edin
- GPU sunucusunun çalışır durumda olduğundan emin olun
- `/transcribe` endpoint'inin doğru olduğunu doğrulayın

### Chatbot Bağlantı Sorunları
- WebSocket bağlantısı başarısız olursa otomatik olarak HTTP'ye geçer
- Ngrok URL'lerinin güncel olduğunu kontrol edin
- Chatbot servisinin canlıda çalıştığını doğrulayın

### TTS Sorunları
- ElevenLabs API anahtarının geçerli olduğunu kontrol edin
- Rate limit aşılıyorsa otomatik bekleme yapılır
- FFmpeg'in kurulu olduğundan emin olun

## 📚 Referanslar

- [FreeSWITCH Documentation](https://freeswitch.org/confluence/)
- [ElevenLabs API](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [WebRTC VAD](https://github.com/wiseman/py-webrtcvad)
- [Whisper ASR](https://github.com/openai/whisper)

## 📄 Lisans

Bu proje özel kullanım içindir.
