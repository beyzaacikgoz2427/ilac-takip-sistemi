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



# Veritabanı


SET QUOTED_IDENTIFIER ON;
SET ANSI_NULLS ON;
GO
 
/* ===== LOOKUP / KIMLIK ===== */
 
CREATE TABLE Roller (
    RolID   INT IDENTITY(1,1) NOT NULL,
    RolAdi  NVARCHAR(50)      NOT NULL,
    CONSTRAINT PK_Roller PRIMARY KEY (RolID),
    CONSTRAINT UQ_Roller_RolAdi UNIQUE (RolAdi)
);
 
CREATE TABLE Kullanicilar (
    KullaniciID       INT IDENTITY(1,1) NOT NULL,
    RolID             INT               NOT NULL,
    TCNoHash          BINARY(32)        NOT NULL, -- HMAC-SHA256
    TCNoSifreli       VARBINARY(256)    NOT NULL, -- AES-256
    Ad                NVARCHAR(100)     NOT NULL,
    Soyad             NVARCHAR(100)     NOT NULL,
    SifreHash         VARCHAR(255)      NOT NULL,
    DogumTarihi       DATE              NOT NULL,
    Cinsiyet          NVARCHAR(10)      NOT NULL,
    Telefon           VARCHAR(20)       NOT NULL,
    Eposta            NVARCHAR(256)     NOT NULL,
    OlusturulmaTarihi DATETIME2         NOT NULL CONSTRAINT DF_Kullanicilar_OlusturulmaTarihi DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_Kullanicilar PRIMARY KEY (KullaniciID),
    CONSTRAINT UQ_Kullanicilar_TCNoHash UNIQUE (TCNoHash),
    CONSTRAINT UQ_Kullanicilar_Eposta UNIQUE (Eposta),
    CONSTRAINT FK_Kullanicilar_Roller FOREIGN KEY (RolID) REFERENCES dbo.Roller (RolID)
);
 
CREATE TABLE Doktorlar (
    KullaniciID   INT           NOT NULL,
    DiplomaNo     VARCHAR(30)   NOT NULL,
    UzmanlikAlani NVARCHAR(100) NULL,
    HastaneAdi    NVARCHAR(200) NULL,
    CONSTRAINT PK_Doktorlar PRIMARY KEY (KullaniciID),
    CONSTRAINT UQ_Doktorlar_DiplomaNo UNIQUE (DiplomaNo),
    CONSTRAINT FK_Doktorlar_Kullanicilar FOREIGN KEY (KullaniciID) REFERENCES dbo.Kullanicilar (KullaniciID)
);
 
CREATE TABLE Eczacilar (
    KullaniciID      INT         NOT NULL,
    EczaneOdaSicilNo VARCHAR(30) NOT NULL,
    CONSTRAINT PK_Eczacilar PRIMARY KEY (KullaniciID),
    CONSTRAINT UQ_Eczacilar_EczaneOdaSicilNo UNIQUE (EczaneOdaSicilNo),
    CONSTRAINT FK_Eczacilar_Kullanicilar FOREIGN KEY (KullaniciID) REFERENCES dbo.Kullanicilar (KullaniciID)
);
 
CREATE TABLE Hastalar (
    KullaniciID     INT           NOT NULL,
    KanGrubu        NVARCHAR(5)   NULL,
    AcilDurumKisisi NVARCHAR(200) NULL,
    Puan            INT           NOT NULL CONSTRAINT DF_Hastalar_Puan DEFAULT 0,
    GuncelSeri      INT           NOT NULL CONSTRAINT DF_Hastalar_GuncelSeri DEFAULT 0,
    EnUzunSeri      INT           NOT NULL CONSTRAINT DF_Hastalar_EnUzunSeri DEFAULT 0,
    CONSTRAINT PK_Hastalar PRIMARY KEY (KullaniciID),
    CONSTRAINT CK_Hastalar_Seri CHECK (GuncelSeri >= 0 AND EnUzunSeri >= 0 AND EnUzunSeri >= GuncelSeri),
    CONSTRAINT FK_Hastalar_Kullanicilar FOREIGN KEY (KullaniciID) REFERENCES dbo.Kullanicilar (KullaniciID)
);
 
CREATE TABLE DoktorHasta (
    DoktorID        INT  NOT NULL,
    HastaID         INT  NOT NULL,
    BaslangicTarihi DATE NOT NULL CONSTRAINT DF_DoktorHasta_BaslangicTarihi DEFAULT CAST(SYSUTCDATETIME() AS DATE),
    AktifMi         BIT  NOT NULL CONSTRAINT DF_DoktorHasta_AktifMi DEFAULT 1,
    ErisimOnayi     BIT  NOT NULL CONSTRAINT DF_DoktorHasta_ErisimOnayi DEFAULT 0,
    CONSTRAINT PK_DoktorHasta PRIMARY KEY (DoktorID, HastaID),
    CONSTRAINT FK_DoktorHasta_Doktorlar FOREIGN KEY (DoktorID) REFERENCES dbo.Doktorlar (KullaniciID),
    CONSTRAINT FK_DoktorHasta_Hastalar  FOREIGN KEY (HastaID)  REFERENCES dbo.Hastalar (KullaniciID)
);
 
