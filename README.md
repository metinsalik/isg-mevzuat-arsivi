
# Sağlık Tesisleri İSG & Yangın Mevzuatı Bilgi Bankası ve Canlı Saha Asistanı

Sağlık tesisleri ve hastane işletmeleri için özel olarak yapılandırılmış, versiyon kontrollü **Türk İş Sağlığı ve Güvenliği (İSG) ve Yangın Güvenliği Mevzuat Arşivi** ile bu arşivi canlı web uygulamalarına bağlayan **Hibrit RAG & Saha Tecrübesi Asistanı** mimarisidir.

---

## 1. Sistem Mimarisi (Temiz Sunucu Kurulumu)

Sistem, sıfırdan kurulacak bağımsız bir Ubuntu sunucu üzerinde **Docker Compose** ile orkestre edilecek 3 katmanlı bir yapıda tasarlanmıştır:

    +-------------------------------------------------------------------+
    |                  CANLI WEB UYGULAMASI (Vite / React)              |
    |  - Aktif Şube / Hastane Birimi Bağlamı (Ameliyathane, Radyoloji)  |
    |  - Tespit & Öneri Defteri / İSG Kurulu / İç Denetim Ekranları     |
    +---------------------------------+---------------------------------+
                                      |  HTTPS / REST API (JWT & Scopes)
                                      v
    +-------------------------------------------------------------------+
    |               İSG AI API SERVİSİ (Python FastAPI)                 |
    |  - Hibrit Arama Motoru (Vektör + YAML Metadata/Etiket Filtresi)   |
    |  - Human-in-the-Loop (Onaylı JSON Taslak Üretimi)                 |
    |  - Canlı İzleme & Saha Tecrübesi Öğrenme Hattı                    |
    +-----------------+-------------------------------+-----------------+
                      |                               |
                      v                               v
    +---------------------------------+ +-------------------------------+
    |   POSTGRESQL 16 + PGVECTOR      | |   9ROUTER / OLLAMA (LLM)      |
    |  - mevzuat_chunks (Statik RAG)  | |  - Embedding (bge-m3)         |
    |  - saha_tecrubeleri (Dinamik)   | |  - LLM Yönlendirme (9router)  |
    +---------------------------------+ +-------------------------------+
                      ^
                      |  GitHub Webhook / Manuel Sync
    +-----------------+-------------------------------------------------+
    |             GITHUB MEVZUAT REPOSU (isg-mevzuat-arsivi)            |
    +-------------------------------------------------------------------+

### Temel Bileşenler
1. **Mevzuat Bilgi Bankası (Statik/Yarı-Dinamik Çekirdek):** GitHub üzerindeki YAML frontmatter etiketli Markdown dosyaları parçalanarak (`chunking`) `pgvector` üzerindeki `mevzuat_chunks` tablosuna indekslenir.
2. **Canlı Uygulama İzleme & Saha Hafızası (Dinamik Katman):** Web uygulamasında çözülen uygunsuzluklar, denetim bulguları ve uzman onayından geçen aksiyonlar anonimleştirilerek `saha_tecrubeleri` vektör tablosuna aktarılır. Bot, yanıtlarında **Mevzuat Hükmü + Kurumsal Saha Tecrübesi** sentezini sunar.
3. **Bağımsız İSG AI API Servisi (FastAPI):** Web uygulamasına sadece okuma (`read:context`) ve yapılandırılmış taslak önerme (`suggest:draft`) izinleriyle bağlanır; doğrudan veritabanına yazma yapmaz (Human-in-the-Loop).

---

## 2. Mevcut Arşiv Kapsamı (Faz 1 Çekirdek Seti — 10 Mevzuat)

Arşivde yer alan tüm dosyalar; `mevzuat_kodu`, `madde_no`, `hastane_birimleri`, `iliskili_maddeler` ve `saha_etiketleri` içeren standart YAML üst verileriyle (frontmatter) yapılandırılmıştır.

