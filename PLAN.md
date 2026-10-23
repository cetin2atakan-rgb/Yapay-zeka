# Yerel Yapay Zeka Asistanı — Tam Plan

Masaüstünde çalışan, sesli komutla yönetilen, video projeleri için toplu görsel
üreten, karakter tutarlılığı sağlayan ve diğer uygulamalarla entegre olan,
modüler (üzerine eklene eklene büyüyen) bir yapay zeka sistemi.

## 1. Hedef sistem

| Özellik | Açıklama |
|---|---|
| Sesli kontrol | Uyandırma kelimesi veya kısayol tuşuyla konuşarak komut verme, sesli cevap |
| Sohbet / yazı | Senaryo, başlık, prompt listesi üretimi — internetsiz |
| Toplu görsel | Tek komutla yüzlerce görseli sıraya alıp numaralı şekilde proje klasörüne üretme |
| Karakter tutarlılığı | Karakter kütüphanesi: LoRA + referans görsel ile her görselde aynı karakter |
| Çoklu stil | Gerçekçi ve çizim/stilize stiller; proje başına stil profili |
| Video | Gerekli sahneler bilgisayardaki **Grok bot** ile üretilir (yerel Wan 2.2 yedek seçenek) |
| Masaüstü entegrasyonu | Uygulama açma, dosya/klasör işlemleri, diğer programlarla köprüler |
| Genişletilebilirlik | Her yetenek ayrı bir "araç" dosyası; yeni araç = yeni dosya |

## 2. Donanım

- GPU: RTX 5080 16 GB (Blackwell — CUDA 12.8+, PyTorch cu128 gerekir; FP4 desteği var)
- RAM: 32 GB
- CPU: i7-14700KF
- OS: Windows

## 3. Mimari

```
            ┌───────────────── Arayüz (tepsi simgesi + yerel web penceresi) ─────────────────┐
            │                                                                                 │
 🎤 ──► Kulak (Whisper, CPU) ──► ÇEKİRDEK / Orkestratör (Python, FastAPI) ──► Ağız (Piper/Kokoro, CPU) ──► 🔊
                                       │
                         Beyin (Ollama · Qwen3 14B, GPU)
                         "ne yapılacak, hangi araç?"
                                       │
                ┌──────────────┬───────┴────────┬──────────────────┬───────────────┐
                ▼              ▼                ▼                  ▼               ▼
            Ressam         Karakter         Yönetmen            Eller          Yazar
         (ComfyUI API)    Kütüphanesi    (Grok bot köprüsü)   (Windows         (senaryo,
         toplu görsel     LoRA eğitimi    + yerel Wan yedek)   otomasyonu)     prompt listesi)
                │
         GPU Sırası (VRAM yöneticisi: Beyin ↔ Ressam ↔ Wan sırayla GPU kullanır)
```

### Temel prensipler
1. **Çekirdek küçük kalır.** Tüm yetenekler `araclar/` klasöründeki eklentilerdir.
2. **Motorlar değiştirilebilir.** Ressam bugün Flux kullanır, yarın daha iyi model
   çıkarsa sadece motor ayarı değişir; komutlar aynı kalır.
3. **Her şey yerel.** İnternet yalnızca kurulum ve Grok bot için gerekir.
4. **Her iş bir projeye bağlıdır.** Üretilen her dosya ilgili video projesinin klasörüne gider.

## 4. Teknoloji yığını

