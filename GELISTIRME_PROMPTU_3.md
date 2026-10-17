# Gemini — Tasarım Geliştirme Promptu (3. tur)

> **Nasıl kullanılır:** Gemini'deki mevcut projeye aşağıdaki "PROMPT" bölümünün tamamını yapıştır. Elindeki `tasarim/` klasörü ve ekran görüntüleri varsa onları da ekle. Bu prompt önceki tüm tasarım talimatlarının üstündedir. Çelişen bir kural varsa bu dosya geçerlidir.

---

## PROMPT

Tasarım hâlâ yeterince modern ve kurumsal değil. Sayfaların kenarlarında boş şeritler var, bölümler birbirine benzemiyor ve bazı yerlerde yazılar zor okunuyor.

Bu turda **iş mantığına, API'lere ve veritabanına dokunmadan** arayüzü baştan sona aşağıdaki sisteme göre yeniden düzenleyeceksin.

Hedef his: **apple.com, Stripe Dashboard ve iyi bir Türk bankasının internet şubesi.**
- Kurumsal, ferah, net.
- Ekranın tamamını kullanan, kendinden emin bir düzen.
- Hiçbir parça "yapay zekâyla üretilmiş şablon" gibi görünmeyecek.

Kod yazmaya başlamadan önce bu dokümanın tamamını oku. Sonra **0. adımdaki raporu** ver.

---

### 0. ÖNCE RAPOR (kod yazmadan)

Aşağıdaki iki tabloyu ver.

**1. Sayfa envanteri.** Her sayfa için:
- yol
- sayfa tipi (bkz. bölüm 6)
- içerik genişliği şu an nasıl (sabit mi, tam genişlik mi, kenarlarda boşluk var mı)
- ana buton nerede

**2. Renk denetimi.** Kodda geçen her yazı rengi + zemin rengi çifti ve kontrast oranı. 4.5:1'in altında kalanları işaretle.

Bu raporu verdikten sonra 9. bölümdeki sırayla kodlamaya geç.

---

### 1. DÜZEN: KENARDAN KENARA (FULL-BLEED)

Ana kural: **Ekranın sağında ve solunda boş, anlamsız gri şerit kalmayacak.**

Zeminler, fotoğraflar, tablolar, takvim ve başlık çubukları ekranın kenarına kadar uzanır. Yalnızca **okunacak metin satırları** okunabilir genişlikte tutulur. Onlar da zemin içinde hizalanır, sayfayı daraltmaz.

**1.1 Izgara**

Ortak bir `<Container>` bileşeni yaz. Bütün sayfalar bunu kullanacak.

| Ekran | Genişlik | Sütun | Sütun aralığı | Kenar boşluğu |
|---|---|---|---|---|
| Telefon | < 640px | 4 | 16px | 16px |
| Tablet | 640–1023px | 8 | 24px | 24px |
| Masaüstü | 1024–1535px | 12 | 24px | 32px |
| Geniş | ≥ 1536px | 12 | 32px | `clamp(32px, 4vw, 80px)` |

- Kenar boşluğu yalnızca içeriğin ekrana yapışmaması içindir. Zeminin rengi bu boşlukta da devam eder.
- Sabit `max-width: 980px` gibi dar kutular **kaldırılacak**.
- Tek istisna uzun metin paragraflarıdır (KVKK metni, açıklamalar): satır uzunluğu en fazla `72ch`, sola hizalı. Sayfa daralmaz.

**1.2 Panel (personel ve yönetici)**

- **Masaüstü (≥ 1024px):**
  - Solda 248px sabit kenar menüsü, ekran yüksekliğinin tamamında.
  - Sağda içerik alanı, kalan genişliğin **tamamı**.
  - İçerik alanının üstünde 64px yapışkan başlık çubuğu: sayfa başlığı solda, ana eylem sağda, alt kenarda 1px ayırıcı.
- **Tablet:** Kenar menüsü 72px'e küçülür ve yalnızca ikon gösterir. Üzerine gelince ipucu metni çıkar.
- **Telefon:** Kenar menüsü yok. Altta 4 sekmeli çubuk var: Takvim, Müşteriler, Bildirimler, Ayarlar.
- **Takvim:** Personel sütunları içerik alanını eşit böler. 6'dan fazla personel varsa yatay kaydırma açılır, saat sütunu sabit kalır.
- **Tablolar** (müşteriler, raporlar, SMS geçmişi): İçerik alanının tam genişliğinde.
  - Başlık satırı yapışkan, satır yüksekliği 52px.
  - Üzerine gelince satır zemini `--bg-subtle` olur.
  - Telefonda tablo, kenardan kenara liste satırlarına dönüşür.

