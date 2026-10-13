# Gemini İçin Geliştirme Promptu — Berber / Kuaför Randevu Uygulaması

> Aşağıdaki bölümün tamamını kopyalayıp Gemini'ye ilk mesaj olarak verin. Gemini'den işi aşamalar hâlinde yapmasını isteyin (bkz. "Çalışma Şekli"). Her aşama bitince bir sonrakine "devam" diyerek geçin.

---

## ROL

Sen deneyimli bir full-stack yazılım mimarı ve ürün tasarımcısısın. Üretim ortamına çıkacak, hızlı, güvenli ve bakımı kolay bir web uygulaması geliştireceksin. Kodu eksiksiz, çalışır hâlde ve tek seferde kopyalanıp çalıştırılabilecek şekilde yaz. "Buraya kodu ekleyin", "...", "TODO" gibi boşluklar bırakma. Emin olmadığın bir noktada varsayımını açıkça yaz ve devam et.

## ÜRÜN ÖZETİ

Türkiye'deki berber, kuaför ve güzellik salonları için **anlık randevu yönetim sistemi**. Üç tür kullanıcı var:

1. **Müşteri** — Salonun bağlantısına girer, hizmet + personel + saat seçer, telefonuyla randevu oluşturur.
2. **Personel (berber/kuaför)** — Kendi kullanıcı adı ve şifresiyle girer; günlük takvimini anlık görür ve yönetir.
3. **Salon sahibi (yönetici)** — Personeli, hizmetleri, çalışma saatlerini, SMS/e-posta ayarlarını yönetir; tüm takvimleri görür.

Uygulama web'de çalışır, **PWA** olarak iOS ve Android'de tek dokunuşla ana ekrana eklenir ve native uygulama gibi tam ekran açılır. Tüm sistem **Docker** ile tek komutla ayağa kalkar.

Arayüz dili: **Türkçe**. Saat dilimi: **Europe/Istanbul**. Para birimi: **TRY (₺)**. Telefon formatı: **+90 5XX XXX XX XX**.

---

## TEKNOLOJİ YIĞINI (bunu kullan, değiştirme)

| Katman | Teknoloji |
|---|---|
| Framework | **Next.js 15 (App Router)** + **TypeScript (strict)** |
| Stil | **Tailwind CSS** — yalnızca aşağıdaki tasarım token'larıyla; hazır UI kiti teması kullanma |
| UI bileşenleri | Radix UI primitives (erişilebilirlik için), ikonlar: **lucide-react** (ince çizgi, 1.5px) |
| Veritabanı | **PostgreSQL 16** |
| ORM | **Drizzle ORM** + drizzle-kit migration |
| Önbellek / Pub-Sub / Kuyruk | **Redis 7** + **BullMQ** |
| Anlık güncelleme | **Server-Sent Events (SSE)** — Redis pub/sub üzerinden yayın |
| Kimlik doğrulama | Personel: kullanıcı adı + şifre (**argon2id**), HTTP-only, Secure, SameSite=Lax oturum çerezi. Müşteri: telefona **SMS ile tek kullanımlık kod (OTP)** |
| Doğrulama | **Zod** (hem istemci hem sunucu) |
| Form | react-hook-form |
| Tarih/saat | date-fns + date-fns-tz |
| Bildirim | **Web Push (VAPID)** — `web-push` paketi |
| SMS | Sağlayıcı-bağımsız arayüz; varsayılan adaptörler: **Netgsm**, **İleti Merkezi**, ayrıca **Twilio**; geliştirmede konsola yazan `MockSmsProvider` |
| E-posta | **Nodemailer (SMTP)**; geliştirmede **Mailpit** container'ı |
| Test | Vitest (birim), Playwright (uçtan uca, en az ana randevu akışı) |
| Container | Docker multi-stage build, `docker-compose.yml` |

---

## MİMARİ

```
[Tarayıcı / PWA]
   │  HTTPS
   ▼
[web]  Next.js (standalone output)  ── SSE ──▶ istemciler
   │         │
   │         └── Redis pub/sub  (salon:{id}:events kanalı)
   ▼
[postgres]   [redis]
                ▲
[worker]  BullMQ işçisi: hatırlatma SMS/push/e-posta, toplu SMS, doluluk uyarıları
```

- `web` ve `worker` aynı kod tabanından, aynı imajdan farklı komutla çalışır.
- Her randevu değişikliği (oluştur, taşı, iptal, araya al, yer değiştir) → veritabanı transaction'ı → commit sonrası Redis'e olay yayını → o salona bağlı tüm açık ekranlar SSE ile **1 saniyenin altında** güncellenir. Polling yok.
- SSE bağlantısı koparsa istemci otomatik yeniden bağlanır (`Last-Event-ID` ile kaçan olayları tamamlar).
- Çok kiracılı (multi-tenant): her salonun kendi `slug`'ı vardır → `/s/{salon-slug}` müşteri sayfası. Tüm sorgular `salon_id` ile filtrelenir.

