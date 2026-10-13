# Gemini — Baştan Sona Yeniden Tasarım Promptu

> **Nasıl kullanılır:**
> 1. Gemini'de mevcut projeyi aç.
> 2. Bu dosyayı ve `tasarim/` klasörünün tamamını yükle.
> 3. `tasarim/ekran/` içindeki PNG görüntüler en önemli kısım. Zip açılmazsa bunları tek tek yükle.
> 4. Aşağıdaki "PROMPT" bölümünü yapıştır.

---

## PROMPT

Bu projenin arayüz tasarımı tamamen kabul edilmedi. Siyah-gold renkler, süslü kartlar ve hazır şablon görünümü yüzünden yapmacık duruyor.

Görevin, **iş mantığına, API'lere ve veritabanındaki randevu tablolarına dokunmadan, arayüzü baştan sona yeniden yapmak**. Ayrıca müşterilerin gördüğü ana sayfayı, işletmenin kendisinin özelleştirebileceği bir **şablon sistemine** dönüştüreceksin.

Hedef görünüm: **apple.com/tr**. Bol beyaz ve açık gri alan, büyük ve net başlıklar, büyük fotoğraflar, tek bir marka rengi, sakin ve kurumsal bir his.

Ekteki dosyalar nasıl görünmesi gerektiğini gösteriyor. Kendi stil fikrini ekleme. Referansla senin fikrin çelişirse referans geçerli.

### 1. Ekteki referans dosyalar

| Dosya | Ne gösteriyor |
|---|---|
| `tasarim/style.css` | Tüm uygulamanın temel stilleri ve renk değişkenleri. Aynen kullan. |
| `tasarim/vitrin.css` | Müşteri ana sayfası ve 3 şablonun stilleri. Aynen kullan. |
| `20-vitrin-klasik.html` + `ekran/20-*.png` | Şablon 1: **Klasik**. Apple ürün sayfası gibi ortalanmış başlık + büyük fotoğraf, ardından açık/koyu bölümler |
| `21-vitrin-galeri.html` + `ekran/21-*.png` | Şablon 2: **Galeri**. Tam ekran fotoğrafla açılır, fotoğraf ağırlıklı |
| `22-vitrin-sade.html` + `ekran/22-*.png` | Şablon 3: **Sade**. Yazı ağırlıklı, solda metin, sağda tek dikey fotoğraf |
| `30-panel-vitrin-duzenle.html` + PNG | İşletmenin siteyi düzenlediği ekran: şablon seçimi, marka rengi, bölümler, fotoğraflar, canlı önizleme |
| `02`, `03`, `04` | Müşteri randevu adımları: hizmet → tarih/saat → bilgiler |
| `10`, `11`, `12` | Personel girişi, günlük takvim, randevu detayı (Ara / SMS / Taşı / İptal) |

Referanslardaki gri kutular **fotoğraf yerleridir**. Kutunun içindeki yazı, oraya nasıl bir fotoğraf geleceğini anlatır. Gerçek uygulamada bu kutulara işletmenin yüklediği fotoğraflar gelecek. İşletme fotoğraf yüklemediyse şablonun varsayılan fotoğrafları gelecek.

### 2. Genel tasarım kuralları

**Renkler:**
- Yalnızca `style.css` içindeki değişkenler kullanılacak. Zemin beyaz `#FFFFFF` ve açık gri `#F5F5F7`, metin `#1D1D1F` ve `#6E6E73`.
- Tek vurgu rengi olarak işletmenin seçtiği marka rengi (`--brand`) kullanılacak.
- Koyu bölüm (`tile--dark`) yalnızca sayfada vurgu için tek bir şerit olarak kullanılabilir. Sayfanın tamamı koyu olmaz.

**Fontlar:**
- Yalnızca sistem fontu: `-apple-system, "SF Pro Text", "Segoe UI", Roboto...`
- Google Fonts yükleme. Serif ya da "vintage" font kullanma.

**Kesinlikle kaldır:**
- Siyah zemin + altın / gold / bronz / bakır renkler
- Mor, pembe, neon renkler ve renk geçişleri. Tek istisna: Galeri şablonunda fotoğrafın üstündeki yazının okunması için fotoğrafın altına konan siyahtan şeffafa geçiş.
- Buzlu cam (`backdrop-blur`), parlama efektleri, büyük gölgeler
- Emoji
- Makas, ustura ve bıyık ikonları
- "Premium", "Tarzını yansıt" gibi sloganlar, ünlem işaretleri