| Katman | Seçim | Not |
|---|---|---|
| Dil | Python 3.12 | |
| Çekirdek sunucu | FastAPI (localhost) | Arayüz ve araçlar bununla konuşur |
| Dil modeli | Ollama + Qwen3 14B (Q4) | Araç çağırma (tool calling) destekli, Türkçe iyi; alternatif Gemma 3 12B |
| Konuşma → yazı | faster-whisper, large-v3-turbo | CPU'da çalışır, VRAM harcamaz |
| Uyandırma kelimesi | openWakeWord | + bas-konuş kısayol tuşu |
| Yazı → konuşma | Piper (TR), Kokoro (EN) | CPU |
| Görsel motoru | ComfyUI (API modu) | Workflow'lar JSON olarak depoda tutulur |
| Ana görsel modeli | Flux.1 dev + Nunchaku FP4 | Gerçekçi ve genel kullanım |
| Stilize / hızlı | SDXL tabanlı modeller (+ stil LoRA'ları) | Çizim stilleri, hızlı taslak |
| Referans düzenleme | Flux Kontext dev, Qwen-Image-Edit | Eğitimsiz karakter tutarlılığı |
| LoRA eğitimi | AI-Toolkit (veya FluxGym) | Karakter başına tek seferlik |
| Video | Grok bot köprüsü; yedek: Wan 2.2 (GGUF + hızlandırma LoRA) | |
| Masaüstü otomasyonu | pywinauto, subprocess, watchdog | |
| Arayüz | pystray (tepsi) + yerel web arayüzü | |
| Ayarlar | YAML dosyaları | Stil profilleri, karakterler, kısayollar |

> Model sürümleri kurulum anında güncel durumla tekrar kontrol edilecek.

## 5. Depo klasör yapısı

```
Yapay-zeka/
├── PLAN.md
├── README.md
├── kurulum/                 # Windows kurulum scriptleri (PowerShell)
│   ├── 1_temel.ps1          # Python, Git, Ollama
│   ├── 2_comfyui.ps1        # ComfyUI + cu128 PyTorch + Nunchaku + düğümler
│   ├── 3_modeller.ps1       # Model indirme
│   └── kontrol.ps1          # Kurulum sağlık kontrolü
├── cekirdek/
│   ├── sunucu.py            # FastAPI orkestratör
│   ├── beyin.py             # Ollama bağlantısı, araç çağırma döngüsü
│   ├── gpu_sirasi.py        # VRAM yöneticisi
│   ├── is_kuyrugu.py        # Uzun işler (toplu üretim) kuyruğu
│   └── arac_yukleyici.py    # araclar/ klasörünü otomatik yükler
├── ses/
│   ├── kulak.py             # Whisper + uyandırma kelimesi
│   └── agiz.py              # Piper / Kokoro
├── araclar/                 # ← üzerine eklenen her yetenek burada
│   ├── ressam.py            # Tek / toplu görsel üretimi
│   ├── karakter.py          # Karakter kütüphanesi, LoRA eğitimi
│   ├── yonetmen.py          # Grok bot köprüsü + yerel Wan
│   ├── dosya.py             # Klasör/dosya işlemleri
│   ├── uygulama.py          # Uygulama açma/kontrol
│   └── yazar.py             # Senaryodan prompt listesi çıkarma
├── workflows/               # ComfyUI workflow JSON'ları
├── ayarlar/
│   ├── genel.yaml
│   ├── stiller/             # Stil profilleri (gercekci.yaml, cizim.yaml …)
│   └── karakterler/         # Karakter tanımları
├── arayuz/                  # Tepsi simgesi + web arayüzü
└── testler/
```

Video projeleri depo dışında, ayarlanabilir bir klasörde tutulur:

```
D:\Projeler\<proje-adı>\
├── proje.yaml               # stil profili, karakterler, çözünürlük
├── senaryo.txt
├── promptlar.txt            # her satır bir görsel
├── gorseller\001.png, 002.png …
├── videolar\
└── kayit.log                # hangi görsel hangi prompt/seed ile üretildi
```

## 6. Aşamalar

Her aşama sonunda kullanılabilir bir sistem olur.

### Aşama 1 — Temel + Beyin + araç iskeleti
- Kurulum scriptleri (Python, Ollama, Qwen3 14B)
- Çekirdek sunucu, araç yükleyici, araç çağırma döngüsü
- İlk basit araçlar: `dosya` (klasör oluştur, listele), `uygulama` (program aç)
- Yazılı sohbet arayüzü
- **Bitti sayılır:** "Masaüstünde Deneme adında klasör aç" yazınca klasör oluşur.

### Aşama 2 — Ses
- Kulak (Whisper, CPU) + bas-konuş kısayolu + uyandırma kelimesi
- Ağız (Piper Türkçe)
- **Bitti sayılır:** Konuşarak verilen komut çalışır ve sesli cevap gelir.

### Aşama 3 — Ressam (görsel üretimi)
- ComfyUI kurulumu (RTX 50 uyumlu), Flux dev + Nunchaku FP4, SDXL
- GPU sırası: görsel üretimi başlarken Beyin bellekten çıkarılır, sonra geri yüklenir
- Tek görsel ve **toplu üretim**: `promptlar.txt` → numaralı görseller, gece çalışabilir
- Yeniden üretim: "37 ve 112'yi yeniden üret"
- Stil profilleri (gerçekçi / çizim)
- Hız ölçümü (hedef: Flux ile 500 görsel ≈ 1 saat)
- **Bitti sayılır:** Sesli komutla 500 satırlık prompt listesi sorunsuz üretilir.

### Aşama 4 — Karakter tutarlılığı
- Karakter kütüphanesi (`ayarlar/karakterler/`)
- Referans görselle üretim (Flux Kontext / Qwen-Image-Edit) — eğitimsiz, hızlı
- LoRA eğitimi aracı: referans görsellerden karakter LoRA'sı (~30–60 dk)
- Prompt'ta karakter adı geçince ilgili LoRA/referans otomatik eklenir
- **Bitti sayılır:** Bir karakter 50 farklı sahnede tanınabilir şekilde aynı çıkar.

### Aşama 5 — Yazar (senaryodan prompt'a)
- Senaryoyu sahnelere bölme, her sahne için görsel prompt'u yazma
- Proje stil profiline ve karakterlere uygun prompt üretimi
- **Bitti sayılır:** "Bu senaryodan görselleri çıkar ve üret" tek komutla çalışır.

### Aşama 6 — Yönetmen (video)
- **Grok bot köprüsü:** Grok bot'a sahne/prompt gönderme, çıkan videoyu proje klasörüne alma
- Yedek: yerel Wan 2.2 ile görselden kısa klip (~2–5 dk / 5 sn)
- **Bitti sayılır:** "12. sahneyi videoya çevir" komutu videoyu `videolar\` klasörüne getirir.

### Aşama 7+ — Genişletme
- Montaj programları entegrasyonu (CapCut / DaVinci Resolve için klasör/zaman çizelgesi hazırlığı)
- Kendi belgelerinizle soru-cevap (RAG)
- Thumbnail araçları, toplu yeniden boyutlandırma, altyazı
- İhtiyaç çıktıkça yeni araçlar

## 7. Sesli komut örnekleri (hedef)

- "Yeni proje aç: Kış Sığınağı, çizim stili."
- "Bu senaryodan görsel listesini çıkar."
- "Görselleri üretmeye başla, bitince haber ver."
- "37, 112 ve 240'ı yeniden üret, daha karanlık olsun."
- "Ahmet karakterini referans görsellerden eğit."
- "15. sahneyi Grok ile videoya çevir."
- "Proje klasörünü aç."

## 8. GPU / bellek yönetimi

16 GB VRAM'e Beyin (~9 GB) ve Flux (~10–12 GB) aynı anda sığmaz. `gpu_sirasi.py`:
- Aynı anda yalnızca bir ağır model GPU'da olur.
- Toplu üretim sırasında Beyin bellekten çıkar; komutlar kuyruğa alınır veya kısa
  cevaplar için küçük bir model / CPU kullanılır.
- Ses (Whisper, Piper) her zaman CPU'da çalışır — GPU'yu meşgul etmez.

## 9. Tahmini performans (RTX 5080)

| İş | Tahmin |
|---|---|
| Flux 1024 px, 1 görsel | ~5–8 sn (Nunchaku FP4), ~15–20 sn (normal) |
| 500 görsel, Flux | ~45 dk – 1 saat |
| 500 görsel, SDXL | ~20 dk |
| Karakter LoRA eğitimi | ~30–60 dk |
| Wan 2.2, 5 sn klip | ~2–5 dk |

Aşama 3'te gerçek ölçümlerle güncellenecek.

## 10. Riskler

| Risk | Önlem |
|---|---|
| RTX 50 serisi uyumsuz paketler | cu128 PyTorch, sürümleri sabitlenmiş kurulum scriptleri |
| VRAM yetersizliği | GPU sırası, FP4/GGUF sıkıştırma |
| Toplu üretimde yarıda kesilme | Kuyruk kaldığı yerden devam eder, `kayit.log` |
| Türkçe ses tanıma hataları | Komut onayı ("15. sahne mi dediniz?"), özel kelime listesi |
| Grok bot'un yapısının bilinmemesi | Aşama 6 öncesi netleştirilecek (aşağıda) |

## 11. Açık sorular

1. **Grok bot nasıl çalışıyor?** Kendi yazdığınız bir script mi, hazır bir program mı?
   Nasıl girdi alıyor (klasör, komut satırı, arayüz) ve videoyu nereye kaydediyor?
   Köprüyü buna göre yazacağız. (Aşama 6'ya kadar zaman var.)
2. **Projeler hangi diskte tutulsun?** (Örn. `D:\Projeler`)
3. **Uyandırma kelimesi** ne olsun?