---

## VERİ MODELİ (Drizzle şeması olarak yaz)

- **salons**: id, slug (benzersiz), ad, adres, telefon, e-posta, saat dilimi, logo_url, randevu_aralığı_dk (varsayılan 15), iptal_son_süre_dk (varsayılan 120), sms_aktif, eposta_aktif, oluşturulma
- **staff**: id, salon_id, ad_soyad, kullanıcı_adı (salon içinde benzersiz), şifre_hash, rol (`owner` | `staff`), telefon, e-posta, renk_kodu (takvimde ayırt etmek için, kurumsal paletten seçilir), aktif, sıra
- **services**: id, salon_id, ad, süre_dk, fiyat, aktif, sıra
- **staff_services**: staff_id, service_id, (opsiyonel personele özel süre/fiyat)
- **working_hours**: staff_id, haftanın_günü, başlangıç, bitiş, mola_başlangıç, mola_bitiş
- **time_off**: staff_id, başlangıç, bitiş, açıklama (izin, tatil, bloke saat)
- **customers**: id, salon_id, ad_soyad, telefon (E.164, salon içinde benzersiz), e-posta, not, sms_izni (KVKK/İYS onayı), oluşturulma, toplam_randevu, gelmedi_sayısı
- **appointments**: id, salon_id, staff_id, customer_id, service_id, başlangıç (timestamptz), bitiş (timestamptz), durum (`confirmed` | `checked_in` | `completed` | `cancelled` | `no_show`), kaynak (`online` | `walk_in` | `phone` | `staff`), not, iptal_nedeni, iptal_eden, sürüm (iyimser kilitleme için integer), oluşturulma, güncellenme
- **appointment_events**: değişiklik geçmişi (kim, ne zaman, ne yaptı, eski/yeni değer) — denetim kaydı
- **push_subscriptions**: id, sahip_tipi (`staff` | `customer`), sahip_id, endpoint, p256dh, auth, user_agent
- **notifications_log**: kanal (`sms` | `email` | `push`), alıcı, şablon, durum, sağlayıcı_yanıtı, gönderim_zamanı
- **otp_codes**: telefon, kod_hash, son_kullanma, deneme_sayısı
- **waitlist**: salon_id, customer_id, staff_id (opsiyonel), tarih, tercih_edilen_aralık

**Çift rezervasyonu veritabanı seviyesinde engelle:** `btree_gist` eklentisi ile `appointments` tablosuna
`EXCLUDE USING gist (staff_id WITH =, tstzrange(başlangıç, bitiş, '[)') WITH &&) WHERE (durum NOT IN ('cancelled','no_show'))`
kısıtı ekle. Aynı anda iki müşteri aynı saati seçerse biri başarılı olur, diğerine "Bu saat az önce doldu, lütfen başka bir saat seçin" mesajı ve güncel boş saatler gösterilir.

Sık kullanılan sorgular için indeksler: (salon_id, başlangıç), (staff_id, başlangıç), (customer_id), (telefon).

---

## ÖZELLİKLER

### A. Müşteri Tarafı (`/s/{salon-slug}`)

1. **Randevu oluşturma akışı** (en fazla 4 adım, her adım tek ekran):
   1. Hizmet seç (süre ve fiyat görünür, birden fazla hizmet seçilebilir → süreler toplanır)
   2. Personel seç ("Fark etmez" seçeneği → en erken müsait personel)
   3. Tarih ve saat seç — yatay kaydırılan gün şeridi + müsait saat butonları. Dolu saatler gösterilmez veya pasif görünür. Boş saatler **anlık** güncellenir (biri o saati alırsa ekrandan anında kalkar).
   4. Ad soyad + telefon → SMS ile gelen 6 haneli kodu gir → onay
2. **Onay ekranı**: randevu özeti, "Takvime ekle" (.ics dosyası), salonu ara (`tel:`), yol tarifi (Apple/Google Maps bağlantısı).
3. **Randevularım**: telefonla giriş yapan müşteri gelecek/geçmiş randevularını görür; iptal son süresinden önceyse **iptal** veya **saat değiştirme** yapabilir.
4. **Bildirimler**: müşteri izin verirse web push; vermezse SMS.
   - Randevu onayı (anında)
   - Hatırlatma: **24 saat önce** ve **1 saat önce** (salon ayarlarından değiştirilebilir)
   - "Randevu saatine 15 dakika kaldı" push bildirimi
   - Randevu personel tarafından taşındıysa veya iptal edildiyse anında bildirim