/* ===== ALERJI / ETKEN MADDE / ILAC ===== */
 
CREATE TABLE Alerjenler (
    AlerjenID  INT IDENTITY(1,1) NOT NULL,
    AlerjenAdi NVARCHAR(150)     NOT NULL,
    CONSTRAINT PK_Alerjenler PRIMARY KEY (AlerjenID),
    CONSTRAINT UQ_Alerjenler_AlerjenAdi UNIQUE (AlerjenAdi)
);
 
CREATE TABLE EtkenMaddeler (
    EtkenMaddeID  INT IDENTITY(1,1) NOT NULL,
    EtkenMaddeAdi NVARCHAR(150)     NOT NULL,
    CONSTRAINT PK_EtkenMaddeler PRIMARY KEY (EtkenMaddeID),
    CONSTRAINT UQ_EtkenMaddeler_EtkenMaddeAdi UNIQUE (EtkenMaddeAdi)
);
 
CREATE TABLE AlerjenEtkenMaddeler (
    AlerjenID    INT NOT NULL,
    EtkenMaddeID INT NOT NULL,
    CONSTRAINT PK_AlerjenEtkenMaddeler PRIMARY KEY (AlerjenID, EtkenMaddeID),
    CONSTRAINT FK_AlerjenEtkenMaddeler_Alerjenler    FOREIGN KEY (AlerjenID)    REFERENCES dbo.Alerjenler (AlerjenID),
    CONSTRAINT FK_AlerjenEtkenMaddeler_EtkenMaddeler FOREIGN KEY (EtkenMaddeID) REFERENCES dbo.EtkenMaddeler (EtkenMaddeID)
);
 
CREATE TABLE HastaAlerjileri (
    HastaID   INT NOT NULL,
    AlerjenID INT NOT NULL,
    CONSTRAINT PK_HastaAlerjileri PRIMARY KEY (HastaID, AlerjenID),
    CONSTRAINT FK_HastaAlerjileri_Hastalar   FOREIGN KEY (HastaID)   REFERENCES dbo.Hastalar (KullaniciID),
    CONSTRAINT FK_HastaAlerjileri_Alerjenler FOREIGN KEY (AlerjenID) REFERENCES dbo.Alerjenler (AlerjenID)
);
 
CREATE TABLE IlacFormlari (
    FormID  INT IDENTITY(1,1) NOT NULL,
    FormAdi NVARCHAR(50)      NOT NULL,
    CONSTRAINT PK_IlacFormlari PRIMARY KEY (FormID),
    CONSTRAINT UQ_IlacFormlari_FormAdi UNIQUE (FormAdi)
);
 
CREATE TABLE ReceteRenkleri (
    RenkID               INT IDENTITY(1,1) NOT NULL,
    RenkAdi              NVARCHAR(50)      NOT NULL,
    OtomattanVerilebilir BIT               NOT NULL CONSTRAINT DF_ReceteRenkleri_OtomattanVerilebilir DEFAULT 0,
    CONSTRAINT PK_ReceteRenkleri PRIMARY KEY (RenkID),
    CONSTRAINT UQ_ReceteRenkleri_RenkAdi UNIQUE (RenkAdi)
);
 
CREATE TABLE Ilaclar (
    IlacID      INT IDENTITY(1,1) NOT NULL,
    TicariAd    NVARCHAR(200)     NOT NULL,
    FormID      INT               NOT NULL,
    RenkID      INT               NOT NULL,
    MinSicaklik DECIMAL(5,2)      NULL,
    MaxSicaklik DECIMAL(5,2)      NULL,
    CONSTRAINT PK_Ilaclar PRIMARY KEY (IlacID),
    CONSTRAINT CK_Ilaclar_Sicaklik CHECK (
        (MinSicaklik IS NULL AND MaxSicaklik IS NULL)
        OR (MinSicaklik IS NOT NULL AND MaxSicaklik IS NOT NULL AND MinSicaklik <= MaxSicaklik)
    ),
    CONSTRAINT FK_Ilaclar_IlacFormlari   FOREIGN KEY (FormID) REFERENCES dbo.IlacFormlari (FormID),
    CONSTRAINT FK_Ilaclar_ReceteRenkleri FOREIGN KEY (RenkID) REFERENCES dbo.ReceteRenkleri (RenkID)
);
 
CREATE TABLE IlacEtkenMaddeleri (
    IlacID       INT NOT NULL,
    EtkenMaddeID INT NOT NULL,
    CONSTRAINT PK_IlacEtkenMaddeleri PRIMARY KEY (IlacID, EtkenMaddeID),
    CONSTRAINT FK_IlacEtkenMaddeleri_Ilaclar       FOREIGN KEY (IlacID)       REFERENCES dbo.Ilaclar (IlacID),
    CONSTRAINT FK_IlacEtkenMaddeleri_EtkenMaddeler FOREIGN KEY (EtkenMaddeID) REFERENCES dbo.EtkenMaddeler (EtkenMaddeID)
);
 