| Kategori | Mevzuat Kodu | Mevzuat Adı | Resmî Gazete | Klasör Yolu |
| :--- | :--- | :--- | :--- | :--- |
| **Kanun** | `6331_ISG_KANUNU` | 6331 Sayılı İş Sağlığı ve Güvenliği Kanunu | 30.06.2012 - 28339 | `01-kanunlar/6331-isg-kanunu/` |
| **Kanun** | `4857_IS_KANUNU` | 4857 Sayılı İş Kanunu (İSG ve Çalışma Düzeni Hükümleri) | 10.06.2003 - 25134 | `01-kanunlar/4857-is-kanunu-ilgili/` |
| **Temel Yön.** | `ALT_ISVERENLIK_YON` | Alt İşverenlik Yönetmeliği | 27.09.2008 - 27010 | `02-temel-yonetmelikler/alt-isverenlik/` |
| **Temel Yön.** | `ANALIK_KISMI_CALISMA_YON` | Analık İzni veya Ücretsiz İzin Sonrası Yapılacak Kısmi Süreli Çalışmalar Hakkında Yönetmelik | 08.11.2016 - 29882 | `02-temel-yonetmelikler/analik-kismi-sureli-calisma/` |
| **Temel Yön.** | `ASANSOR_BAKIM_YON` | Asansör İşletme ve Bakım Yönetmeliği (+ EK 1–5) | 06.04.2019 - 30737 | `02-temel-yonetmelikler/asansor-isletme-ve-bakim/` |
| **Temel Yön.** | `ASANSOR_PERIYODIK_YON` | Asansör Periyodik Kontrol Yönetmeliği | 04.05.2018 - 30411 | `02-temel-yonetmelikler/asansor-periyodik-kontrol/` |
| **Temel Yön.** | `BASIT_BASINCLI_KAPLAR_YON` | Basit Basınçlı Kaplar Yönetmeliği (2014/29/AB) (+ EK I–IV) | 03.11.2016 - 29877 | `02-temel-yonetmelikler/basit-basincli-kaplar/` |
| **Temel Yön.** | `BASINCLI_EKIPMANLAR_YON` | Basınçlı Ekipmanlar Yönetmeliği (2014/68/AB) (+ EK I–IV) | 03.03.2018 - 30349 | `02-temel-yonetmelikler/basincli-ekipmanlar/` |
| **Temel Yön.** | `ISG_KURULLARI_YON` | İş Sağlığı ve Güvenliği Kurulları Hakkında Yönetmelik | 18.01.2013 - 28532 | `02-temel-yonetmelikler/isg-kurullari/` |
| **Sağlık Özel** | `YANGIN_YON` | Binaların Yangından Korunması Hakkında Yönetmelik (+ EK 1–14) | 19.12.2007 - 26735 | `03-saglik-ozel-yonetmelikler/binalarin-yangindan-korunmasi/` |

---

## 3. Dizin Yapısı

    isg-mevzuat-arsivi/
    ├── manifest.json                         # Aktif mevzuat kataloğu ve öncelik skorları
    ├── README.md                             # Mimari dokümantasyon ve yol haritası
    ├── 01-kanunlar/
    │   ├── 6331-isg-kanunu/
    │   └── 4857-is-kanunu-ilgili/
    ├── 02-temel-yonetmelikler/
    │   ├── alt-isverenlik/
    │   ├── analik-kismi-sureli-calisma/
    │   ├── asansor-isletme-ve-bakim/
    │   ├── asansor-periyodik-kontrol/
    │   ├── basit-basincli-kaplar/
    │   ├── basincli-ekipmanlar/
    │   └── isg-kurullari/
    └── 03-saglik-ozel-yonetmelikler/
        └── binalarin-yangindan-korunmasi/

---

## 4. Uygulama Yol Haritası (Temiz Sunucu & Canlı Entegrasyon)