5. **Bekleme listesi**: istenen gün doluysa "Yer açılırsa haber ver" → iptal olduğunda sıradaki müşteriye SMS/push.
6. KVKK aydınlatma metni ve ticari ileti izni için ayrı, işaretlenmemiş onay kutuları.

### B. Personel Paneli (`/panel`)

1. **Giriş**: salon kodu + kullanıcı adı + şifre. Her personelin kendi şifresi vardır. Şifre değiştirme, 5 hatalı denemede 15 dakika kilit, "beni hatırla" (30 gün).
2. **Günlük takvim (ana ekran)**:
   - Dikey zaman çizelgesi (ör. 09:00–21:00), 15 dakikalık ızgara, şu anki saati gösteren ince kırmızı çizgi.
   - Yönetici tüm personeli yan yana sütunlar hâlinde görür; personel varsayılan olarak yalnızca kendi sütununu görür.
   - Randevu kartında: saat, müşteri adı, hizmet, durum, kaynak simgesi (online / plansız / telefon).
   - Gün ve hafta görünümü, üstte tarih değiştirici, "Bugün" butonu.
   - Tüm değişiklikler **anlık** (SSE) — başka bir cihazdan eklenen randevu animasyonsuz, sade bir vurgu ile belirir.
3. **Plan dışı müşteri ekleme (walk-in)**: tek dokunuşla "Şimdi Ekle" → en yakın boşluğa yerleştirilir; isim/telefon opsiyonel ("Misafir").
4. **Araya alma**: iki randevu arasındaki boşluğa (boşluk hizmet süresinden kısaysa uyarı ver) veya sonraki randevuları belirli dakika **kaydırarak** müşteri ekle. Kaydırılan müşterilere otomatik "Randevunuz X dakika ileri alındı" SMS'i gönderme seçeneği (onay kutusu, varsayılan açık).
5. **Taşıma / yer değiştirme**:
   - Masaüstünde sürükle-bırak, mobilde uzun bas → sürükle. Ayrıca erişilebilir alternatif: karttaki "Taşı" menüsü ile saat/personel seçimi.
   - İki randevunun **yerini değiştir** (swap) — tek transaction, kısıt ihlali olmadan (geçici olarak bir randevuyu iptal statüsüne almadan, ertelenebilir kısıt veya sıralı güncelleme ile).
   - Başka personele aktarma.
   - Her taşımada müşteriye bildirim gönderme seçeneği.
   - Çakışma olursa işlem reddedilir, kullanıcıya net Türkçe mesaj gösterilir.
6. **İptal**: neden seçimi (müşteri istedi / personel müsait değil / gelmedi / diğer) + müşteriye SMS gönder kutusu. Geri alma (undo) 10 saniye boyunca mümkün.
7. **Müşteriyi ara**: kartta telefon simgesi → `tel:` bağlantısı ile doğrudan arama.
8. **SMS gönder**: kartta "SMS" → hazır şablonlar (Gecikiyorum, Randevunuz hazır, Lütfen bizi arayın, Serbest metin) → karakter sayacı (Türkçe karakterlerde 1 SMS = 70/155 karakter kuralını hesaba kat).
9. **Durum işaretleme**: Geldi → Tamamlandı → Gelmedi (no-show sayısı müşteri kartına yansır).
10. **Müşteri kartı**: geçmiş randevular, notlar ("kısa kesim sever" vb.), gelmeme sayısı, toplam ziyaret.
11. **Personel bildirimleri (push + isteğe bağlı SMS/e-posta)**:
    - Yeni online randevu geldi
    - Randevu iptal edildi / müşteri saatini değiştirdi
    - Bir sonraki müşterine 10 dakika kaldı
    - **"Bugün 14:00–18:00 arası tamamen dolu"** gibi doluluk uyarıları
    - Günün başında özet: "Bugün 9 randevun var, ilki 09:30"

### C. Yönetici Paneli (`/panel/ayarlar`) — yalnızca `owner`

- Salon bilgileri, logo, müşteri bağlantısı + **QR kod** (yazdırılabilir).
- Personel ekle/düzenle/pasifleştir, şifre sıfırla, rol ata.
- Hizmetler, süreler, fiyatlar; personel-hizmet eşleştirme.
- Çalışma saatleri, molalar, izinler, resmî tatiller.
- Randevu kuralları: zaman aralığı, en erken/en geç rezervasyon süresi, iptal son süresi, aynı müşteri için günlük azami randevu.
- **Bildirim ayarları**: SMS sağlayıcı seçimi ve API bilgileri (veritabanında şifrelenmiş saklanır), gönderici başlığı, şablon metinleri (değişken desteği: `{musteri_adi}`, `{tarih}`, `{saat}`, `{personel}`, `{salon}`), hangi olayda hangi kanal.
- **Toplu SMS / e-posta**: filtre (son 60 günde gelmeyenler, tüm izinli müşteriler vb.) → yalnızca ticari ileti izni olanlara gönderilir, gönderim kuyruğa alınır.
- **Raporlar** (sade tablolar + tek renkli basit çubuk grafikler): günlük/haftalık randevu sayısı, doluluk oranı, iptal ve gelmeme oranı, personel bazlı ciro tahmini.

