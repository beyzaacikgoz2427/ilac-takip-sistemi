# Kişisel İlaç Takip Sistemi (Akıllı Sağlık Ekosistemi)

Çok paydaşlı mimariye sahip, gamification ve AI destekli ilaç uyum ve takip sistemi. 
Hasta, doktor ve eczane paydaşlarını tek bir dijital ekosistemde birleştirir.

> **Uyarı:** Bu sistem tıbbi cihaz veya karar destek sistemi değildir. Hatırlatıcı ve bilgilendirici yardımcı araçtır.
>  AI çıktıları tanı veya tedavi önerisi içermez: "Bilgilendirme amaçlıdır, tıbbi tavsiye değildir."

## Temel Özellikler

### Hasta (Mobil)
- Profil, alerji ve ilaç listesi yönetimi
- Push bildirim (FCM), SMS (Twilio) ve sesli (TTS) ilaç hatırlatma; "Aldım" / "Ertele" aksiyonları
- İlaç stok takibi, "kaç gün yeter" hesabı ve 3 gün kala uyarı
- "İlacım Nerede?" ile yakındaki eczane arama (OpenStreetMap + Leaflet)
- Puan, rozet ('İstikrarlı Kullanıcı' 7 gün, 'Sağlık Şampiyonu' 30 gün) ve alışkanlık zinciri
- Sağlık günlüğü; manuel glukoz, tansiyon ve nabız girişi
- Günlük / haftalık / aylık uyum grafikleri
- Offline-first: internet olmadan ilaç listesi ve yerel bildirimler

### Doktor (Web)
- Hastaya özel reçete ve dozaj girişi (dozaj yalnızca doktor yetkisinde)
- Reçete anında alerji ve etkileşim uyarıları
- Hasta ilaç geçmişi, yan etki kayıtları ve uyum oranı (%)
- Gerçek zamanlı hasta izleme ve kritik değer alarmı
- Yalnızca kendi hastalarına erişim (RBAC)

### Eczane / Otomat
- İlaç başına ayrı stok yönetim paneli, kritik seviye bildirimi
- QR / e-reçete doğrulama (süresi geçmiş ve mükerrer reçete engeli)
- FSM tabanlı sanal ilaç otomatı
- Soğuk zincir (2–8°C) simülasyonu ve "Servis Dışı" modu
- Reçete rengi sınıflandırması (Beyaz, Turuncu, Mor; Yeşil/Kırmızı için eczaneye yönlendirme)

### Yapay Zeka
- Kural tabanlı ilaç-ilaç, ilaç-besin ve ilaç-alerji kontrolü
- Düşük / Orta / Yüksek risk derecelendirmesi
- Yüksek riskte ilaç teslimini durdurma ve "Doktor onayı gereklidir" uyarısı

### Güvenlik ve Uyumluluk
- AES-256 (durağan veri), TLS 1.2+ (aktarım), BCrypt (parola)
- JWT (AccessToken + RefreshToken, sunucu taraflı kara liste) ve RBAC
- Tamper-evident audit log
- KVKK / GDPR: sağlık verisi için ayrı açık rıza, silme/düzeltme/aktarma talepleri (30 gün)
- Üçüncü taraf servislere (Twilio vb.) yalnızca iletişim bilgisi aktarımı
- Biyometrik veri toplanmaz

## Mimari ve Teknolojiler

| Katman               |                              Teknoloji                                 
| Veritabanı (merkezi) | SQL Server veya PostgreSQL, AES-256 şifreli sütunlar, 3NF 
| Veritabanı (yerel)   | SQLite (mobil, offline-first) 
| Backend              | .NET Core veya Node.js (REST API), JWT, RBAC, node-cron / Quartz 
| Web                  | React.js, SignalR / WebSocket, Chart.js / Recharts, Leaflet.js 
| Mobil                | Flutter, FCM, Flutter TTS, mobile_scanner, Keystore/Keychain 
| AI                   | Python 3.10+, scikit-learn, Pandas, XGBoost (prototip); gevşek bağlı REST mikroservis 
| Otomat Simülasyonu   | React kiosk arayüzü, FSM, react-qr-reader / ZXing, Lottie / CSS, sanal POS 
| Bildirim             | Firebase Cloud Messaging, Twilio API 
| Harita               | OpenStreetMap, Leaflet.js 
| Raporlama            | Chart.js, ReportLab (PDF), openpyxl (Excel) 
| Test                 | Postman, Jest, pytest, JMeter, OWASP ZAP, Selenium / Appium 
| Standartlar          | HL7 FHIR R4 (MedicationRequest), IEEE 830 / 29148, WCAG 2.1 AA 

### Veritabanı
- Merkezi veritabanı tüm paydaşların senkronizasyonunu sağlar
- Hassas sütunlar (ilaç adı, tanı, alerji, TC Kimlik) AES-256 ile şifrelenir
- İlaç kayıtları FHIR MedicationRequest formatını referans alır
- Senkronizasyon zaman damgalı; çakışmada sunucu doğrulaması ve en güncel zaman damgası esas alınır

### Web
- Doktor ve eczane panelleri ayrı rollerle JWT + RBAC üzerinden çalışır
- Responsive tasarım (masaüstü, tablet, mobil tarayıcı)

### Mobil
- Offline-first: SQLite okuma, yazma kuyruğu, bağlantı gelince senkronizasyon
- Şifreleme anahtarları Keystore (Android) / Keychain (iOS) içinde saklanır
- Android 10+ ve iOS 14+; dokunmatik hedefler min. 48x48 dp

### AI
- Deterministik kural katmanı kritik kararları verir (doz aşımı, alerji)
- ML katmanı isteğe bağlıdır; veri seti DrugBank / OpenFDA / RxNorm kaynaklıdır
- Hedef yanıt süresi ≤ 200 ms

### Otomat Simülasyonu
- Ödeme öncesi stok rezervasyonu; ödeme başarısızsa iptal
- Teslimde stok düşümü (StockCount-1), eş zamanlı işlemlerde kilit mekanizması
- COLLECTION aşamasında 60 sn zaman aşımı
- Soğuk zincir verisi rastgele üretilen simüle sıcaklıklardan oluşur
- AI "Kritik Risk" döndürürse ödeme alınmış olsa da teslimat durdurulur

  
