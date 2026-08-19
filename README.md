# İstanbul Konser/Etkinlik Web Scraper

İstanbul'daki konser ve etkinlik bilgilerini **Biletix**, **Passo**,
**Biletinial** ve **İBB Kültür Sanat** sitelerinden toplayan, verileri
**JSON** ve **SQLite** olarak kaydeden bir Python web scraper projesidir.

Toplanan bilgiler: sanatçı/grup adı, etkinlik adı, tarih ve saat, mekan ve
ilçe, müzik türü, bilet fiyat aralığı (min–max) / kategori bazlı fiyatlar,
ücretsiz olup olmadığı, kampanya/indirim bilgisi, bilet linki ve görsel URL'si.

---

## 📁 Proje Yapısı

```
istanbul_events/
├── scraper.py          # Ana script — çıktıyı events.json'a yazar
├── scraper_sqlite.py   # Aynı mantık — çıktıyı events.db (SQLite) veritabanına yazar
├── requirements.txt    # Python bağımlılıkları
├── README.md           # Bu dosya
├── events.json         # Örnek çıktı yapısı (dummy veri)
└── scraper.log         # Çalışma sırasında otomatik oluşur (loglar)
```

---

## ⚙️ Kurulum

1. (Önerilir) Sanal ortam oluşturun ve etkinleştirin:

   ```bash
   python3 -m venv venv
   source venv/bin/activate      # Windows: venv\Scripts\activate
   ```

2. Bağımlılıkları kurun:

   ```bash
   pip install -r requirements.txt
   ```

3. Playwright tarayıcılarını kurun (dinamik/JS sayfalar için gereklidir):

   ```bash
   playwright install chromium
   ```

   > Playwright yüklü değilse script yine çalışır; sadece dinamik içerik
   > çekme adımı atlanır ve bu durum loglanır.

---

## 🚀 Çalıştırma

### JSON versiyonu

```bash
python3 scraper.py
```

Çıktı `events.json` dosyasına yazılır. Her çalıştırmada:

- **Yeni** etkinlikler eklenir,
- **Mevcut** etkinliklerin fiyat/kampanya bilgisi güncellenir,
- **Tarihi geçmiş** etkinlikler silinir.

### SQLite versiyonu

```bash
python3 scraper_sqlite.py
```

Çıktı `events.db` SQLite veritabanına yazılır. Aynı ekle/güncelle/sil mantığı
`INSERT OR REPLACE` ve `delete_past_events()` ile uygulanır.

Veritabanını incelemek için:

```bash
sqlite3 events.db "SELECT event_name, date, venue, price_min FROM events;"
```

---

## 📤 Çıktı Formatı (JSON Şeması)

`events.json`, aşağıdaki alanlara sahip etkinlik nesnelerinden oluşan bir
dizidir:

| Alan               | Tip              | Açıklama                                        |
|--------------------|------------------|-------------------------------------------------|
| `id`               | string           | `md5(isim + tarih)` ile üretilen benzersiz kimlik |
| `artist`           | string           | Sanatçı / grup adı                              |
| `event_name`       | string           | Etkinlik adı                                    |
| `date`             | string           | Tarih (tercihen `YYYY-MM-DD`)                   |
| `time`             | string           | Saat (örn. `21:00`)                             |
| `venue`            | string           | Mekan adı                                       |
| `district`         | string           | İlçe                                            |
| `genre`            | string           | Müzik türü / kategori                           |
| `price_min`        | number \| null   | En düşük fiyat                                  |
| `price_max`        | number \| null   | En yüksek fiyat                                 |
| `price_categories` | object           | Kategori bazlı fiyatlar `{"kategori": fiyat}`   |
| `is_free`          | boolean          | Ücretsiz mi?                                    |
| `discount_info`    | string           | Tespit edilen kampanya/indirim metni            |
| `ticket_url`       | string           | Bilet satın alma / detay linki                  |
| `image_url`        | string           | Etkinlik görseli URL'si                         |
| `source`           | string           | Kaynak site (`biletix`, `passo`, `biletinial`, `ibb`) |
| `scraped_at`       | string           | Verinin çekildiği ISO zaman damgası             |

### Örnek