---

## PWA VE MOBİL

- `manifest.webmanifest`: `display: "standalone"`, `theme_color` ve `background_color` paletten, 192/512 px ikonlar + **maskable** ikon, `apple-touch-icon` (180×180), `apple-mobile-web-app-capable`, `apple-mobile-web-app-status-bar-style: default`.
- Personel ve müşteri için iki ayrı manifest/başlangıç adresi (`/panel` ve `/s/{slug}`).
- **Service Worker**: uygulama kabuğu ve statik dosyalar önbellekte (açılış anında), API istekleri network-first. Çevrimdışıyken son yüklenen takvim salt-okunur gösterilir ve üstte "Çevrimdışısınız" şeridi çıkar.
- **"Ana ekrana ekle" yönlendirmesi**: Android/Chrome'da `beforeinstallprompt` ile kendi sade butonumuz; iOS Safari'de bunu desteklemediği için Paylaş → "Ana Ekrana Ekle" adımlarını gösteren küçük, kapatılabilir bir alt bilgi kartı.
- **iOS web push notu**: iOS 16.4+ yalnızca ana ekrana eklenmiş PWA'larda push destekler. Bu yüzden bildirim izni, uygulama ana ekrandan açıldığında ve kullanıcı bir butona bastığında istenir (sayfa açılır açılmaz izin isteme). iOS'ta push mümkün değilse otomatik olarak SMS'e düş.
- Güvenli alanlar (`env(safe-area-inset-*)`), 44×44 px minimum dokunma alanı, kaydırmada sıçrama yok, yatay taşma yok.

---

## PERFORMANS HEDEFLERİ (zorunlu)

- Müşteri sayfası: mobil 4G'de **LCP < 1,5 sn**, ilk yük JS **< 120 KB gzip**. Lighthouse Performans, Erişilebilirlik, En İyi Uygulamalar ≥ 95, PWA kontrolleri geçer.
- Müsait saat hesaplama API'si: **p95 < 100 ms** (müsaitlik sonuçlarını Redis'te kısa süre önbelleğe al, randevu değişince ilgili anahtarı sil).
- Server Components varsayılan; yalnızca etkileşimli parçalar `"use client"`.
- Fontlar: sistem fontu yığını (indirme yok). Görseller `next/image`, AVIF/WebP.
- Ağır kütüphanelerden kaçın (moment.js, büyük grafik kütüphaneleri, tam UI kitleri yok). Grafikler için hafif SVG bileşenleri yaz.
- Veritabanı bağlantı havuzu, N+1 sorgu yok, takvim tek sorguda gelir.

---

## TASARIM — KESİN KURALLAR

Bu bölüm önerilerden oluşmuyor; hepsi zorunlu. Amaç: uygulama apple.com/tr, bir banka uygulaması ya da iOS'un kendi Takvim ve Ayarlar ekranları gibi görünsün. "Yapay zekâyla yapılmış bir SaaS şablonu" gibi görünmesin. Emin olmadığın her durumda daha sade, daha az renkli, daha az süslü olanı seç.

### 0. Referans ekranlar ve vitrin şablonları — önce bunlara bak

Bu mesaja `tasarim/` klasörü ve `YENIDEN_TASARIM_PROMPTU.md` eklendi. Tasarımla ilgili her konuda o dosyanın **2. (Genel tasarım kuralları)** ve **3. (Müşteri ana sayfası = şablon sistemi)** bölümleri bu projenin parçasıdır, aynen uygula:
- Müşteri ana sayfası, işletmenin şablon (Klasik / Galeri / Sade), marka rengi, bölüm ve fotoğraf seçerek özelleştirdiği bir vitrin sitesidir (`20–22` ve `30` numaralı referanslar).
- Randevu akışı ve panel `02–04` ve `10–12` numaralı referanslara birebir benzer.
- Siyah-gold, serif font, makas/ustura ikonları, emoji kesinlikle yok.

### 1. Hazır stil dosyası — aynen kullan

Tasarım token'larını kendin uydurma. `app/globals.css` dosyası `tasarim/style.css` ile aynı olacak. Token kısmı şudur:

```css
:root {
  --bg: #FFFFFF;
  --bg-subtle: #F5F5F7;
  --surface: #FFFFFF;
  --border: #D2D2D7;
  --border-light: #E8E8ED;
  --text: #1D1D1F;
  --text-2: #6E6E73;
  --text-3: #86868B;
  --accent: #0071E3;
  --accent-hover: #0077ED;
  --success: #1D7A3A;
  --warning: #B25000;
  --danger: #D70015;
  --radius-sm: 8px;
  --radius: 12px;
  --font: -apple-system, BlinkMacSystemFont, "SF Pro Text", "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
}
@media (prefers-color-scheme: dark) {
  :root {
    --bg: #000000;
    --bg-subtle: #1C1C1E;
    --surface: #1C1C1E;
    --border: #38383A;
    --border-light: #2C2C2E;
    --text: #F5F5F7;
    --text-2: #A1A1A6;
    --text-3: #8E8E93;
    --accent: #2997FF;
    --accent-hover: #47A8FF;
    --success: #30D158;
    --warning: #FF9F0A;
    --danger: #FF453A;
  }
}
html { -webkit-text-size-adjust: 100%; }
body {
  margin: 0;
  background: var(--bg);
  color: var(--text);
  font: 17px/1.47 var(--font);
  letter-spacing: -0.022em;
  -webkit-font-smoothing: antialiased;
}
.num { font-variant-numeric: tabular-nums; }
```

`tailwind.config.ts` içinde `theme.colors` objesini **tamamen bu değişkenlerle değiştir** (`extend` değil, override). Böylece `purple-500`, `indigo-600` gibi varsayılan renkler hiç olmasın:

```ts
colors: {
  transparent: "transparent",
  current: "currentColor",
  white: "#FFFFFF",
  black: "#000000",
  bg: "var(--bg)",
  subtle: "var(--bg-subtle)",
  surface: "var(--surface)",
  line: "var(--border)",
  "line-light": "var(--border-light)",
  ink: "var(--text)",
  "ink-2": "var(--text-2)",
  "ink-3": "var(--text-3)",
  accent: "var(--accent)",
  "accent-hover": "var(--accent-hover)",
  success: "var(--success)",
  warning: "var(--warning)",
  danger: "var(--danger)",
},
borderRadius: { none: "0", sm: "8px", DEFAULT: "12px", full: "9999px" },
boxShadow: { none: "none", sheet: "0 -1px 0 var(--border), 0 8px 32px rgba(0,0,0,0.12)" },
fontFamily: { sans: ["var(--font)"] },
```

Inter, Poppins, Manrope, Plus Jakarta Sans gibi Google Fonts **yükleme**. Yalnızca sistem fontu kullan.

### 2. Yasak listesi

Kodda aşağıdakilerden biri geçerse tasarım hatalı sayılır. Teslim etmeden önce kodu bu kelimeler için tara ve hepsini temizle:

- **Tailwind sınıfları:** `bg-gradient-`, `from-`, `via-`, `to-` (gradyan; tek istisna Galeri şablonunda fotoğraf üstü yazı için siyahtan şeffafa geçiş), `bg-clip-text`, `text-transparent`, `backdrop-blur`, `blur-`, `shadow-lg`, `shadow-xl`, `shadow-2xl`, `rounded-3xl`, `animate-bounce`, `animate-pulse` (iskelet yükleme hariç), `animate-ping`, `ring-` ile parlama, `hover:scale-`, `hover:-translate-y`
- **Renk adları:** purple, violet, indigo, fuchsia, pink, rose, cyan, teal, emerald, lime, sky, amber, orange, yellow. Bunlar config'te zaten olmayacak, inline style ile de yazma.
- **Emoji:** arayüzde, bildirim metinlerinde ve SMS şablonlarında hiç olmayacak.
- **İkonlar:** ✨ yıldız, sihirli değnek, roket, alev, taç, kupa yok. Renkli daire veya kare içinde ikon yok. İkon her zaman tek renk (`ink-2` ya da `accent`), 20px, 1.5 çizgi kalınlığında.
- **Düzen kalıpları:**
  - Panel ekranlarında "hero" bölümü (vitrin sitesindeki açılış bölümü hariç; o `20–22` referanslarındaki gibi Apple tarzı olacak)
  - "Neden biz?" ya da "Özellikler" diye 3'lü ikonlu kart ızgarası
  - Müşteri yorumları ve yıldızlar
  - Başlığın üstünde "Yeni" ya da "Beta" hap rozeti
  - Dashboard'da trend oklu, yüzdeli renkli istatistik kartları
  - Sol tarafta renkli ikonlu kalın sidebar
  - Kart içinde kart
  - Her şeyi kartlara sarmak (liste satırı gereken yerde liste satırı kullan)