**Ölçüler:**

| Öğe | Masaüstü | Mobil |
|---|---|---|
| Büyük başlık | 56px / 600 | 36px |
| Alt başlık | 28px | 21px |
| Bölüm başlığı | 40px | 28px |
| Gövde metni | 17px | 17px |

- Butonlar hap şeklinde (`border-radius: 980px`).
- Kartların köşeleri 18px, gölgeleri çok hafif.
- Liste satırları iOS Ayarlar tarzında.

**Hareket:**
- Yalnızca opaklık ve kayma, 200ms.
- Sayfa açılışında uçuşan, zıplayan öğe olmayacak.

### 3. Müşteri ana sayfası = şablon sistemi

Her salonun müşteriye açık sayfası (`/s/{slug}`) bir **vitrin sitesi** olacak.

**3.1 Şablonlar**

Üç hazır şablon olacak. Yapıları referans HTML'lerle birebir aynı olmalı:
- `klasik`
- `galeri`
- `sade`

Şablon değişince içerik aynı kalır, yalnızca düzen değişir.

**3.2 Bölümler**

İşletme hangi bölümlerin görüneceğini açıp kapatabilir ve bölümlerin sırasını sürükleyerek değiştirebilir:

| Bölüm | İçerik |
|---|---|
| `acilis` | Salon adı, üst başlık, alt yazı, "Randevu Al" butonu, açılış fotoğrafı. Alt yazı varsayılan olarak otomatik gelir: "Bugün 14:30'da yeriniz hazır." (en yakın boş saat, canlı) |
| `duyuru` | Başlık, kısa metin, fotoğraf, bağlantı (kampanya, yeni hizmet vb.) |
| `hizmetler` | Hizmetler tablosundan otomatik. Her hizmetin isteğe bağlı fotoğrafı olur. Kartta "Randevu al ›" bağlantısı doğrudan o hizmet seçili şekilde randevu akışına gider. |
| `ekip` | Personel tablosundan otomatik. Portre fotoğrafı, ünvan, "Randevu al ›" bağlantısı (o personel seçili gelir) |
| `galeri` | 3–9 fotoğraf |
| `saatler_konum` | Çalışma saatleri tablosundan otomatik; bugünün satırı kalın. Adres, harita görüntüsü, telefon, yol tarifi |
| `sss` | Soru–cevap listesi (isteğe bağlı) |

**3.3 Marka rengi**

İşletme yalnızca şu 6 kurumsal renkten birini seçebilir. Serbest renk seçici olmayacak:
- Mavi `#0071E3`
- Lacivert `#1E3A5F`
- Orman `#2F5D50`
- Bordo `#7A2E39`
- Grafit `#3A3A3C`
- Kahve `#6B4E3D`

Seçilen renk `--brand` değişkenine yazılır. Yalnızca butonlarda ve bağlantılarda görünür.

**3.4 Veri modeli**

Yeni tablo `salon_sites`:
```
salon_id (PK, FK)
template        'klasik' | 'galeri' | 'sade'
brand_color     yukarıdaki 6 değerden biri (enum)
logo_url
content         JSONB  → { acilis: {...}, duyuru: {...}, sss: [...], sections: [{ type, enabled }] }  — sıra = dizi sırası
draft_content   JSONB  → düzenleyicide yapılan, henüz yayınlanmamış değişiklikler
published_at
```
Yeni tablo `salon_images`:
```
id, salon_id, slot ('acilis'|'duyuru'|'galeri'|'hizmet:{id}'|'personel:{id}'),
sira, original_key, variants JSONB (640/1280/1920 genişlik, avif+webp), blur_data_url, alt_text, created_at
```
JSON içeriğini Zod şemasıyla doğrula. Bilinmeyen bölüm tipi kabul edilmez.

**3.5 Fotoğraf yükleme**

- Yükleme sunucuda `sharp` ile işlenir:
  - EXIF bilgisi temizlenir ve fotoğraf doğru yöne çevrilir.
  - 640, 1280 ve 1920 px genişliklerde AVIF ve WebP sürümleri üretilir.
  - Küçük, bulanık bir ön izleme (blur placeholder) üretilir.