**1.3 Vitrin (müşteri ana sayfası)**

- Her bölümün zemini kenardan kenara uzanır. Bölümler sırayla `--bg` ve `--bg-subtle` zeminlidir, bir bölüm de koyu olabilir.
- Açılış fotoğrafı tam genişlikte. Oranı masaüstünde 21:9, telefonda 4:5.
- Başlık ve buton 12 sütunluk ızgarada hizalanır.
- Hizmet ve ekip kartları ızgarada şöyle yerleşir:

  | Ekran | Hizmet kartları | Ekip kartları |
  |---|---|---|
  | Masaüstü | 4'lü | 4'lü |
  | Tablet | 2'li | 3'lü |
  | Telefon | Yatay kaydırılan şerit, her kart ekranın %82'si | 2'li |

- Galeri kenardan kenara mozaik; görseller arasında 4px boşluk.

**1.4 Telefonda listeler**

Liste grupları kenarlardan içeri girmiş yuvarlak kutular **olmayacak**. Tam genişlikte, düz zeminli satırlar olacak (iOS "plain" stili).
- Satırın içinde 16px boşluk.
- Gruplar arasında başlıklı 32px aralık.

---

### 2. RENK SİSTEMİ (ölçülmüş, kontrast sorunu yok)

Aşağıdaki değerler WCAG 2.2 AA'ya göre **hesaplanmıştır**. Kodda bunların dışında renk olmayacak.
- Elle yazılmış hex kodu, Tailwind varsayılan rengi ya da `opacity` ile soldurulmuş yazı kullanılmayacak.
- `text-gray-400` gibi soluk yazı yasak.

**2.1 Açık tema**