- **Metin dili:**
  - Ünlem işareti yok.
  - "Harika!", "Hadi başlayalım", "Saniyeler içinde", "Kolayca", "Akıllı", "Sorunsuz", "Mükemmel", "Yapay zekâ" yok.
  - Pazarlama cümlesi yok.
  - Buton metni fiil olacak ve 1–2 kelimeyi geçmeyecek: "Devam", "Onayla", "İptal Et", "Ara", "SMS Gönder".

### 3. Bileşen ölçüleri

| Bileşen | Kural |
|---|---|
| Birincil buton | `bg-accent text-white`, yükseklik 44px, `rounded-full`, yatay padding 22px, 17px/400. Hover'da yalnızca `bg-accent-hover`. Gölge ve kenarlık yok. |
| İkincil buton | Zemin yok, `text-accent`, aynı yükseklik. Ya da `text-accent` düz metin bağlantısı. |
| Tehlikeli eylem | Kırmızı zeminli buton yok. `text-danger` düz metin + onay sayfası (action sheet). |
| Input | Yükseklik 44px, `bg-subtle`, kenarlık yok, `rounded` (12px), padding 0 14px. Focus'ta 2px `accent` outline, glow yok. Etiket input'un üstünde, 13px, `ink-2`. |
| Liste | iOS Ayarlar tarzı: beyaz grup, satır yüksekliği en az 52px, satırlar arasında sol 16px girintili 0.5px `line` ayırıcı. Sağda `ink-3` renkli chevron. Grup arka planı `subtle`. |
| Kart | Panelde mümkünse kullanma; gerekirse `bg-surface`, 1px `line-light` kenarlık, 12px köşe, gölge yok. Vitrin sitesinde `vitrin.css` içindeki `.card` (18px köşe, çok hafif gölge). |
| Alt sayfa (sheet) | Mobilde alttan açılır, üst köşeler 12px, tutamaç çizgisi 36×5px `line`. Tek gölge `shadow-sheet`. Arka plan karartması `rgba(0,0,0,0.4)`, blur yok. |
| Sekme çubuğu (mobil panel) | Altta 49px + safe-area. 4 sekme: Takvim, Müşteriler, Bildirimler, Ayarlar. Seçili ikon ve etiket `accent`, diğerleri `ink-3`. Etiket 10px. |
| Üst başlık | iOS büyük başlık: 34px/700, sola hizalı, kaydırınca 17px/600 ortalanmış başlığa küçülür. Arka plan düz `bg`, alt kenarda 0.5px `line`. |
| Durum etiketi | Hap rozet yok. Küçük 8px nokta + metin: `● Onaylandı`. Nokta durum renginde, metin `ink-2`. |
| Toast | Altta ortada, `bg-ink text-bg`, 14px, `rounded` (12px), 3 saniye. "Geri Al" bağlantısı `accent` değil, kalın beyaz metin. |
| Boşluklar | Yalnızca 4, 8, 12, 16, 20, 24, 32, 48. Sayfa yan boşluğu mobilde 16px, masaüstünde 24px. İçerik genişliği en fazla 980px. |
| Animasyon | Yalnızca opaklık ve konum, 200ms, `cubic-bezier(0.25, 0.1, 0.25, 1)`. Sayfa açılışında giriş animasyonu yok. |

### 4. Ekran ekran düzen

**Müşteri: salon ana sayfası (`/s/{slug}`)** — vitrin sitesi; bkz. `YENIDEN_TASARIM_PROMPTU.md` bölüm 3 ve `20–22` numaralı referanslar.
- Randevu adımlarında seçili satırın solunda marka renginde onay işareti. Satırın arka planı değişmez.
- Üstte ince adım göstergesi: "1 / 3", 13px `ink-3`. İlerleme çubuğu yok.

**Müşteri: saat seçimi**
- Gün şeridi: 7 gün yan yana. Her gün iki satır: kısa gün adı 13px `ink-3`, gün sayısı 20px. Seçili gün `accent` dolu daire içinde beyaz sayı. Bugün `accent` renkli sayı.
- Saatler 4 sütunlu ızgara. Her saat 44px yükseklikte, `bg-subtle`, `rounded-sm`. Seçili saat `bg-accent text-white`. Dolu saatler hiç gösterilmez.
- Boş saat yoksa tek satır: "Bu gün için boş saat yok." ve altında "Sonraki boş gün: Perşembe" bağlantısı.