### Faz 1: Temiz Sunucu Altyapısı ve Veritabanı Kurulumu
* **Hedef:** Yeni Ubuntu sunucuda Docker Compose ile `postgres:16` (`pgvector` eklentili), `ollama` (veya `9router` bağlantısı) ve `isg-ai-api` konteynerlerinin ayağa kaldırılması.
* **Veritabanı Tabloları:**
  * `mevzuat_chunks`: Mevzuat metinleri, vektör gömmeleri (`embedding vector(1024)`), YAML metadata (`JSONB`) ve tam metin arama (`tsvector`) indeksleri.
  * `saha_tecrubeleri`: Canlı web uygulamasından aktarılan onaylı saha çözümleri, birim etiketleri ve denetim notları.
  * `chat_logs` & `feedback`: Kullanıcı soruları, atıf yapılan maddeler ve uzman geri bildirimleri.
* **İndeksleme (`ingest_mevzuat.py`):** GitHub reposunu tarayarak `manifest.json` içindeki tüm Markdown dosyalarını madde bazlı parçalayıp veritabanına yazar.

### Faz 2: Çekirdek RAG ve FastAPI Servisi
* **Hibrit Arama (Hybrid Search):** Kullanıcının sorusundaki hastane birimi (`Ameliyathane`, `Radyoloji`, `Teknik Servis` vb.) ve anahtar kelimeler ile vektör benzerliğini (`cosine similarity` + `BM25`) birleştirir.
* **Katı Atıf ve Halüsinasyon Koruması:** Model yalnızca getirilen mevzuat maddelerine ve onaylı saha tecrübelerine dayanarak yanıt üretir; her yanıtta `Mevzuat Adı + Madde No` kaynak gösterir.
* **API Uç Noktaları:**
  * `POST /api/v1/ask`: Bağlam duyarlı (şube, birim, aktif modül) soru-cevap ve streaming yanıt.
  * `POST /api/v1/draft-finding`: Tespit-Öneri Defteri veya Kurul gündemi için yapılandırılmış JSON taslağı üretimi.
  * `POST /api/v1/sync/github`: Yeni mevzuat eklendiğinde vektör indeksini güncelleyen webhook uç noktası.

### Faz 3: Web Uygulaması Entegrasyonu ve Yetkilendirme
* **Bağlam Aktarımı (Context Injection):** Web uygulaması, asistanı açan kullanıcının rolünü, aktif hastane şubesini, bulunduğu birimi ve açık olan form/kayıt özetini API'ye iletir.
* **Yetki Sınırları:**
  * `read:unit_context`: Sadece kullanıcının yetkili olduğu şube/birim verilerini okuma izni.
  * `suggest:action`: Doğrudan kayıt açma yetkisi olmadan, kullanıcının tek tıkla formuna aktarabileceği öneri taslağı oluşturma izni.

### Faz 4: Canlı İzleme ve Sürekli Öğrenme Döngüsü (Feedback Loop)
* **Olay Dinleme (Event / Webhook / Read-Only View):** Web uygulamasında durumu **"Kapatıldı / Çözüldü"** olarak işaretlenen Tespit-Öneri Defteri kayıtları, Kurul kararları ve İç Denetim aksiyonları periyodik olarak taranır.
* **Anonimleştirme ve Vektörleştirme:** Kişisel verilerden (KVKK) arındırılan vaka özetleri (`Sorun -> Mevzuat Dayanağı -> Uygulanan Çözüm`) `saha_tecrubeleri` tablosuna eklenir.
* **Büyüyen Mevzuat Havuzu:** İlerleyen aşamalarda *Tıbbi Atıkların Kontrolü Yönetmeliği*, *Radyasyon Güvenliği Yönetmeliği*, *Hasta ve Çalışan Güvenliğinin Sağlanmasına Dair Yönetmelik* ve *İş Ekipmanlarının Kullanımında Sağlık ve Güvenlik Şartları Yönetmeliği* aynı YAML şablonuyla arşive eklenerek sisteme dahil edilir.
EOF
git add README.md && git commit -m "docs: temiz sunucu kurulumu ve canli saha entegrasyonu yol haritasi eklendi" && git push origin main