CREATE TABLE Partiler (
    PartiID           INT IDENTITY(1,1) NOT NULL,
    IlacID            INT               NOT NULL,
    PartiNo           VARCHAR(50)       NOT NULL,
    SonKullanmaTarihi DATE              NOT NULL,
    CONSTRAINT PK_Partiler PRIMARY KEY (PartiID),
    CONSTRAINT UQ_Partiler_IlacID_PartiNo UNIQUE (IlacID, PartiNo),
    CONSTRAINT FK_Partiler_Ilaclar FOREIGN KEY (IlacID) REFERENCES dbo.Ilaclar (IlacID)
);
 
/* ===== ETKILESIM KURALLARI ===== */
 
CREATE TABLE RiskSeviyeleri (
    RiskSeviyeID  INT IDENTITY(1,1) NOT NULL,
    RiskSeviyeAdi NVARCHAR(50)      NOT NULL,
    CONSTRAINT PK_RiskSeviyeleri PRIMARY KEY (RiskSeviyeID),
    CONSTRAINT UQ_RiskSeviyeleri_RiskSeviyeAdi UNIQUE (RiskSeviyeAdi)
);
 
CREATE TABLE IlacEtkilesimKurallari (
    KuralID        INT IDENTITY(1,1) NOT NULL,
    EtkenMaddeA_ID INT               NOT NULL,
    EtkenMaddeB_ID INT               NOT NULL,
    RiskSeviyeID   INT               NOT NULL,
    Aciklama       NVARCHAR(500)     NULL,
    CONSTRAINT PK_IlacEtkilesimKurallari PRIMARY KEY (KuralID),
    CONSTRAINT CK_IlacEtkilesimKurallari_AB CHECK (EtkenMaddeA_ID < EtkenMaddeB_ID),
    CONSTRAINT UQ_IlacEtkilesimKurallari_AB UNIQUE (EtkenMaddeA_ID, EtkenMaddeB_ID),
    CONSTRAINT FK_IlacEtkilesimKurallari_EtkenA FOREIGN KEY (EtkenMaddeA_ID) REFERENCES dbo.EtkenMaddeler (EtkenMaddeID),
    CONSTRAINT FK_IlacEtkilesimKurallari_EtkenB FOREIGN KEY (EtkenMaddeB_ID) REFERENCES dbo.EtkenMaddeler (EtkenMaddeID),
    CONSTRAINT FK_IlacEtkilesimKurallari_RiskSeviyeleri FOREIGN KEY (RiskSeviyeID) REFERENCES dbo.RiskSeviyeleri (RiskSeviyeID)
);
 
CREATE TABLE BesinEtkilesimKurallari (
    BesinKuralID INT IDENTITY(1,1) NOT NULL,
    EtkenMaddeID INT               NOT NULL,
    BesinAdi     NVARCHAR(150)     NOT NULL,
    Aciklama     NVARCHAR(500)     NULL,
    CONSTRAINT PK_BesinEtkilesimKurallari PRIMARY KEY (BesinKuralID),
    CONSTRAINT UQ_BesinEtkilesimKurallari UNIQUE (EtkenMaddeID, BesinAdi),
    CONSTRAINT FK_BesinEtkilesimKurallari_EtkenMaddeler FOREIGN KEY (EtkenMaddeID) REFERENCES dbo.EtkenMaddeler (EtkenMaddeID)
);
 
/* ===== RECETE ===== */
 
CREATE TABLE Receteler (
    ReceteID          INT IDENTITY(1,1) NOT NULL,
    EReceteNo         VARCHAR(30)       NOT NULL,
    DoktorID          INT               NOT NULL,
    HastaID           INT               NOT NULL,
    GecerlilikTarihi  DATE              NOT NULL,
    OlusturmaTarihi   DATETIME2         NOT NULL CONSTRAINT DF_Receteler_OlusturmaTarihi DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_Receteler PRIMARY KEY (ReceteID),
    CONSTRAINT UQ_Receteler_EReceteNo UNIQUE (EReceteNo),
    CONSTRAINT FK_Receteler_DoktorHasta FOREIGN KEY (DoktorID, HastaID) REFERENCES dbo.DoktorHasta (DoktorID, HastaID)
);
 
CREATE TABLE ReceteKalemleri (
    ReceteKalemID INT IDENTITY(1,1) NOT NULL,
    ReceteID      INT               NOT NULL,
    IlacID        INT               NOT NULL,
    Doz           NVARCHAR(50)      NOT NULL,
    Adet          INT               NOT NULL,
    Siklik        NVARCHAR(50)      NOT NULL,
    CONSTRAINT PK_ReceteKalemleri PRIMARY KEY (ReceteKalemID),
    CONSTRAINT CK_ReceteKalemleri_Adet CHECK (Adet > 0),
    CONSTRAINT FK_ReceteKalemleri_Receteler FOREIGN KEY (ReceteID) REFERENCES dbo.Receteler (ReceteID),
    CONSTRAINT FK_ReceteKalemleri_Ilaclar   FOREIGN KEY (IlacID)   REFERENCES dbo.Ilaclar (IlacID)
);
 
/* ===== HASTA ILAC PLANI (SQLite SENKRON - GUID) ===== */
 