**Personel: takvim (ana ekran)**
```
┌─────────────────────────────────────┐
│ Bugün                    [+] Ekle   │  ← büyük başlık; Ekle accent metin
│ 14 Ekim Salı  ‹  ›                  │  ← 15px ink-2
├──────┬──────────────────────────────┤
│ 09:00│▍Ahmet Y. · Saç Kesimi       │  ← sol 3px personel renk çizgisi
│      │▍09:00–09:30                  │     zemin subtle, yazı ink
│ 09:30│                              │
│ 10:00│▍Mehmet K. · Sakal   ● Geldi  │
│ ─────┼──────────────────── 10:12 ── │  ← şu an çizgisi: 1px danger
│ 10:30│                              │
└──────┴──────────────────────────────┘
│ Takvim  Müşteriler  Bildirim  Ayarlar│  ← sekme çubuğu
```
- Saat sütunu 56px genişliğinde, 13px `ink-3`, tabular-nums. Izgara çizgileri 0.5px `line-light`. 15 dakikalık çizgiler daha da soluk ya da hiç yok, yalnızca saat başları.
- Randevu bloğu: `bg-subtle`, solda 3px personel rengi, `rounded-sm`. İçinde müşteri adı 15px/600, hizmet ve saat 13px `ink-2`. Gölge yok, renkli zemin yok.
- Bloğa dokununca alttan sheet açılır. Sheet'te sırasıyla: müşteri adı (büyük), telefon, hizmet, saat. Altında tek sıra halinde 4 eylem: Ara, SMS, Taşı, İptal. Her eylem 24px ikon üstte, 12px etiket altta, renk `accent`; İptal `danger`.
- Masaüstünde personel sütunları yan yana. Sütun başlığında yalnızca personel adı 15px/600. Avatar fotoğrafı ve baş harf dairesi yok.

**Personel: giriş**
- Ekranın ortasında, en fazla 360px genişlikte: salon logosu (yoksa salon adı 22px/600), 3 input (Salon kodu, Kullanıcı adı, Şifre), birincil "Giriş" butonu tam genişlikte.
- Arka plan düz `bg`. İllüstrasyon veya desen yok.

**Yönetici: raporlar**
- Rakamlar düz metin satırları: "Bu hafta 142 randevu" (sayı 28px/600 tabular-nums, açıklama 15px `ink-2`). Renkli kart, ok veya yüzde rozeti yok.
- Grafik gerekiyorsa tek renk (`accent`) ince çubuklar. Eksen çizgileri `line-light`, etiketler 12px `ink-3`. Grafik kütüphanesi kullanma, SVG ile çiz.

### 5. Referans duygu

Bir ekranı bitirdiğinde kendine şunu sor: "Bu ekran iPhone'daki Takvim, Saat veya Ayarlar uygulamasının içinde olsa yadırganır mı?" Yadırganıyorsa sadeleştir. Renk yalnızca üç yerde kullanılır:
- tıklanabilir şeyler (`accent`)
- durum noktaları
- personel çizgisi

Geri kalan her şey siyah, beyaz ve gri.

### 6. Teslimden önce kontrol

Her aşamanın sonunda şu iki komutu çalıştırmış gibi kontrol et ve sonucunu yaz. Çıktı boş değilse düzelt:
```
grep -rniE "gold|#d4af37|amber|yellow|backdrop-blur|shadow-(lg|xl|2xl)|rounded-3xl|purple|violet|indigo|pink|fuchsia|emerald|cyan|teal|font-serif" app src components
grep -rnP "[\x{1F300}-\x{1FAFF}\x{2728}]" app src components
```

---

## GÜVENLİK VE UYUMLULUK

- Şifreler argon2id; oturum token'ları veritabanında hash'li; CSRF koruması (SameSite + origin kontrolü).
- **Rate limiting** (Redis): OTP gönderimi telefon başına 3/10 dk ve IP başına; giriş denemeleri; randevu oluşturma.
- OTP: 6 hane, 5 dk geçerli, en fazla 5 deneme, hash'li saklanır.
- Yetkilendirme her API'de sunucu tarafında: personel yalnızca kendi salonunun verisini, `staff` rolü ayarları göremez.
- SMS sağlayıcı API anahtarları ortam değişkeni veya AES-256-GCM ile şifrelenmiş DB alanında.
- Güvenlik başlıkları: CSP, HSTS, X-Content-Type-Options, Referrer-Policy, Permissions-Policy.
- **KVKK**: aydınlatma metni sayfası, açık rıza kayıtları, müşterinin verisini silme talebi (yönetici panelinden anonimleştirme). Toplu/ticari SMS yalnızca izinli müşterilere; bilgilendirme SMS'leri (randevu onayı/hatırlatma) ayrı kategoride. İYS (İleti Yönetim Sistemi) entegrasyonuna uygun alan yapısı bırak.
- Tüm girdiler Zod ile doğrulanır; loglarda telefon numarası maskelenir.

