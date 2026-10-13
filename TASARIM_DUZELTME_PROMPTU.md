# Gemini Tasarım Düzeltme Promptu

> Gemini'de devam eden projeye aşağıdakilerin tamamını yapıştırın. Yanına `tasarim/` klasöründeki `style.css` dosyasını, HTML dosyalarını ve `ekran/` içindeki PNG görüntülerini ekleyin. Görüntüler en önemli kısım, mutlaka ekleyin.
>
> Bu prompt özellikleri değiştirmez, yalnızca görünümü baştan düzeltir.

---

Şu anki tasarım kabul edilmedi. Siyah-gold renkler, giriş ekranındaki saç/sakal kartları ve süslü düzen yüzünden hazır bir şablon gibi görünüyor. Ekteki ekran görüntüleri nasıl görünmesi gerektiğini gösteriyor. İş mantığına, API'lere ve veritabanına dokunma. Yalnızca arayüzü aşağıdaki kurallara göre baştan yaz.

Sırayla şunları yap:
1. Mevcut tüm renkleri, fontları ve tema ayarlarını sil. `globals.css` dosyasını ekteki `style.css` ile değiştir. `tailwind.config.ts` dosyasını aşağıdaki kodla değiştir.
2. Bütün bileşenleri ve sayfaları tek tek gez. Yasak listesindeki her şeyi kaldır.
3. Ekranları ekteki HTML dosyalarına ve görüntülere birebir benzet. Müşteri ilk ekranını `01-musteri-salon` gibi yap.
4. En sonda kontrol komutlarının çıktısını yaz. Çıktı boş olmalı.
5. Değişen her dosyanın tam içeriğini ver. Kısaltma yapma.

## TASARIM — KESİN KURALLAR

Bu bölüm önerilerden oluşmuyor; hepsi zorunlu. Amaç: uygulama apple.com/tr, bir banka uygulaması ya da iOS'un kendi Takvim ve Ayarlar ekranları gibi görünsün. "Yapay zekâyla yapılmış bir SaaS şablonu" gibi görünmesin. Emin olmadığın her durumda daha sade, daha az renkli, daha az süslü olanı seç.

### 0. Referans ekranlar — önce bunlara bak

Bu mesaja `tasarim/` klasöründen şu dosyalar eklendi:
- `style.css`
- 7 HTML ekran: `01-musteri-salon`, `02-musteri-hizmet`, `03-musteri-saat`, `04-musteri-onay`, `10-panel-giris`, `11-panel-takvim`, `12-panel-randevu`
- `ekran/` klasöründe bu ekranların görüntüleri

Uygulamanın görünümü **bu ekranların aynısı** olacak:
- `style.css` içindeki renkleri, ölçüleri ve sınıf yapısını aynen al. Next.js bileşenlerine çevirirken görünümü değiştirme.
- Burada olmayan bir ekranı (müşteriler, ayarlar, raporlar vb.) yaparken de aynı parçaları kullan: gri zemin, beyaz liste grupları, mavi metin bağlantıları, tek mavi buton.
- Kendi stil fikrini ekleme. Referansla senin fikrin çelişirse referans geçerli.

**Bu bir berber uygulaması, ama berber gibi görünmeyecek.** Bir banka ya da Apple uygulaması gibi görünecek. Berber sitelerinin klişelerini kesinlikle kullanma:
- Siyah zemin üstüne altın / gold / bronz / bakır renkler (`#D4AF37`, `#C9A227`, `#B8860B` ve benzerleri)
- Koyu tema varsayılanı (koyu tema yalnızca telefon koyu moddaysa açılır)
- Makas, ustura, bıyık, berber direği ikon veya illüstrasyonları
- Serif ya da "vintage" başlık fontları (Playfair, Cinzel vb.)
- Arka planda fotoğraf, tam ekran salon görseli, karartılmış fotoğraf üstüne yazı
- "Premium", "Erkek bakımında yeni dönem", "Tarzını yansıt" gibi sloganlar

Müşteri siteye girdiğinde ilk ekranda hizmet listesi, fiyatlar ya da "Saç / Sakal" kartları **görmeyecek**. İlk ekranda yalnızca şunlar olacak (bkz. `01-musteri-salon`):
- Salon adı ve adresi
- En yakın boş saat
- Tek bir "Randevu Al" butonu
- Telefon ve yol tarifi

Hizmetler ancak "Randevu Al"a basınca gelir.

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