CREATE TABLE KullaniciIlaclari (
    KullaniciIlacID  UNIQUEIDENTIFIER NOT NULL CONSTRAINT DF_KullaniciIlaclari_ID DEFAULT NEWSEQUENTIALID(),
    HastaID          INT              NOT NULL,
    IlacID           INT              NOT NULL,
    ReceteKalemID    INT              NULL,
    Doz              NVARCHAR(50)     NOT NULL,
    Siklik           NVARCHAR(50)     NOT NULL,
    BaslangicTarihi  DATE             NOT NULL,
    BitisTarihi      DATE             NULL,
    KalanAdet        INT              NULL,
    GuncellemeZamani DATETIME2        NOT NULL CONSTRAINT DF_KullaniciIlaclari_Guncelleme DEFAULT SYSUTCDATETIME(),
    SilindiMi        BIT              NOT NULL CONSTRAINT DF_KullaniciIlaclari_SilindiMi DEFAULT 0,
    CONSTRAINT PK_KullaniciIlaclari PRIMARY KEY (KullaniciIlacID),
    CONSTRAINT CK_KullaniciIlaclari_Tarih CHECK (BitisTarihi IS NULL OR BitisTarihi >= BaslangicTarihi),
    CONSTRAINT CK_KullaniciIlaclari_KalanAdet CHECK (KalanAdet IS NULL OR KalanAdet >= 0),
    CONSTRAINT FK_KullaniciIlaclari_Hastalar        FOREIGN KEY (HastaID)       REFERENCES dbo.Hastalar (KullaniciID),
    CONSTRAINT FK_KullaniciIlaclari_Ilaclar         FOREIGN KEY (IlacID)        REFERENCES dbo.Ilaclar (IlacID),
    CONSTRAINT FK_KullaniciIlaclari_ReceteKalemleri FOREIGN KEY (ReceteKalemID) REFERENCES dbo.ReceteKalemleri (ReceteKalemID)
);
CREATE INDEX IX_KullaniciIlaclari_Hasta_Guncelleme ON dbo.KullaniciIlaclari (HastaID, GuncellemeZamani);
 
CREATE TABLE IlacSaatleri (
    IlacSaatID       UNIQUEIDENTIFIER NOT NULL CONSTRAINT DF_IlacSaatleri_ID DEFAULT NEWSEQUENTIALID(),
    KullaniciIlacID  UNIQUEIDENTIFIER NOT NULL,
    Saat             TIME(0)          NOT NULL,
    GuncellemeZamani DATETIME2        NOT NULL CONSTRAINT DF_IlacSaatleri_Guncelleme DEFAULT SYSUTCDATETIME(),
    SilindiMi        BIT              NOT NULL CONSTRAINT DF_IlacSaatleri_SilindiMi DEFAULT 0,
    CONSTRAINT PK_IlacSaatleri PRIMARY KEY (IlacSaatID),
    CONSTRAINT FK_IlacSaatleri_KullaniciIlaclari FOREIGN KEY (KullaniciIlacID) REFERENCES dbo.KullaniciIlaclari (KullaniciIlacID)
);
CREATE INDEX IX_IlacSaatleri_KullaniciIlacID ON dbo.IlacSaatleri (KullaniciIlacID, GuncellemeZamani);
 
CREATE TABLE DozDurumlari (
    DozDurumID  INT IDENTITY(1,1) NOT NULL,
    DozDurumAdi NVARCHAR(50)      NOT NULL,
    CONSTRAINT PK_DozDurumlari PRIMARY KEY (DozDurumID),
    CONSTRAINT UQ_DozDurumlari_DozDurumAdi UNIQUE (DozDurumAdi)
);
 
CREATE TABLE DozKayitlari (
    DozKayitID       UNIQUEIDENTIFIER NOT NULL CONSTRAINT DF_DozKayitlari_ID DEFAULT NEWSEQUENTIALID(),
    IlacSaatID       UNIQUEIDENTIFIER NOT NULL,
    PlanlananZaman   DATETIME2        NOT NULL,
    DozDurumID       INT              NOT NULL,
    OnayZamani       DATETIME2        NULL,
    GuncellemeZamani DATETIME2        NOT NULL CONSTRAINT DF_DozKayitlari_Guncelleme DEFAULT SYSUTCDATETIME(),
    SilindiMi        BIT              NOT NULL CONSTRAINT DF_DozKayitlari_SilindiMi DEFAULT 0,
    CONSTRAINT PK_DozKayitlari PRIMARY KEY (DozKayitID),
    CONSTRAINT FK_DozKayitlari_IlacSaatleri FOREIGN KEY (IlacSaatID) REFERENCES dbo.IlacSaatleri (IlacSaatID),
    CONSTRAINT FK_DozKayitlari_DozDurumlari FOREIGN KEY (DozDurumID) REFERENCES dbo.DozDurumlari (DozDurumID)
);
CREATE UNIQUE INDEX UX_DozKayitlari_Saat_Zaman ON dbo.DozKayitlari (IlacSaatID, PlanlananZaman) WHERE SilindiMi = 0;
CREATE INDEX IX_DozKayitlari_Guncelleme ON dbo.DozKayitlari (GuncellemeZamani);
 
