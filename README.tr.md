<img src="docs/banner.svg" alt="Jev Ultrafast · Browser Use × TypeSafe" width="100%" />

<p align="center"><a href="README.md">English</a> · Türkçe</p>

> [!NOTE]
> **Jev Ultrafast artık [Browser Use](https://github.com/browser-use/browser-use) içinde "ultrafast" modu olarak yer alıyor.** Fikir aynı, ama artık yerleşik: ajan, sayfadaki numaralandırılmış öğeler üzerinde işlem yapar. Browser Use ayrıca uzun, çok adımlı görevleri de yönetir.
>
> Python: `pip install browser-use`, ardından `Agent(task, mode='ultrafast')` (`bu-fast` ve `bu-ultrafast` modellerinde zaten varsayılan). Barındırılan sürüm: Browser Use Cloud'daki `bu-ultrafast` modeli.
>
> Bu repo, orijinal Browser Use × TypeSafe deneyi olarak yayında kalmaya devam ediyor.

# Jev Ultrafast ⚡

> [!IMPORTANT]
> **Browser Use Cloud'un kendi barındırdığı ajanı BU Ultrafast, tüm planlarda kullanıma açık.** Kullanmak için `model: "bu-ultrafast"` ayarla.
> **[Daha fazla bilgi](https://browser-use.com/ultrafast?utm_source=github&utm_medium=readme&utm_campaign=jev-ultrafast)** · **[Browser Use Cloud'u aç](https://cloud.browser-use.com?utm_source=github&utm_medium=readme&utm_campaign=jev-ultrafast)**

**Dinamik, indekslenmiş bir eylem uzayına sahip tarayıcı ajanı.**

Ona tek bir hedef ver. [TypeSafe'in Jev modeli](https://docs.typesafe.ai/introduction) bir işlem ve bir öğe seçer. Küçük bir LLM yalnızca işlem `TYPE_TEXT` olduğunda metin yazar.

**Google Flights'ta Zürih → Londra araması 7,1 saniyede.** Tek bir doğal dil hedefi; gerçek metin üretimi ve sayfa yükleme beklemeleri dahil.

<a href="docs/demo.mp4"><img src="docs/demo.gif" alt="1× hızda gerçek bir Google Flights araması; üretilen şehir adları ve dinamik işlem/hedef kararlarıyla" width="100%" /></a>

[MP4'ü izle](docs/demo.mp4) · [Ölçümler](docs/performance.md) · [Döngünün kodunu oku](jev_ultrafast/agent.py)

## Eylem uzayı

Her gözlem yeni bir öğe tablosu üretir:

```text
[1] button    Change ticket type · Round trip
[2] combobox  Where from?        · San Francisco
[3] combobox  Where to?          · empty
[4] textbox   Departure          · empty
...
```

İşlemler şunlardır: `CLICK`, `TYPE_TEXT`, `SELECT`, `SCROLL_UP`, `SCROLL_DOWN`, `WAIT`, `DONE` ve `BLOCKED`. Yalnızca desteklenen işlemler ve hedefler sunulur.

```text
                      tek TypeSafe isteği
                     ┌───────────────────────────┐
sayfa → öğe tablosu → işlem                      │
                     │ click_target              │
                     │ type_text_target          │
                     │ select_target, varsa      │
                     └─────────────┬─────────────┘
                       eşleşen hedefi kullan
                                   │
                    CLICK [7] ─────┤──→ tarayıcı
                TYPE_TEXT [3] ─────┘
                          ↓
                 küçük LLM → metin → tarayıcı
```

Hedef soruları spekülatif olarak sorulur. İşlem `CLICK` ise yalnızca `click_target` çalıştırılabilir. İki karar, **tek ağ gidiş-dönüşü**. Her hedef başlığı yalnızca uyumlu öğeleri içerir. Yerel açılır menü (dropdown) seçenekleri, gözlemlenen bir öğe/seçenek indeksi taşır.

Politikada siteye özel eylem betikleri ya da önceden hazırlanmış alan metinleri yoktur. Flights örneği yalnızca bir hedef verir ve sonucu bağımsız olarak doğrular. Ekran görüntüsü oluşturucu etiketleri sonradan ekler; tarayıcıyı o yönetmez.

## Orijinal demoyu dene

```bash
git clone https://github.com/browser-use/jev-ultrafast.git
cd jev-ultrafast
uv sync
cp .env.example .env
# TYPESAFE_API_KEY ve TEXT_MODEL_API_KEY değerlerini ekle.
uv run jev
```

**http://127.0.0.1:8766** adresini aç ve **Start demo → Run automatically** butonuna tıkla. Denetleyici (inspector); numaralandırılmış öğeleri, işlem olasılıklarını, hedef olasılıklarını ve çalıştırılan eylemleri gösterir. **Choose next**, çalıştırmadan önce duraklatır.

Chrome, `uv sync` ile kurulan [Browser Harness](https://github.com/browser-use/browser-harness) üzerinden bağlanır. Bağlantı gerekirse `uv run browser-harness --doctor` komutunu çalıştır. Chrome sorduğunda uzaktan hata ayıklamaya (remote debugging) izin ver.

Örnek yapılandırmada `TEXT_MODEL_API_KEY` bir OpenRouter anahtarıdır. Mevcut demo, akıl yürütmesi (reasoning) kapalı `inception/mercury-2.5` modelini kullanır. Gemini, GLM ve DeepSeek de OpenAI uyumlu metin yardımcısını kullanabilir; uygun modeli, uç noktayı (endpoint) ve akıl yürütme ayarını yapılandır.

## Orijinal kütüphaneyi kullan

```python
from jev_ultrafast import Agent

with Agent(
    "https://www.google.com/travel/flights?hl=en",
    "Find one-way flights from Zurich to London on September 20, 2026, "
    "for one adult in economy. Stop when matching flight options are visible.",
) as agent:
    for state in agent.run():
        print(state["elapsed_ms"], state["status"])
```

`uv run --env-file .env python your_script.py` ile çalıştır. Aynı politika farklı bir görevi de çalıştırabilir:

```bash
uv run --env-file .env python examples/run.py \
  --url https://en.wikipedia.org/wiki/Main_Page \
  --goal 'Find and open the Wikipedia article about Gödel’s incompleteness theorems.'
```

`uv run --env-file .env python examples/flights.py --keep-open` uçuş aramasını yapar, gerçek rota/tarih/sonuçları kontrol eder ve izini (trace) kaydeder. Uçuş seçmez ya da rezervasyon yapmaz.

## Neden bu kadar hızlı

- **Her karar döngüsünde tek istek.** İşlem ve hedef başlıkları aynı gözlemlenen durumu paylaşır.
- **Varsayılan ajan döngüsünde ekran görüntüsü yok.** Jev yapılandırılmış durumu kullanır. Ekran görüntüsünü yalnızca denetleyici ister; video ayrı, sürekli bir ekran kaydı kullanır.
- **Her anlık görüntü için tek tarayıcı çağrısı.** Görünür kontroller, adları, değerleri ve metinleri tek seferde (atomik olarak) okunur. Gerçek DOM düğümlerine referanslar tutulur.
- **Seçilen hedef doğrulanır.** Tıklamalar belgeyi, form değerlerini, hedefi ve yakın bağlamı kontrol eder. Sadece animasyon yüzünden yeni bir tahmin yapılmaz. Girdiden önce güncel geometri hesaplanır ve üstü kapalı kontroller reddedilir.
- **İşe yarar durum beklenir.** Bir combobox'a yazdıktan sonra görünür öneriler beklenir, en fazla 200 ms. Diğer etkileşimler en fazla iki animasyon karesi ya da 50 ms bekler. Bu okumalar, çalıştırma kaydedildikten sonra yapılır.
- **Gizli sekmeler render edilmeye devam eder.** Odak emülasyonu, Chrome'un görünür sekmesini değiştirmeden arka plan animasyonlarının yavaşlatılmasını önler.
- **Yalnızca görünür metin gönderilir.** Ekran dışındaki makale gövdeleri ve altbilgiler model bağlamını doldurmaz.
- **Yarıda kalan metin isteği yeniden kullanılır.** Üretilen bir değer, sayfa eskidiği için yapılan yeniden denemede yalnızca metin yardımcısının girdisi tamamen aynıysa korunur.

Çalıştırılan her hedef, gözlemlenmiş bir düğümden çözümlenir. Yürütücü, sayfanın güncelliğini ve tıklamanın engellenip engellenmediğini yeniden kontrol eder. Model çıktısı hiçbir zaman seçiciye (selector), koordinata, kabuk komutuna ya da çalıştırılabilir JavaScript'e dönüşmez. Metin yardımcısının çıktısı, yazılmadan önce küçük bir JSON nesnesi olarak ayrıştırılabilmelidir.

## Okunacak kadar küçük

| Dosya | Görevi |
| --- | --- |
| [agent.py](jev_ultrafast/agent.py) | Döngünün tamamı ve metin yardımcısına devir |
| [snapshot.js](jev_ultrafast/snapshot.js) | Atomik DOM anlık görüntüsü, indekslenmiş kontroller, güncellik korumaları |
| [browser.py](jev_ultrafast/browser.py) | Tarayıcı bağlantısı, güncel geometri, çalıştırma |
| [model.py](jev_ultrafast/model.py) | Dinamik işlem/hedef başlıkları ve metin üretimi |
| [questions.py](jev_ultrafast/questions.py) | Model talimatları |
| [demo.py](jev_ultrafast/demo.py) | Yerel denetleyici |

## Kanıtlar ve sınırlar

Mevcut video **7.073 ms** süren bir Google Flights çalıştırmasıdır. Süre ölçümü, ilk sayfa gözleminden sonra başlar ve model çağrılarını, üretilen metni, tarayıcı işlerini, eskimiş kararları ve yükleme beklemelerini içerir. Bağımsız ve taze bir kontrol; tek yön ayarını, Zürih'i, Londra'yı, 20 Eylül 2026 tarihini ve görünür uçuş seçeneklerini doğrular. Video 1× hızda oynar; başta bekleme yok, sonda 0,5 saniyelik bekleme var.

Aynı modeller ve ayarlarla yapılan, sırayla değişen altı çalıştırmada iki sürüm de **3/3** başarılı oldu. Ortanca görev süresi **9,450 sn → 7,092 sn**'ye indi, yani **%25 azalma**; ortanca tarayıcı protokol çağrısı **1.092 → 101**'e düştü. Bu, tek bir tarayıcı profilinde tek bir görevin üç tekrarıdır; genel bir güvenilirlik ölçütü değildir.

Aynı politika, istenen Wikipedia makalesini **2,798 sn**'de açtı ve yerel bir otel arama/filtreleme görevini **1,896 sn**'de geçti. Çalıştırmalar, başarısızlıklar, kaynak hash'leri ve ölçüm sınırları [performance.md](docs/performance.md) dosyasında.

`DONE` seçimi yine de sonucun bağımsız olarak doğrulanmasını gerektirir. DOM okuyucu yaygın HTML ve ARIA kontrollerini işler, erişilebilir ad (accessible name) spesifikasyonunun tamamını değil. Shadow root'lar, frame'ler, canvas, dosya yüklemeleri, açılır pencere sekmeleri, iç içe kaydırma ve rastgele klavye bileşenleri bu MVP'nin kapsamı dışında. Ajanın açtığı sekmeler mevcut Chrome profilini paylaşır.

## Geliştirme

```bash
uv run ruff check .
uv run pytest
node --check jev_ultrafast/static/app.js
node --check jev_ultrafast/snapshot.js
uv build
```

Testler çevrimdışı çalışır. `uv run python scripts/check_guards.py`, model çağrısı yapmadan yerel bir tarayıcıda gerçek kontrolleri test eder. Canlı örnekler ve kayıt betikleri ücretli API çağrıları yapar. `scripts/record_flights.py <yeni-klasör>` orijinal tarayıcı zaman damgalarını kaydeder; `scripts/render_demo.py <kayıt-klasörü>` doğrulanmış çalıştırmayı 1× hızda render eder ve Google hesap şeridini kırpar. Kimlik bilgileri ve ham izler git tarafından yok sayılır.

---

[Browser Use](https://github.com/browser-use/browser-use) · [Browser Harness](https://github.com/browser-use/browser-harness) · [TypeSafe spekülatif fan-out](https://docs.typesafe.ai/patterns/fan-out)