- **Tailwind sınıfları:** `bg-gradient-`, `from-`, `via-`, `to-` (gradyan), `bg-clip-text`, `text-transparent`, `backdrop-blur`, `blur-`, `shadow-md`, `shadow-lg`, `shadow-xl`, `shadow-2xl`, `rounded-2xl`, `rounded-3xl`, `animate-bounce`, `animate-pulse` (iskelet yükleme hariç), `animate-ping`, `ring-` ile parlama, `hover:scale-`, `hover:-translate-y`
- **Renk adları:** purple, violet, indigo, fuchsia, pink, rose, cyan, teal, emerald, lime, sky, amber, orange, yellow. Bunlar config'te zaten olmayacak, inline style ile de yazma.
- **Emoji:** arayüzde, bildirim metinlerinde ve SMS şablonlarında hiç olmayacak.
- **İkonlar:** ✨ yıldız, sihirli değnek, roket, alev, taç, kupa yok. Renkli daire veya kare içinde ikon yok. İkon her zaman tek renk (`ink-2` ya da `accent`), 20px, 1.5 çizgi kalınlığında.
- **Düzen kalıpları:**
  - Ortalanmış dev başlık + altında iki buton olan "hero" bölümü
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
| Kart | Mümkünse kullanma. Gerekirse `bg-surface`, 1px `line-light` kenarlık, `rounded` (12px), gölge yok. |
| Alt sayfa (sheet) | Mobilde alttan açılır, üst köşeler 12px, tutamaç çizgisi 36×5px `line`. Tek gölge `shadow-sheet`. Arka plan karartması `rgba(0,0,0,0.4)`, blur yok. |
| Sekme çubuğu (mobil panel) | Altta 49px + safe-area. 4 sekme: Takvim, Müşteriler, Bildirimler, Ayarlar. Seçili ikon ve etiket `accent`, diğerleri `ink-3`. Etiket 10px. |
| Üst başlık | iOS büyük başlık: 34px/700, sola hizalı, kaydırınca 17px/600 ortalanmış başlığa küçülür. Arka plan düz `bg`, alt kenarda 0.5px `line`. |
| Durum etiketi | Hap rozet yok. Küçük 8px nokta + metin: `● Onaylandı`. Nokta durum renginde, metin `ink-2`. |
| Toast | Altta ortada, `bg-ink text-bg`, 14px, `rounded` (12px), 3 saniye. "Geri Al" bağlantısı `accent` değil, kalın beyaz metin. |
| Boşluklar | Yalnızca 4, 8, 12, 16, 20, 24, 32, 48. Sayfa yan boşluğu mobilde 16px, masaüstünde 24px. İçerik genişliği en fazla 980px. |
| Animasyon | Yalnızca opaklık ve konum, 200ms, `cubic-bezier(0.25, 0.1, 0.25, 1)`. Sayfa açılışında giriş animasyonu yok. |

### 4. Ekran ekran düzen

**Müşteri: salon sayfası (`/s/{slug}`)** — bkz. `01-musteri-salon.png`
```
┌──────────────────────────────┐
│                 Randevularım │  ← accent metin bağlantı
│ Kuzey Berber                 │  ← 34px/700
│ Moda Cad. No 18, Kadıköy     │  ← 15px ink-2
│ RANDEVU                      │  ← 13px ink-2 grup etiketi
│ ┌──────────────────────────┐ │
│ │ En yakın boş saat        │ │
│ │ Bugün 14:30              │ │
│ │ [      Randevu Al      ] │ │  ← tek birincil buton
│ └──────────────────────────┘ │
│ BİLGİ                        │
│ ┌──────────────────────────┐ │
│ │ Bugün açık   09:00–21:00 │ │
│ │ Telefon   0216 333 00 00 │ │  ← tel: bağlantısı
│ │ Yol tarifi             › │ │
│ └──────────────────────────┘ │
└──────────────────────────────┘
```
- Hero, kapak fotoğrafı, slogan, hizmet listesi ve fiyat yok. Hizmetler "Randevu Al"dan sonraki ekranda (`02-musteri-hizmet.png`).
- Hizmet ekranında seçili satırın solunda `accent` onay işareti. Satırın arka planı değişmez.
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
- Masaüstünde personel sütunları yan yana. Sütun başlığında yalnızca personel adı 15px/600. Avatar fotoğrafı yok, baş harf dairesi de yok.

**Personel: giriş**
- Ekranın ortasında, en fazla 360px genişlikte: salon logosu (yoksa salon adı 22px/600), 3 input (Salon kodu, Kullanıcı adı, Şifre), birincil "Giriş" butonu tam genişlikte.
- Arka plan düz `bg`. İllüstrasyon, desen veya fotoğraf yok.

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
grep -rnE "gradient|from-|via-|to-[a-z]|backdrop-blur|shadow-(md|lg|xl|2xl)|rounded-(2xl|3xl)|purple|violet|indigo|pink|fuchsia|emerald|cyan|teal" app src components
grep -rnP "[\x{1F300}-\x{1FAFF}\x{2728}]" app src components
```