CREATE TABLE EtkilesimUyarilari (
    UyariID         INT IDENTITY(1,1) NOT NULL,
    HastaID         INT               NOT NULL,
    KuralID         INT               NULL,
    BesinKuralID    INT               NULL,
    KullaniciIlacID UNIQUEIDENTIFIER  NOT NULL,
    RiskSkoru       DECIMAL(5,2)      NULL,
    Durum           NVARCHAR(30)      NOT NULL CONSTRAINT DF_EtkilesimUyarilari_Durum DEFAULT N'Yeni',
    OlusturmaTarihi DATETIME2         NOT NULL CONSTRAINT DF_EtkilesimUyarilari_OlusturmaTarihi DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_EtkilesimUyarilari PRIMARY KEY (UyariID),
    CONSTRAINT CK_EtkilesimUyarilari_XOR CHECK (
        (KuralID IS NOT NULL AND BesinKuralID IS NULL)
        OR (KuralID IS NULL AND BesinKuralID IS NOT NULL)
    ),
    CONSTRAINT CK_EtkilesimUyarilari_RiskSkoru CHECK (RiskSkoru IS NULL OR (RiskSkoru >= 0 AND RiskSkoru <= 100)),
    CONSTRAINT FK_EtkilesimUyarilari_Hastalar                 FOREIGN KEY (HastaID)         REFERENCES dbo.Hastalar (KullaniciID),
    CONSTRAINT FK_EtkilesimUyarilari_IlacEtkilesimKurallari   FOREIGN KEY (KuralID)         REFERENCES dbo.IlacEtkilesimKurallari (KuralID),
    CONSTRAINT FK_EtkilesimUyarilari_BesinEtkilesimKurallari  FOREIGN KEY (BesinKuralID)    REFERENCES dbo.BesinEtkilesimKurallari (BesinKuralID),
    CONSTRAINT FK_EtkilesimUyarilari_KullaniciIlaclari        FOREIGN KEY (KullaniciIlacID) REFERENCES dbo.KullaniciIlaclari (KullaniciIlacID)
);
 
/* ===== ECZANE / OTOMAT / STOK ===== */
 
CREATE TABLE Eczaneler (
    EczaneID     INT IDENTITY(1,1) NOT NULL,
    SahipID      INT               NOT NULL,
    EczaneAdi    NVARCHAR(200)     NOT NULL,
    EczaneEnlem  DECIMAL(9,6)      NOT NULL,
    EczaneBoylam DECIMAL(9,6)      NOT NULL,
    Telefon      VARCHAR(20)       NULL,
    CONSTRAINT PK_Eczaneler PRIMARY KEY (EczaneID),
    CONSTRAINT CK_Eczaneler_Koordinat CHECK (EczaneEnlem BETWEEN -90 AND 90 AND EczaneBoylam BETWEEN -180 AND 180),
    CONSTRAINT FK_Eczaneler_Eczacilar FOREIGN KEY (SahipID) REFERENCES dbo.Eczacilar (KullaniciID)
);
 
CREATE TABLE OtomatDurumlari (
    OtomatDurumID  INT IDENTITY(1,1) NOT NULL,
    OtomatDurumAdi NVARCHAR(50)      NOT NULL,
    CONSTRAINT PK_OtomatDurumlari PRIMARY KEY (OtomatDurumID),
    CONSTRAINT UQ_OtomatDurumlari_Adi UNIQUE (OtomatDurumAdi)
);
 
CREATE TABLE Otomatlar (
    OtomatID      INT IDENTITY(1,1) NOT NULL,
    EczaneID      INT               NOT NULL,
    OtomatDurumID INT               NOT NULL,
    OtomatEnlem   DECIMAL(9,6)      NOT NULL,
    OtomatBoylam  DECIMAL(9,6)      NOT NULL,
    CONSTRAINT PK_Otomatlar PRIMARY KEY (OtomatID),
    CONSTRAINT CK_Otomatlar_Koordinat CHECK (OtomatEnlem BETWEEN -90 AND 90 AND OtomatBoylam BETWEEN -180 AND 180),
    CONSTRAINT FK_Otomatlar_Eczaneler       FOREIGN KEY (EczaneID)      REFERENCES dbo.Eczaneler (EczaneID),
    CONSTRAINT FK_Otomatlar_OtomatDurumlari FOREIGN KEY (OtomatDurumID) REFERENCES dbo.OtomatDurumlari (OtomatDurumID)
);
 
CREATE TABLE EczaneStoklari (
    EczaneID     INT NOT NULL,
    PartiID      INT NOT NULL,
    Miktar       INT NOT NULL CONSTRAINT DF_EczaneStoklari_Miktar DEFAULT 0,
    KritikSeviye INT NOT NULL CONSTRAINT DF_EczaneStoklari_Kritik DEFAULT 0,
    CONSTRAINT PK_EczaneStoklari PRIMARY KEY (EczaneID, PartiID),
    CONSTRAINT CK_EczaneStoklari_Miktar CHECK (Miktar >= 0 AND KritikSeviye >= 0),
    CONSTRAINT FK_EczaneStoklari_Eczaneler FOREIGN KEY (EczaneID) REFERENCES dbo.Eczaneler (EczaneID),
    CONSTRAINT FK_EczaneStoklari_Partiler  FOREIGN KEY (PartiID)  REFERENCES dbo.Partiler (PartiID)
);
 