| Değişken | Değer | Kullanım | Kontrast (beyaz / #F5F5F7 zeminde) |
|---|---|---|---|
| `--bg` | `#FFFFFF` | Ana zemin | — |
| `--bg-subtle` | `#F5F5F7` | İkinci zemin, tablo hover, bölüm zemini | — |
| `--surface` | `#FFFFFF` | Kart, sheet, menü | — |
| `--text` | `#1D1D1F` | Başlık, gövde | 16.8 / 15.5 |
| `--text-2` | `#636366` | Açıklama, etiket | 6.0 / 5.5 |
| `--text-3` | `#6E6E73` | Yardımcı metin, saat etiketleri | 5.1 / 4.7 |
| `--separator` | `#D2D2D7` | Yalnızca dekoratif ayırıcı çizgi | (yazı değil) |
| `--control-border` | `#86868B` | Input, checkbox, radio ve kapalı toggle kenarı | 3.6 / 3.3 (≥ 3:1 gerekli) |
| `--success` | `#1D7A3A` | Onaylandı, geldi | 5.4 / 5.0 |
| `--warning` | `#A34700` | Yaklaşan, gecikme | 6.1 / 5.6 |
| `--danger` | `#C40012` | İptal, hata, şu-an çizgisi | 6.3 / 5.8 |

**Yasak:** `#86868B` ve daha açık griler yazı rengi olarak kullanılmaz. Beyaz zeminde kontrastları 3.6:1, yani okunmuyor.

**2.2 Koyu tema** (yalnızca cihaz koyu moddaysa açılır)

| Değişken | Değer | Kontrast (#000 / #1C1C1E / #2C2C2E zeminde) |
|---|---|---|
| `--bg` | `#000000` | — |
| `--bg-subtle` | `#1C1C1E` | — |
| `--surface` | `#1C1C1E` (üstteki katman `#2C2C2E`) | — |
| `--text` | `#F5F5F7` | 19.3 / 15.6 / 12.8 |
| `--text-2` | `#A1A1A6` | 8.2 / 6.6 / 5.4 |
| `--text-3` | `#98989D` | 7.3 / 5.9 / 4.9 |
| `--separator` | `#38383A` | (dekoratif) |
| `--control-border` | `#8E8E93` | 6.4 / 5.2 |
| `--success` | `#30D158` | 10.4 / 8.4 / 6.9 |
| `--warning` | `#FF9F0A` | 10.2 / 8.3 / 6.8 |
| `--danger` | `#FF6961` | 7.5 / 6.0 / 4.9 |

**2.3 Marka rengi** (işletme 6 renkten birini seçer)

Her marka renginin üç tonu var:
- **`--brand`:** dolu buton zemini, açık tema
- **`--brand-text`:** marka renginde yazı ve bağlantı, açık tema
- **`--brand-dark`:** koyu temada hem bağlantı hem dolu buton zemini

| Renk | `--brand` (buton, beyaz yazı) | `--brand-text` | `--brand-dark` |
|---|---|---|---|
| Mavi | `#0071E3` (4.7) | `#0066CC` (5.6 / 5.1) | `#2997FF` |
| Lacivert | `#1E3A5F` (11.5) | `#1E3A5F` | `#7FA8D9` |
| Orman | `#2F5D50` (7.5) | `#2F5D50` | `#6FBFA3` |
| Bordo | `#7A2E39` (9.2) | `#7A2E39` | `#E8828F` |
| Grafit | `#3A3A3C` (11.4) | `#3A3A3C` | `#C7C7CC` |
| Kahve | `#6B4E3D` (7.6) | `#6B4E3D` | `#CFA88E` |

**Mavi için önemli:** `#0071E3` gri zeminde yazı olarak 4.3:1 kalıyor, standardın altında. Mavi seçildiğinde yazı ve bağlantılarda `#0066CC` kullanılır.

**Koyu temada dolu butonlar:**
- Zemin `--brand-dark`, yazı `#000000`.
- Kontrastı hepsinde en az 6.9:1.
- Koyu temada lacivert ya da grafit buton siyah zeminde kaybolmasın diye bu kural var.

**2.4 Renk kullanım kuralları**

- Marka rengi yalnızca şu yerlerde: tıklanabilir öğeler, seçili durum, odak halkası, takvimde personel çizgisi, grafik çubukları.
- Büyük alanları marka rengine boyama. Tek istisna dolu butonlar ve küçük seçim göstergeleri.
- Durum renkleri tek başına anlam taşımaz. Her zaman **renk + ikon + metin** birlikte kullanılır.
- Randevu durumları her ekranda aynı şekilde gösterilir:

  | Durum | Renk | İkon (lucide) | Metin |
  |---|---|---|---|
  | Onaylandı | `--text-2` | `circle` | Onaylandı |
  | Geldi | `--success` | `circle-check` | Geldi |
  | Tamamlandı | `--text-3` | `check` | Tamamlandı |
  | Gecikiyor | `--warning` | `clock` | Gecikiyor |
  | Gelmedi | `--danger` | `user-x` | Gelmedi |
  | İptal | `--danger` | `circle-x` | İptal edildi |

- Fotoğraf üstündeki yazı için fotoğrafın alt kısmına `rgba(0,0,0,.55)` → şeffaf geçiş konur ve yazı beyaz olur. Yazının arkasındaki en açık nokta bile 4.5:1 sağlamalı.

---

### 3. TİPOGRAFİ

**Font:**
- `-apple-system, BlinkMacSystemFont, "SF Pro Display", "SF Pro Text", "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif`
- Web fontu indirilmez.

**Akışkan ölçek** (CSS `clamp`):

| Rol | Telefon → Masaüstü | Ağırlık | Satır yüksekliği | Harf aralığı |
|---|---|---|---|---|
| Display (vitrin açılışı) | 40 → 80px | 700 | 1.05 | -0.015em |
| H1 (sayfa başlığı) | 28 → 40px | 700 | 1.1 | -0.01em |
| H2 (bölüm) | 22 → 28px | 600 | 1.2 | -0.005em |
| H3 (kart başlığı) | 19 → 21px | 600 | 1.25 | 0 |
| Gövde | 17px | 400 | 1.47 | -0.022em |
| Küçük | 15px | 400 | 1.4 | -0.01em |
| Etiket | 13px | 500 | 1.3 | 0 |
| Büyük rakam (istatistik) | 28 → 40px | 600 | 1.1 | tabular-nums |

- Saat, tarih, fiyat ve sayı içeren her yerde `font-variant-numeric: tabular-nums`.
- 13px'ten küçük yazı yok. Tek istisna 11px sekme çubuğu etiketi.
- Başlıklar kısa olur, ünlem işareti olmaz.

---

### 4. BOŞLUK, KÖŞE, GÖLGE, İKON

**Boşluk ölçeği:** yalnızca 4, 8, 12, 16, 20, 24, 32, 40, 48, 64, 96, 128.

| Nerede | Telefon | Masaüstü |
|---|---|---|
| Vitrin bölümleri arası dikey boşluk | 64px | 128px |
| Panel bölümleri arası | 32px | 48px |

**Köşeler:**
- 6px: küçük rozet, takvim bloğu
- 10px: input, küçük kart
- 16px: kart, sheet
- 980px: buton (hap şekli)

Bunların dışında köşe ölçüsü yok.

**Gölgeler** (yalnızca iki seviye):
- `--shadow-1: 0 1px 2px rgba(0,0,0,.05), 0 4px 12px rgba(0,0,0,.05)`: kart
- `--shadow-2: 0 8px 32px rgba(0,0,0,.12)`: açılır menü, sheet, dialog
- Koyu temada gölge yok. Katmanlar zemin tonuyla ayrılır (`#1C1C1E` → `#2C2C2E`).

**Ayırıcı:** 1px `--separator`. Hiçbir yerde kalın kenarlık yok.

**İkonlar:**
- Yalnızca `lucide-react`, çizgi kalınlığı 1.5.
- Boyutlar: 16 / 20 / 24px. Rengi yazı rengiyle aynı ya da `--brand-text`.

---

### 5. BİLEŞENLER (hepsi `src/components/ui` altında, her sayfa bunları kullanır)

**Butonlar**

Üç tür:
- `primary`: dolu, marka rengi
- `secondary`: `--bg-subtle` zemin, `--text` yazı
- `ghost`: zemin yok, `--brand-text` yazı

Ayrıca `danger-ghost`: zemin yok, `--danger` yazı.

| Boyut | Yükseklik | Yatay iç boşluk | Yazı |
|---|---|---|---|
| sm | 32px | 14px | 15px |
| md | 44px | 20px | 17px |
| lg | 52px | 28px | 17px / 600 |

Durumlar:
- **hover:** `primary` %8 koyulaşır. `secondary` zemini `#E8E8ED` (koyu temada `#3A3A3C`) olur.
- **active:** `scale(.98)`
- **focus-visible:** 2px `--brand-text` halka, 2px boşlukla
- **disabled:** opaklık .4, `cursor: not-allowed`
- **loading:** metin yerinde 16px ince spinner. Buton genişliği değişmez.

**Input / Select / Textarea**
- Yükseklik 44px, 10px köşe, zemin `--surface`, 1px `--control-border` kenar.
- Focus'ta 2px `--brand-text` halka.
- Etiket üstte (13px/500, `--text-2`).
- Yardımcı metin altta (13px, `--text-3`).
- Hata olursa kenar ve mesaj `--danger`, mesajın başında `alert-circle` ikonu.
- Placeholder rengi `--text-3`. Placeholder etiketin yerine kullanılmaz.

**Seçim kontrolleri**
- Toggle: 51×31px, açıkken `--success`, kapalıyken `--bg-subtle` + 1px `--control-border`.
- Checkbox ve radio: 20px, kenarı `--control-border`, seçiliyken `--brand`.
- Segment kontrol (gün/hafta, masaüstü/telefon): `--bg-subtle` zemin, seçili parça `--surface` + `--shadow-1`.

**Kart**
- `--surface` zemin, 16px köşe, `--shadow-1`, iç boşluk 24px (telefonda 16px).
- Kart içinde kart yok.
- Tıklanabilir kartın tamamı tıklanır. Hover'da gölge değişmez, kenara 1px `--separator` eklenir.

**İstatistik**
- Etiket (13px/500, `--text-2`) → büyük rakam → altında küçük karşılaştırma metni ("geçen haftaya göre 12 fazla", `--text-2`).
- Renkli ok, yüzde rozeti ve renkli kart yok.

**Avatar**
- Personel fotoğrafı daire içinde: 32 / 40 / 64px.
- Fotoğraf yoksa `--bg-subtle` zemin üstünde `--text-2` renkli baş harfler.

**Durum göstergesi**
- 2.4'teki tabloya göre ikon + metin. Hap şeklinde renkli rozet yok.

**Tablo**
- Tam genişlik. Başlık 13px/500 `--text-2`, yapışkan.
- Satır 52px, satırlar arasında 1px ayırıcı.
- Sayı sütunları sağa hizalı ve tabular.
- Satıra tıklanınca sağdan detay paneli açılır (masaüstü) ya da yeni sayfa açılır (telefon).

**Sheet / Dialog**
- Telefonda alttan açılan sayfa (sheet), masaüstünde ortada 480px dialog.
- 16px köşe, `--shadow-2`.
- Arka plan karartması `rgba(0,0,0,.4)`, blur yok.
- Eylemler en altta: ana eylem sağda (masaüstü) ya da tam genişlikte (telefon).

**Toast**
- Altta ortada, en fazla 420px.
- Zemin `--text`, yazı `--bg`. Kontrastı 16.8:1.
- 4 saniye görünür. "Geri al" bağlantısı kalın.

**Boş durum**
- 48px `--text-3` ikon, H3 başlık, tek satır açıklama, tek buton.

**İskelet (yükleniyor)**
- İçeriğin şeklinde `--bg-subtle` bloklar, hafif opaklık nabzı (1.2s).
- Ortada dönen spinner yok.

**Komut paleti (panel, masaüstü)**
- `⌘K` / `Ctrl+K` ile açılır.
- Müşteri, randevu ve sayfa araması yapılır.
- 640px genişlikte, ortada dialog.

---

### 6. SAYFA TİPLERİ VE ANA BUTONUN YERİ (istisnasız)

| Tip | Sayfalar | Başlık | Ana eylem |
|---|---|---|---|
| A. Vitrin | `/s/[slug]` | Üstte yapışkan menü, kenardan kenara | Menünün sağında "Randevu Al". Telefonda ayrıca altta sabit çubuk |
| B. Adım akışı | Randevu adımları, OTP, kurulum | "‹ Geri" + "2 / 3" + H1 | **Altta sabit çubuk**: solda özet, sağda "Devam" |
| C. Liste / tablo | Müşteriler, hizmetler, personel, raporlar, bildirimler | Yapışkan başlık çubuğu (64px) | **Başlık çubuğunun sağında, üstte**: "Ekle" |
| D. Düzenleme formu | Hizmet/personel düzenle, ayarlar | Telefonda: "Vazgeç" – başlık – "Kaydet". Masaüstünde başlık çubuğunda sağda "Vazgeç" + "Kaydet" | Kaydet sağ üstte. Silme en altta, `danger-ghost`, onaylı |
| E. Takvim | Panel ana ekranı | Tarih + gün/hafta segmenti | "Plansız Ekle" sağ üstte. Randevu eylemleri sheet içinde |
| F. Sheet / dialog | Her türlü açılır pencere | Üstte başlık + kapat | En altta |

Ortak bileşenleri yaz ve her sayfa bunları kullansın:
- `<AppShell>`
- `<PageHeader>`
- `<BottomBar>`
- `<FormHeader>`
- `<Sheet>`
- `<Container>`

Sayfaların hiçbirinde başlık ya da buton yerleşimi elle yazılmayacak.

---

### 7. MODERNLİK (yapay zekâ havası olmadan)

**Modern görünümü şunlar sağlar:**
- Büyük ve kendinden emin tipografi
- Gerçek fotoğraflar
- Kenardan kenara kompozisyon
- Bol dikey boşluk
- Yapışkan başlık çubukları
- Segment kontroller
- Masaüstünde yoğun ama düzenli tablolar
- Klavye kısayolları
- Hızlı ve sessiz geçişler: 150–250ms, yalnızca opaklık ve konum

**Vitrin sitesinde hareket:**
- Sayfa kaydırılınca bölümler bir kez, 16px aşağıdan, opaklıkla belirir.
- `prefers-reduced-motion` açıksa hiç hareket olmaz.

**Panel ana ekranı (takvimin üstünde, masaüstünde yan yana, telefonda alt alta):**
- **"Sıradaki müşteri" kartı:** avatar 64px, isim (H3), saat ve hizmet, "Ara" ve "Geldi" butonları.
- **"Bugün" özeti:** randevu sayısı, tamamlanan, doluluk yüzdesi ve 4px yüksekliğinde marka renginde doluluk çubuğu.

**Kesinlikle kullanma (yapay zekâ şablonu görünümü):**
- Gradyan zeminler ve gradyan yazılar
- Mor, pembe, turkuaz, neon renkler
- Buzlu cam (`backdrop-blur`) ve parlayan kenarlar
- Siyah zemin + altın / gold renkler
- Emoji ve ✨ yıldız ikonları
- Renkli daire ya da kare içinde ikon dizileri
- "Neden biz?" kartları
- Yıldızlı müşteri yorumları
- "Yeni" / "Beta" rozetleri
- Ortası boş, büyük yuvarlak kartlar
- Kalın renkli kenarlıklar
- Her şeyin kart içinde olması
- `rounded-3xl` ve üstü köşeler
- `shadow-xl` ve üstü gölgeler
- Zıplayan, dönen, büyüyen hover efektleri
- Pazarlama dili ("Saniyeler içinde", "Akıllı", "Sihirli", "Harika!")

---

### 8. ERİŞİLEBİLİRLİK VE UYUM

- **WCAG 2.2 AA:**
  - Yazı en az 4.5:1. 24px ve üstü ya da 19px kalın yazı en az 3:1.
  - Kontrol kenarları ve odak halkası en az 3:1.
- Dokunma alanı en az 44×44px.
- Klavyeyle her şeye ulaşılır. Odak sırası mantıklı ve görünür olur.
- İkon butonlarında `aria-label` olur. Görsellerde anlamlı `alt` metni olur (dekoratifse `alt=""`).
- Koyu tema her sayfada eksiksiz çalışır. Koyu temada beyaz kutu, okunmayan yazı ya da kaybolan buton kalmaz.
- Yazı boyutu %200 büyütüldüğünde düzen bozulmaz.

---

### 9. ÇALIŞMA SIRASI

1. **Rapor:** 0. bölümdeki iki tablo.
2. **Token'lar:**
   - `globals.css` içine bölüm 2–4'teki değişkenler.
   - `tailwind.config.ts`: `theme.colors` tamamen bu değişkenlerle değiştirilir (`extend` değil). Köşe, gölge, boşluk ve font ölçekleri de buna göre ayarlanır.
3. **Bileşenler:**
   - Bölüm 5 ve 6'daki bütün bileşenler.
   - `/panel/tasarim` adında, yalnızca geliştirmede açılan bir sayfa: bütün bileşenleri açık ve koyu temada, tüm durumlarıyla (hover, focus, disabled, hata, yükleniyor) yan yana gösterir.
4. **Sayfalar:**
   - Sırayla bütün sayfaları yeni düzene ve bileşenlere geçir: önce vitrin, sonra randevu akışı, sonra panel ve yönetici.
   - Eski stil sınıflarını ve kullanılmayan CSS'i sil.
5. **Otomatik denetim:**
   - `@axe-core/playwright` ile bütün sayfaları açık ve koyu temada tara. `color-contrast` hatası 0 olmalı.
   - Şu komutların çıktısı boş olmalı:
     ```
     grep -rniE "gradient|backdrop-blur|shadow-(xl|2xl)|rounded-(3xl|\[2[4-9]px\])|purple|violet|indigo|pink|fuchsia|cyan|teal|gold|amber|#86868B|text-gray-[34]00|opacity-(50|60)" app src components --include=*.tsx --include=*.ts --include=*.css
     grep -rnP "[\x{1F300}-\x{1FAFF}\x{2728}]" app src components
     grep -rnoE "#[0-9A-Fa-f]{6}" app src components --include=*.tsx | grep -v globals.css
     ```
6. **Görsel kontrol:**
   - Her sayfanın 390px, 768px, 1440px ve 1920px genişlikte, açık ve koyu temada ekran görüntüsünü al.
   - Kenarlarda boş şerit kalmış mı, ana buton bölüm 6'daki yerinde mi, kontrol et.
   - Envanter tablosunu yeniden ver. Her satır "Uyuyor" olmalı.

Her adımın sonunda değişen dosyaların **tam içeriğini** ver. "...", "aynı kalacak" gibi kısaltmalar kullanma. Sonra "Devam edeyim mi?" diye sor.