```json
{
  "id": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6",
  "artist": "Sezen Aksu",
  "event_name": "Sezen Aksu Konseri",
  "date": "2026-09-15",
  "time": "21:00",
  "venue": "Harbiye Cemil Topuzlu Açıkhava Sahnesi",
  "district": "Şişli",
  "genre": "Pop",
  "price_min": 750.0,
  "price_max": 2500.0,
  "price_categories": {"Arka Blok": 750.0, "Ön Blok": 2500.0},
  "is_free": false,
  "discount_info": "%20 erken rezervasyon indirimi",
  "ticket_url": "https://www.biletix.com/etkinlik/sezen-aksu-2026/TURKIYE/tr",
  "image_url": "https://i.ytimg.com/vi/bkZ_k3xkbJA/maxresdefault.jpg",
  "source": "biletix",
  "scraped_at": "2026-08-19T10:30:00"
}
```

SQLite tablosunda ek olarak `raw_data` sütunu bulunur; bu sütun etkinliğin tüm
alanlarını JSON string olarak saklar.

---

## 🛡️ Öne Çıkan Teknik Özellikler

- **robots.txt kontrolü** — Her site için çekme izni `urllib.robotparser` ile
  kontrol edilir.
- **User-Agent rotasyonu** — Her istekte farklı bir tarayıcı kimliği seçilir.
- **Rate limiting** — İstekler arasında `random.uniform(2, 5)` saniye beklenir.
- **Statik + dinamik** — Önce `requests` + `BeautifulSoup`, gerekirse Playwright
  (async, `headless=True`, `timeout=30000`) denenir.
- **Kampanya tespiti** — `%\d+`, "indirim", "kampanya", "fırsat", "erken
  rezervasyon" gibi ifadeler regex ile yakalanır.
- **Duplicate önleme** — `md5(isim + tarih)` tabanlı benzersiz ID.
- **Sağlam hata yönetimi** — Tüm exception'lar loglanır; script asla çökmez.
- **Loglama** — Hem konsola hem `scraper.log` dosyasına (INFO seviyesi).

---

## 🔧 Olası Sorunlar ve Çözümleri

| Sorun | Açıklama / Çözüm |
|-------|------------------|
| **Bot koruması (403 / Cloudflare)** | Site otomatik erişimi engelleyebilir. Script bunu tespit edip uyarı loglar ve o kaynak için boş liste döner. Çözüm: istek sıklığını azaltın, proxy kullanın veya resmi API'yi tercih edin. |
| **CAPTCHA sayfası** | Sayfa içeriğinde CAPTCHA tespit edilirse ilgili kaynak atlanır. Otomatik çözüm önerilmez; manuel/oturumlu erişim gerekir. |
| **HTML yapısı değişti** | Siteler tasarım değiştirdiğinde CSS seçicileri (`div.event-card` vb.) güncellenmelidir. Hangi aşamada kaç kart bulunduğu `scraper.log`'da görülür; seçicileri güncel HTML'e göre düzenleyin. |
| **Playwright hataları** | `playwright install chromium` komutunu çalıştırdığınızdan emin olun. Sunucu ortamlarında ek sistem kütüphaneleri gerekebilir (`playwright install-deps`). |
| **Boş çıktı** | Loglarda "0 kart bulundu" görüyorsanız seçiciler güncel değildir veya içerik JS ile yükleniyordur (Playwright'ın kurulu olduğundan emin olun). |
| **Tarih ayrıştırma** | Geçmiş etkinlik temizliği yaygın tarih formatlarını dener. Farklı bir format varsa `_tarih_gecmis_mi()` fonksiyonuna kalıp ekleyin. |

---

## ⚖️ Yasal Uyarı

Bu araç yalnızca **eğitim ve kişisel kullanım** amaçlıdır. Kullanmadan önce:

- Hedef sitelerin **`robots.txt`** kurallarına ve **Kullanım Koşullarına (ToS)**
  uyduğunuzdan emin olun. Bazı siteler otomatik veri toplamayı açıkça
  yasaklayabilir.
- Sunucuları aşırı yüklememek için **rate limiting**'i devre dışı bırakmayın.
- Toplanan verilerin **telif hakkı** ve **kişisel veri** mevzuatına
  (örn. KVKK) uygun şekilde kullanılması sizin sorumluluğunuzdadır.
- Ticari kullanım için ilgili sitelerin **resmi API**'lerini veya yazılı
  iznini tercih edin.

Kod tarafından yapılan varsayılan CSS seçicileri örnektir; gerçek veriler için
sitelerin güncel HTML yapısına göre uyarlanmalıdır.