CREATE TABLE OtomatStoklari (
    OtomatID      INT NOT NULL,
    PartiID       INT NOT NULL,
    Miktar        INT NOT NULL CONSTRAINT DF_OtomatStoklari_Miktar DEFAULT 0,
    RezerveMiktar INT NOT NULL CONSTRAINT DF_OtomatStoklari_Rezerve DEFAULT 0,
    KritikSeviye  INT NOT NULL CONSTRAINT DF_OtomatStoklari_Kritik DEFAULT 0,
    CONSTRAINT PK_OtomatStoklari PRIMARY KEY (OtomatID, PartiID),
    CONSTRAINT CK_OtomatStoklari_Miktar CHECK (Miktar >= 0 AND RezerveMiktar >= 0 AND KritikSeviye >= 0 AND RezerveMiktar <= Miktar),
    CONSTRAINT FK_OtomatStoklari_Otomatlar FOREIGN KEY (OtomatID) REFERENCES dbo.Otomatlar (OtomatID),
    CONSTRAINT FK_OtomatStoklari_Partiler  FOREIGN KEY (PartiID)  REFERENCES dbo.Partiler (PartiID)
);
 
CREATE TABLE SicaklikKayitlari (
    SicaklikID BIGINT IDENTITY(1,1) NOT NULL,
    OtomatID   INT                  NOT NULL,
    Sicaklik   DECIMAL(5,2)         NOT NULL,
    KayitZamani DATETIME2           NOT NULL CONSTRAINT DF_SicaklikKayitlari_KayitZamani DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_SicaklikKayitlari PRIMARY KEY (SicaklikID),
    CONSTRAINT FK_SicaklikKayitlari_Otomatlar FOREIGN KEY (OtomatID) REFERENCES dbo.Otomatlar (OtomatID)
);
CREATE INDEX IX_SicaklikKayitlari_Otomat_Zaman ON dbo.SicaklikKayitlari (OtomatID, KayitZamani);
 
/* ===== ILAC CIKISI (SATIS / TESLIM) ===== */
 
CREATE TABLE OdemeDurumlari (
    OdemeDurumID INT IDENTITY(1,1) NOT NULL,
    DurumAdi     NVARCHAR(50)      NOT NULL,
    CONSTRAINT PK_OdemeDurumlari PRIMARY KEY (OdemeDurumID),
    CONSTRAINT UQ_OdemeDurumlari_DurumAdi UNIQUE (DurumAdi)
);
 
CREATE TABLE IlacCikisDurumlari (
    IlacCikisDurumID INT IDENTITY(1,1) NOT NULL,
    DurumKodu        VARCHAR(30)       NOT NULL,
    IslemDurumAdi    NVARCHAR(100)     NOT NULL,
    CONSTRAINT PK_IlacCikisDurumlari PRIMARY KEY (IlacCikisDurumID),
    CONSTRAINT UQ_IlacCikisDurumlari_DurumKodu UNIQUE (DurumKodu)
);
 
CREATE TABLE IlacCikislari (
    IlacCikisID       INT IDENTITY(1,1) NOT NULL,
    HastaID           INT               NULL,
    OtomatID          INT               NULL,
    EczaneID          INT               NULL,
    ReceteID          INT               NULL,
    IlacCikisDurumID  INT               NOT NULL,
    Tutar             DECIMAL(10,2)     NOT NULL CONSTRAINT DF_IlacCikislari_Tutar DEFAULT 0,
    OdemeDurumID      INT               NOT NULL,
    OlusturulmaTarihi DATETIME2         NOT NULL CONSTRAINT DF_IlacCikislari_OlusturulmaTarihi DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_IlacCikislari PRIMARY KEY (IlacCikisID),
    CONSTRAINT CK_IlacCikislari_Tutar CHECK (Tutar >= 0),
    CONSTRAINT CK_IlacCikislari_XOR_OtomatEczane CHECK (
        (OtomatID IS NOT NULL AND EczaneID IS NULL)
        OR (OtomatID IS NULL AND EczaneID IS NOT NULL)
    ),
    CONSTRAINT FK_IlacCikislari_Hastalar            FOREIGN KEY (HastaID)          REFERENCES dbo.Hastalar (KullaniciID),
    CONSTRAINT FK_IlacCikislari_Otomatlar           FOREIGN KEY (OtomatID)         REFERENCES dbo.Otomatlar (OtomatID),
    CONSTRAINT FK_IlacCikislari_Eczaneler           FOREIGN KEY (EczaneID)         REFERENCES dbo.Eczaneler (EczaneID),
    CONSTRAINT FK_IlacCikislari_Receteler           FOREIGN KEY (ReceteID)         REFERENCES dbo.Receteler (ReceteID),
    CONSTRAINT FK_IlacCikislari_IlacCikisDurumlari  FOREIGN KEY (IlacCikisDurumID) REFERENCES dbo.IlacCikisDurumlari (IlacCikisDurumID),
    CONSTRAINT FK_IlacCikislari_OdemeDurumlari      FOREIGN KEY (OdemeDurumID)     REFERENCES dbo.OdemeDurumlari (OdemeDurumID)
);
 