- Dosyalar Docker'da bir volume'a ya da S3 uyumlu depoya (compose'a **MinIO** servisi ekle) kaydedilir.
- Sınırlar: 10 MB'a kadar, yalnızca JPEG, PNG, HEIC ve WebP.
- Yüklerken kırpma ekranı açılır. Kırpma oranı slota göre sabittir:
  - Açılış: 16:7
  - Hizmet: 4:3
  - Personel: 3:4
  - Galeri: serbest
- Her şablonun **varsayılan fotoğraf seti** olacak. Ticari kullanıma izin veren stok fotoğraflar kullan (ör. Unsplash lisanslı salon içi, ayna, koltuk, ürün rafı, kesim detayı). Bunları `public/templates/{şablon}/` altına koy. İşletme kendi fotoğrafını yükleyince varsayılan fotoğrafın yerine geçer.
- Fotoğraflarda yüz ya da marka logosu öne çıkmasın. Aydınlık, doğal ışıklı, sade kareler olsun. Karanlık, dumanlı, "vintage" filtreli fotoğraf kullanma.

**3.6 Site düzenleyici (`/panel/site`, yalnızca yönetici)**

Referans: `30-panel-vitrin-duzenle.png`.
- **Sol sütun:** şablon kartları, marka rengi, bilgiler, bölümler (aç/kapa + sürükle-bırak), fotoğraflar.
- **Sağ taraf:** canlı önizleme. Üstte Masaüstü / Telefon geçişi var.
- **Kaydetme:** Değişiklikler otomatik olarak taslağa kaydedilir. Müşteriler "Yayınla"ya basılana kadar eski sürümü görür. "Son yayınlanan sürüme dön" seçeneği olacak.
- **Mobil:** Aynı ayarlar tek sütun hâlinde gösterilir. Önizleme ayrı bir sekmede açılır.

**3.7 Performans**

- Vitrin sayfası sunucuda üretilir (ISR). Yayınla'ya basınca yeniden oluşturulur (`revalidateTag`).
- Açılış fotoğrafı `priority` ile hemen yüklenir. Diğer fotoğraflar sayfa kaydırıldıkça yüklenir (`loading="lazy"`), hepsinde bulanık ön izleme olur.
- Hedefler: mobilde LCP 2 saniyenin altında, CLS 0.
- "En yakın boş saat" bilgisi sayfadan ayrı, küçük bir istekle gelir ve 60 saniye önbellekte tutulur. Böylece sayfanın kendisi uzun süre önbellekte kalabilir.

### 4. Diğer ekranlar

Randevu akışını ve personel/yönetici panelinin tamamını referans ekranlara (`02–04`, `10–12`) benzet:
- Gri zemin üstünde beyaz liste grupları
- Büyük sola hizalı başlık
- Altta sabit tek birincil buton
- Panelde alttan açılan sayfa (sheet) ve alt sekme çubuğu

Referansı olmayan ekranlar için de aynı parçaları kullan:
- müşteriler
- ayarlar
- raporlar
- bildirimler
- SMS gönderme

Randevu akışında da marka rengi (`--brand`) kullanılır. Böylece müşteri vitrin sitesinden randevu ekranına geçince aynı salonun içinde olduğunu hisseder.

### 5. Çalışma sırası

Her adımın sonunda değişen dosyaların **tam içeriğini** ver. Kısaltma yapma. Sonra "Devam edeyim mi?" diye sor.

1. Eski tema dosyalarını, renkleri ve fontları sil. `style.css` + `vitrin.css` ekle. Tailwind renk paletini `style.css` değişkenleriyle değiştir (varsayılan Tailwind renkleri kalmasın).
2. `salon_sites` + `salon_images` tabloları, migration, Zod şemaları, fotoğraf yükleme servisi (sharp + MinIO), varsayılan fotoğraf setleri.
3. Vitrin sitesi: bölüm bileşenleri ve 3 şablon.
4. Site düzenleyici: taslak/yayın, canlı önizleme.
5. Randevu akışının yeniden tasarımı.
6. Personel ve yönetici panelinin yeniden tasarımı.
7. Kontrol: aşağıdaki komutların çıktısı boş olmalı.
   ```
   grep -rniE "gold|#d4af37|#c9a227|#b8860b|amber|yellow|purple|violet|indigo|pink|backdrop-blur|shadow-(lg|xl|2xl)|playfair|cinzel|font-serif" app src components
   grep -rnP "[\x{1F300}-\x{1FAFF}\x{2728}]" app src components
   ```
   Ardından her şablonun masaüstü (1440px) ve telefon (390px) ekran görüntüsünü referans PNG'lerle yan yana karşılaştır. Farkları listele ve düzelt.
