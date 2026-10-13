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

## TASARIM SİSTEMİ — Kurumsal, Sade, Apple Tarzı

**İlham:** apple.com/tr — bol beyaz alan, net tipografi, az renk, içeriği öne çıkaran düzen. Hedef his: güvenilir, sakin, profesyonel.

**KESİNLİKLE YAPMA (yapay zekâ / "AI ürünü" görünümü):**
- Mor, pembe, neon, turkuaz tonları; mor-mavi **gradyanlar**; parlayan (glow) efektler
- Glassmorphism, bulanık arka plan blob'ları, parıltı/yıldız ✨ ikonları
- Arayüzde emoji kullanımı
- Her köşede aşırı yuvarlak "baloncuk" kartlar, abartılı gölgeler
- Gereksiz animasyon; zıplayan, dönen öğeler
- "Yapay zekâ destekli", "akıllı", "sihirli" gibi pazarlama dili

**Renk paleti (Tailwind tema token'ı olarak tanımla, başka renk kullanma):**

| Token | Açık tema | Koyu tema | Kullanım |
|---|---|---|---|
| `bg` | `#FFFFFF` | `#000000` | Sayfa zemini |
| `bg-subtle` | `#F5F5F7` | `#1C1C1E` | Bölüm zemini, takvim ızgarası |
| `surface` | `#FFFFFF` | `#1C1C1E` | Kart, panel |
| `border` | `#D2D2D7` | `#38383A` | Ayırıcı çizgiler |
| `text` | `#1D1D1F` | `#F5F5F7` | Ana metin |
| `text-secondary` | `#6E6E73` | `#A1A1A6` | Açıklama, etiket |
| `text-tertiary` | `#86868B` | `#8E8E93` | Pasif, ipucu |
| `accent` | `#0071E3` | `#2997FF` | Birincil buton, bağlantı, seçili öğe |
| `accent-hover` | `#0077ED` | `#47A8FF` | Hover |
| `success` | `#1D7A3A` | `#30D158` | Onaylandı, tamamlandı |
| `warning` | `#B25000` | `#FF9F0A` | Yaklaşan, dikkat |
| `danger` | `#D70015` | `#FF453A` | İptal, hata, şu anki saat çizgisi |

- Renk tek başına anlam taşımasın: durumlar renk + simge + metin ile gösterilsin.
- Personel renk kodları da yalnızca bu düşük doygunluklu kurumsal setten: `#0071E3`, `#1D7A3A`, `#B25000`, `#6E6E73`, `#00627A`, `#8A5A00` — kartta sadece ince sol kenar çizgisi olarak.
- Tüm metin/zemin kombinasyonları **WCAG AA** (≥ 4.5:1) kontrastını sağlamalı.
- Koyu tema `prefers-color-scheme` ile otomatik, ayarlardan manuel seçilebilir.

**Tipografi:**
- Font yığını: `-apple-system, BlinkMacSystemFont, "SF Pro Text", "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif`
- Ölçek: 12 / 14 / 17 (gövde) / 21 / 28 / 40 px. Başlıklar 600 ağırlık, gövde 400. Satır yüksekliği gövdede 1.47.
- Saat ve sayılarda `font-variant-numeric: tabular-nums`.

**Bileşen kuralları:**
- Köşe yarıçapı: butonlar **980px (hap)** veya 12px, kartlar 18px, input'lar 12px. Tutarlı ol.
- Gölge neredeyse yok; ayrımı `border` ve `bg-subtle` ile yap. Gerekirse tek gölge: `0 2px 12px rgba(0,0,0,0.06)`.
- Birincil buton: `accent` zemin, beyaz metin. İkincil: şeffaf zemin, `accent` metin. Tehlikeli: `danger` metin, onay diyaloğu ile.
- Boşluk ölçeği: 4'ün katları (4, 8, 12, 16, 24, 32, 48, 64).
- Geçişler: 150–200 ms `ease-out`, yalnızca opaklık/konum. `prefers-reduced-motion`'a uy.
- Boş durumlar: tek satır net açıklama + tek eylem butonu ("Bugün randevu yok. Plansız müşteri ekleyin.").
- Hata mesajları: ne oldu + ne yapmalı, sade Türkçe ("Bu saat dolu. Lütfen başka bir saat seçin.").

**Anlaşılırlık:**
- Her ekranda tek birincil eylem.
- İkon tek başına kalmasın; önemli eylemlerde metin etiketi olsun.
- 50+ yaşındaki bir berberin eğitim almadan kullanabileceği kadar basit: büyük dokunma alanları, kısa menüler, teknik terim yok.

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