CREATE TABLE IlacCikisKalemleri (
    IlacCikisKalemID INT IDENTITY(1,1) NOT NULL,
    IlacCikisID      INT               NOT NULL,
    PartiID          INT               NOT NULL,
    Adet             INT               NOT NULL,
    BirimFiyat       DECIMAL(10,2)     NOT NULL,
    CONSTRAINT PK_IlacCikisKalemleri PRIMARY KEY (IlacCikisKalemID),
    CONSTRAINT CK_IlacCikisKalemleri CHECK (Adet > 0 AND BirimFiyat >= 0),
    CONSTRAINT FK_IlacCikisKalemleri_IlacCikislari FOREIGN KEY (IlacCikisID) REFERENCES dbo.IlacCikislari (IlacCikisID),
    CONSTRAINT FK_IlacCikisKalemleri_Partiler      FOREIGN KEY (PartiID)     REFERENCES dbo.Partiler (PartiID)
);
 
CREATE TABLE StokHareketleri (
    StokHareketleriID BIGINT IDENTITY(1,1) NOT NULL,
    IlacCikisID       INT                  NULL,
    OtomatID          INT                  NULL,
    EczaneID          INT                  NULL,
    PartiID           INT                  NOT NULL,
    HareketTuru       NVARCHAR(30)         NOT NULL,
    Miktar            INT                  NOT NULL,
    Tarih             DATETIME2            NOT NULL CONSTRAINT DF_StokHareketleri_Tarih DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_StokHareketleri PRIMARY KEY (StokHareketleriID),
    CONSTRAINT CK_StokHareketleri_Miktar CHECK (Miktar <> 0),
    CONSTRAINT CK_StokHareketleri_XOR_OtomatEczane CHECK (
        (OtomatID IS NOT NULL AND EczaneID IS NULL)
        OR (OtomatID IS NULL AND EczaneID IS NOT NULL)
    ),
    CONSTRAINT FK_StokHareketleri_IlacCikislari FOREIGN KEY (IlacCikisID) REFERENCES dbo.IlacCikislari (IlacCikisID),
    CONSTRAINT FK_StokHareketleri_Otomatlar     FOREIGN KEY (OtomatID)    REFERENCES dbo.Otomatlar (OtomatID),
    CONSTRAINT FK_StokHareketleri_Eczaneler     FOREIGN KEY (EczaneID)    REFERENCES dbo.Eczaneler (EczaneID),
    CONSTRAINT FK_StokHareketleri_Partiler      FOREIGN KEY (PartiID)     REFERENCES dbo.Partiler (PartiID)
);
 
/* ===== OYUNLASTIRMA ===== */
 
CREATE TABLE Rozetler (
    RozetID     INT IDENTITY(1,1) NOT NULL,
    RozetAdi    NVARCHAR(100)     NOT NULL,
    GerekliSeri INT               NOT NULL,
    CONSTRAINT PK_Rozetler PRIMARY KEY (RozetID),
    CONSTRAINT UQ_Rozetler_RozetAdi UNIQUE (RozetAdi),
    CONSTRAINT CK_Rozetler_GerekliSeri CHECK (GerekliSeri > 0)
);
 
CREATE TABLE KullaniciRozetleri (
    HastaID       INT       NOT NULL,
    RozetID       INT       NOT NULL,
    KazanmaTarihi DATETIME2 NOT NULL CONSTRAINT DF_KullaniciRozetleri_KazanmaTarihi DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_KullaniciRozetleri PRIMARY KEY (HastaID, RozetID),
    CONSTRAINT FK_KullaniciRozetleri_Hastalar FOREIGN KEY (HastaID) REFERENCES dbo.Hastalar (KullaniciID),
    CONSTRAINT FK_KullaniciRozetleri_Rozetler FOREIGN KEY (RozetID) REFERENCES dbo.Rozetler (RozetID)
);
 
CREATE TABLE PuanHareketleri (
    PuanHareketleriID INT IDENTITY(1,1) NOT NULL,
    HastaID           INT               NOT NULL,
    DozKayitID        UNIQUEIDENTIFIER  NULL,
    PuanMiktari       INT               NOT NULL,
    HareketTuru       NVARCHAR(50)      NOT NULL,
    Aciklama          NVARCHAR(250)     NULL,
    Tarih             DATETIME2         NOT NULL CONSTRAINT DF_PuanHareketleri_Tarih DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_PuanHareketleri PRIMARY KEY (PuanHareketleriID),
    CONSTRAINT FK_PuanHareketleri_Hastalar    FOREIGN KEY (HastaID)    REFERENCES dbo.Hastalar (KullaniciID),
    CONSTRAINT FK_PuanHareketleri_DozKayitlari FOREIGN KEY (DozKayitID) REFERENCES dbo.DozKayitlari (DozKayitID)
);
CREATE INDEX IX_PuanHareketleri_Hasta_Tarih ON dbo.PuanHareketleri (HastaID, Tarih);
 
/* ===== GUVENLIK / KVKK / BILDIRIM ===== */
 