---

## DOCKER

- **Dockerfile** (multi-stage): `deps` → `build` → `runner`. Taban imaj `node:22-alpine`. Next.js `output: "standalone"`. Root olmayan kullanıcı. `HEALTHCHECK` (`/api/health`). Nihai imaj < 200 MB.
- **docker-compose.yml** servisleri:
  - `web` (port 3000)
  - `worker` (aynı imaj, `node dist/worker.js`)
  - `postgres` (kalıcı volume, healthcheck)
  - `redis` (AOF açık, kalıcı volume)
  - `mailpit` (yalnızca geliştirme profili; 8025 web arayüzü)
- `docker-compose.prod.yml`: önüne **Caddy** reverse proxy (otomatik HTTPS — PWA ve push için HTTPS şart).
- Container başlarken migration'ları otomatik çalıştır (`migrate` adımı veya ayrı tek seferlik servis).
- `.env.example`: tüm değişkenler açıklamalı (`DATABASE_URL`, `REDIS_URL`, `SESSION_SECRET`, `ENCRYPTION_KEY`, `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT`, `SMS_PROVIDER`, `NETGSM_USERCODE`, `NETGSM_PASSWORD`, `NETGSM_HEADER`, `SMTP_*`, `APP_URL`).
- `seed` komutu: örnek salon ("Demo Berber"), 3 personel, 6 hizmet, bir haftalık örnek randevular, giriş bilgileri README'de.
- Hedef: `git clone` → `cp .env.example .env` → `docker compose up -d` → http://localhost:3000 çalışıyor.

---

## PROJE YAPISI

```
/app
  /(public)/s/[slug]/...        müşteri akışı
  /(panel)/panel/...            personel & yönetici
  /api/...                      route handler'lar (REST, JSON)
  /api/events/route.ts          SSE uç noktası
/src
  /db (schema.ts, migrations, seed.ts)
  /lib (auth, availability, sms/, email/, push/, rate-limit, realtime)
  /services (appointment.service.ts — tüm iş kuralları burada, UI'dan bağımsız)
  /components/ui                tasarım sistemi bileşenleri
  /worker (index.ts, jobs/)
/public (icons, manifest, sw.js)
Dockerfile, docker-compose.yml, docker-compose.prod.yml, .env.example, README.md
```

---

## KABUL KRİTERLERİ

1. İki farklı tarayıcıda aynı salonun paneli açıkken birinde randevu taşınınca diğerinde 1 saniye içinde güncellenir.
2. İki müşteri aynı anda aynı saati onaylarsa yalnızca biri başarılı olur (testle kanıtla).
3. Walk-in ekleme 2 dokunuşta tamamlanır.
4. Araya alma ile kaydırılan randevuların müşterilerine SMS kuyruğa düşer (Mock sağlayıcı loglarında görünür).
5. Hatırlatma işleri randevu taşınınca/iptal edilince yeniden planlanır/silinir (BullMQ job id = randevu id + tür).
6. Uygulama iPhone Safari ve Android Chrome'da ana ekrana eklenip tam ekran açılır; ana ekrandan açılan PWA'da push bildirimi alınır.
7. Lighthouse hedefleri tutturulur.
8. `docker compose up` sonrası seed verisiyle her şey çalışır.
9. Arayüzde paletin dışında renk, gradyan veya emoji yoktur.

---

## ÇALIŞMA ŞEKLİ

İşi aşağıdaki aşamalarla yap. Her aşamanın sonunda **dosya ağacını ve tam dosya içeriklerini** ver, sonra "Devam edeyim mi?" diye sor:

1. Proje iskeleti, Docker, compose, `.env.example`, Tailwind tasarım token'ları, temel UI bileşenleri (Button, Input, Card, Dialog, Sheet, Toast)
2. Veritabanı şeması, migration, seed, appointment.service (müsaitlik hesaplama, oluşturma, taşıma, swap, araya alma, iptal) + Vitest testleri
3. Kimlik doğrulama (personel şifre, müşteri OTP), rate limiting
4. Müşteri randevu akışı + randevularım
5. Personel takvimi, sürükle-bırak, walk-in, araya alma, SSE anlık güncelleme
6. Bildirimler: SMS adaptörleri, e-posta, web push, worker ve hatırlatma işleri, doluluk uyarıları
7. Yönetici paneli, toplu SMS, raporlar
8. PWA (manifest, service worker, kurulum yönlendirmesi), performans optimizasyonu, Playwright testleri, README

Kodu yazarken her zaman bu dokümandaki tasarım kurallarına, renk paletine ve performans hedeflerine uy. Bir kararın bu dokümanla çeliştiğini fark edersen dokümanı esas al.
