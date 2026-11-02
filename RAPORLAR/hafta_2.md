# Hafta 2 raporu

-ER diyagramı düzeltildi.

-Veritabanı MSSQL de yazıldı.

-ER diyagramındaki eksik, fazla,hatalı tablolar ve ilişkiler düzeltildi.

-Kullanıcılar tablosu hibrit yapıya geçirilerek doktorlar, hastalar, eczacılar tabloları oluşturuldu ve buna bağlı ilişkiler düzeltildi.Kimlik doğrulama, rıza, denetim ve token yönetimi tek Kullanicilar tablosunda toplanmış, role özgü alanlar Hastalar, Doktorlar, Eczacilar profil tablolarına ayrılmıştır. Böylece bir reçetenin doktoru olarak hastanın gösterilmesi veritabanı seviyesinde imkânsızdır.

-Yeni tablolar eklendi. Gereksiz tablolar çıkartıldı

-Bazı tablo isimleri karışıklığı ve yükü önlemek adına değiştirildi.

-Farklı tablolardaki aynı işe dair niteliklerden uygun olanı kaldırılarak veri tekrarıve uyuşmazlığı kaldırıldı.


# Not:

-KVKK gereği kilit ekranı bildirimlerinde ilaç ismi geçemez. Dış servise (Twilio/FCM) giden metin jenerik olsun: “İlaç saatiniz geldi.” Servisi INotificationSender arayüzüyle soyutla, demoda mock kullan (Twilio trial sadece doğrulanmış numaralara gönderir). E-posta için Twilio’nun SendGrid’i ayrı bir servistir.

-TCNo direkt tutulamaz. TCNoHash şeklinde tutulabilir.

-TCNo ya göre sorgu yapmak istenirse TCNoSifreli kullanılır. TC kimlik numarası 11 haneli küçük bir arama uzayına sahip olduğundan düz hash kolayca kaba kuvvetle çözülebilir. Arama ve tekillik için anahtarlı HMAC, geri okuma gerektiren durumlar için AES-256 kullanılmıştır.

-Parti seçimi FEFO olmalı. SKT si en yakın olanı ver. SKT si geçmiş olanı asla verme.

-Reçetenin mükerrer kullanımı için reçete kontrolünü ve IlacCikislari insert’ini aynı transaction’da UPDLOCK ile yap. Şemada reçete için ayrı bir “kullanıldı” bayrağı yok, bu bilgi IlacCikislari’ndan türetiliyor.

-Senkron imlecini sunucu zamanından üret, istemci saatine güvenme. GuncellemeZamani sunucuda atansın.

-Saatleri UTC sakla, Türkiye’de sabit UTC+3 olduğu için basit kalır. IlacSaatleri.Saat yerel saat, PlanlananZaman UTC.

-SKT takibi, FEFO dağıtımı ve geri çağırma (recall) izlenebilirliği için stok ilaç değil parti düzeyinde tutulmuştur.

-Bir ilaç çıkışı ve stok hareketi ya bir otomata ya da bir eczaneye aittir. Bu özel-ya-da kuralı veri katmanında kısıtla garanti edilmiştir.

-IlacCikislari.Tutar ve Hastalar.Puan her sorguda SUM maliyetini önlemek için kasıtlı önbellek alanlardır, kalem ve hareket tablolarıyla aynı transaction’da güncellenir. Diğer tüm yapı 3NF’dir.

-Denetim kayıtları, önceki kaydın hash’ini içerdiğinden değiştirilmeye karşı delil niteliği taşır (tamper-evident). Bu, mHealth güvenlik literatüründeki değişmez log gereksinimine karşılık gelir.

-Sağlık verisi özel nitelikli olduğundan genel kullanım koşullarından ayrı açık rıza (RizaKayitlari) alınır. Üçüncü taraf servislere yalnızca iletişim bilgisi gider, ilaç adı gitmez (veri minimizasyonu)

-Sistem tanı veya tedavi belirlemez, hatırlatıcı ve takip aracıdır. Etkileşim uyarıları tıbbi tavsiye değildir ve her uyarıda yasal uyarı gösterilir.

-Hata toleransı sıfır olan kontroller (alerji, doz, bilinen etkileşim) deterministik kurallarla, literatürde yeni ve olasılıksal riskler ML ile tahmin edilir. Kural motoru nihai karar verici, ML danışmandır




-Eczaneden fiziki teslimat bu fazda kapsam dışı bırakılmış, teslimat aksiyonları akıllı otomat modülüne odaklanmıştır.” (şemadaki XOR eczane desteğini “genişlemeye hazır mimari” olarak belirt.)

-Gerçek hasta verisi kullanılmamış, sentetik veriyle doğrulama yapılmıştır. Bu nedenle model doğruluğu klinik değil prototip düzeyindedir.