CREATE TABLE DenetimKayitlari (
    KayitID     BIGINT IDENTITY(1,1) NOT NULL,
    KullaniciID INT                  NULL,
    Islem       NVARCHAR(100)        NOT NULL,
    VarlikTuru  NVARCHAR(100)        NOT NULL,
    VarlikID    NVARCHAR(64)         NULL,
    Zaman       DATETIME2            NOT NULL CONSTRAINT DF_DenetimKayitlari_Zaman DEFAULT SYSUTCDATETIME(),
    OncekiHash  CHAR(64)             NULL,
    KayitHash   CHAR(64)             NOT NULL,
    IPAdresi    VARCHAR(45)          NULL,
    CONSTRAINT PK_DenetimKayitlari PRIMARY KEY (KayitID),
    CONSTRAINT UQ_DenetimKayitlari_KayitHash UNIQUE (KayitHash),
    CONSTRAINT FK_DenetimKayitlari_Kullanicilar FOREIGN KEY (KullaniciID) REFERENCES dbo.Kullanicilar (KullaniciID)
);
CREATE INDEX IX_DenetimKayitlari_Kullanici_Zaman ON dbo.DenetimKayitlari (KullaniciID, Zaman);
 
CREATE TABLE YenilemeTokenlari (
    TokenID       INT IDENTITY(1,1) NOT NULL,
    KullaniciID   INT               NOT NULL,
    TokenHash     CHAR(64)          NOT NULL,
    BitisZamani   DATETIME2         NOT NULL,
    IptalEdildiMi BIT               NOT NULL CONSTRAINT DF_YenilemeTokenlari_Iptal DEFAULT 0,
    CONSTRAINT PK_YenilemeTokenlari PRIMARY KEY (TokenID),
    CONSTRAINT UQ_YenilemeTokenlari_TokenHash UNIQUE (TokenHash),
    CONSTRAINT FK_YenilemeTokenlari_Kullanicilar FOREIGN KEY (KullaniciID) REFERENCES dbo.Kullanicilar (KullaniciID)
);
 
CREATE TABLE RizaTurleri (
    RizaTuruID INT IDENTITY(1,1) NOT NULL,
    RizaAdi    NVARCHAR(150)     NOT NULL,
    CONSTRAINT PK_RizaTurleri PRIMARY KEY (RizaTuruID),
    CONSTRAINT UQ_RizaTurleri_RizaAdi UNIQUE (RizaAdi)
);
 
CREATE TABLE RizaKayitlari (
    RizaID      INT IDENTITY(1,1) NOT NULL,
    KullaniciID INT               NOT NULL,
    RizaTuruID  INT               NOT NULL,
    VerildiMi   BIT               NOT NULL,
    Zaman       DATETIME2         NOT NULL CONSTRAINT DF_RizaKayitlari_Zaman DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_RizaKayitlari PRIMARY KEY (RizaID),
    CONSTRAINT FK_RizaKayitlari_Kullanicilar FOREIGN KEY (KullaniciID) REFERENCES dbo.Kullanicilar (KullaniciID),
    CONSTRAINT FK_RizaKayitlari_RizaTurleri  FOREIGN KEY (RizaTuruID)  REFERENCES dbo.RizaTurleri (RizaTuruID)
);
CREATE INDEX IX_RizaKayitlari_Kullanici_Tur ON dbo.RizaKayitlari (KullaniciID, RizaTuruID, Zaman);
 
CREATE TABLE CihazTokenlari (
    CihazTokenID  INT IDENTITY(1,1) NOT NULL,
    KullaniciID   INT               NOT NULL,
    CihazToken    NVARCHAR(512)     NOT NULL,
    CihazTuru     NVARCHAR(20)      NOT NULL,
    SonAktifTarih DATETIME2         NOT NULL CONSTRAINT DF_CihazTokenlari_SonAktif DEFAULT SYSUTCDATETIME(),
    CONSTRAINT PK_CihazTokenlari PRIMARY KEY (CihazTokenID),
    CONSTRAINT FK_CihazTokenlari_Kullanicilar FOREIGN KEY (KullaniciID) REFERENCES dbo.Kullanicilar (KullaniciID)
);
CREATE INDEX IX_CihazTokenlari_KullaniciID ON dbo.CihazTokenlari (KullaniciID);
 
CREATE TABLE BildirimLoglari (
    BildirimID        BIGINT IDENTITY(1,1) NOT NULL,
    KullaniciID       INT                  NOT NULL,
    BildirimTuru      NVARCHAR(50)         NOT NULL,
    Kanal             NVARCHAR(20)         NOT NULL,
    Baslik            NVARCHAR(200)        NOT NULL,
    Icerik            NVARCHAR(1000)       NOT NULL, -- jenerik metin veya AES-256 sifreli
    GonderilmeZamani  DATETIME2            NOT NULL CONSTRAINT DF_BildirimLoglari_Gonderilme DEFAULT SYSUTCDATETIME(),
    GonderimDurumu    NVARCHAR(20)         NOT NULL,
    HataMesaji        NVARCHAR(500)        NULL,
    OkunduMu          BIT                  NOT NULL CONSTRAINT DF_BildirimLoglari_OkunduMu DEFAULT 0,
    TiklanmaZamani    DATETIME2            NULL,
    CONSTRAINT PK_BildirimLoglari PRIMARY KEY (BildirimID),
    CONSTRAINT FK_BildirimLoglari_Kullanicilar FOREIGN KEY (KullaniciID) REFERENCES dbo.Kullanicilar (KullaniciID)
);
CREATE INDEX IX_BildirimLoglari_Kullanici_Zaman ON dbo.BildirimLoglari (KullaniciID, GonderilmeZamani);
GO


