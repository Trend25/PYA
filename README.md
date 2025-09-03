# PYA
proje-yonetimi-arastirmasi
# README — Proje Hikayeleri Anketi → Google Sheets Entegrasyonu

Bu doküman, **anket.projegercegi.com** sayfasındaki formun verilerini **Google Sheets**’e kaydetmek için gereken tüm adımları ve kodları içerir. Kurulum 10–15 dk sürer.

---

## 1) Mimari Özet

* **Frontend (HTML/JS)** form verilerini **Google Apps Script Web App**’ine POST eder.
* **Apps Script** verileri alır ve belirlediğiniz **Google Sheet**’e yeni satır olarak yazar.
* Gönderim formatı **URL-encoded** (önerilen). Script aynı zamanda JSON gövdeyi de destekler.

---

## 2) Google Sheet’i Hazırla

1. Yeni bir Google Sheet oluşturun.
2. Adres çubuğundan **Spreadsheet ID**’yi kopyalayın:
   `https://docs.google.com/spreadsheets/d/<SPREADSHEET_ID>/edit`
3. Verileri yazmak istediğiniz sekmenin adını not alın (örn. `Anket`). Boş bırakırsanız **ilk sayfa** kullanılır.

---

## 3) Apps Script (Sunucu) — Kodu Yapıştır ve Dağıt

### 3.1 Kod (tamamını yapıştır)

```javascript
/**
 * Google Sheets Web App – SAĞLAM
 * - JSON (application/json veya text/plain gövde) + URL-encoded destekler
 * - SPREADSHEET_ID ile yazar
 * - İlk satıra başlıkları basar
 */

const SPREADSHEET_ID = 'BURAYA_SHEET_ID';     // Örn: 1Amh...Zbjs
const SHEET_NAME     = '';                    // Örn: 'Anket' (boşsa ilk sayfa)

const HEADERS = [
  'submission_date',
  'name',
  'email',
  'experience',
  'sector',
  'project_name',
  'success_story',
  'human_challenge',
  'operational_challenge',
  'biggest_crisis',
  'total_quality_score',
  'section_scores',
  'completion_percentage',
  'earned_badges',
  'version'
];

function getSheet() {
  const ss = SpreadsheetApp.openById(SPREADSHEET_ID);
  return SHEET_NAME ? (ss.getSheetByName(SHEET_NAME) || ss.getSheets()[0]) : ss.getSheets()[0];
}

function ensureHeaders(sheet) {
  const range = sheet.getRange(1, 1, 1, HEADERS.length);
  const current = range.getValues()[0] || [];
  const need = current.length < HEADERS.length || HEADERS.some((h, i) => current[i] !== h);
  if (need) range.setValues([HEADERS]);
}

function parseBody(e) {
  const post = (e && e.postData) || {};
  const type = post.type || '';
  const contents = (post.contents || '').trim();

  // JSON başlığı
  if (type.indexOf('application/json') !== -1) return JSON.parse(contents || '{}');

  // no-cors altında text/plain gelebilir; JSON gibi görünüyorsa yine parse et
  if (contents.startsWith('{') || contents.startsWith('[')) {
    try { return JSON.parse(contents); } catch (_) {}
  }

  // URL-encoded
  return Object.assign({}, e.parameter);
}

function doPost(e) {
  try {
    const data = parseBody(e);
    const sheet = getSheet();
    ensureHeaders(sheet);

    const row = HEADERS.map(k => (k in data ? String(data[k]) : ''));
    sheet.appendRow(row);

    return ContentService
      .createTextOutput(JSON.stringify({ status: 'success', saved: true }))
      .setMimeType(ContentService.MimeType.JSON);

  } catch (err) {
    console.error('doPost error:', err);
    return ContentService
      .createTextOutput(JSON.stringify({ status: 'error', message: String(err) }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function doGet(e) {
  return ContentService
    .createTextOutput(JSON.stringify({ status: 'ok', headers: HEADERS }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### 3.2 Değiştirmeniz gerekenler

* `SPREADSHEET_ID`: Kendi Sheet ID’niz.
* `SHEET_NAME`: (opsiyonel) Sekme adı.

### 3.3 Web App olarak dağıt

1. **Deploy → Manage deployments → New deployment**
2. **Web app** seçin.
3. **Execute as (Şu kullanıcı olarak çalıştır):** **Me / Ben**
4. **Who has access (Erişimi olanlar):** **Anyone / Herkes**
5. **Deploy** → çıkan **/exec** URL’sini kopyalayın.

> Sağlık kontrolü: `/exec` URL’sini tarayıcıda açınca `{"status":"ok"...}` görmelisiniz.

---

## 4) Frontend (İstemci) — Gönderim

Form verilerini `responses` adlı bir nesnede topladıktan sonra **URL-encoded** olarak gönderin:

```javascript
const GOOGLE_SCRIPT_URL = 'https://script.google.com/macros/s/XXX/exec'; // Kendi /exec URL’iniz

// responses: formdan topladığınız alanlar
// Zorunlu/temel alanlar:
// name, email, experience, sector, project_name,
// success_story, human_challenge, operational_challenge, biggest_crisis,
// submission_date, total_quality_score, section_scores, completion_percentage, earned_badges, version

const params = new URLSearchParams();
Object.entries(responses).forEach(([k, v]) => params.append(k, String(v ?? '')));

fetch(GOOGLE_SCRIPT_URL, {
  method: 'POST',
  mode: 'no-cors',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded;charset=UTF-8' },
  body: params.toString()
});
```

> **Not:** “no-cors” ile tarayıcıdan “başarılı” mesajını göremezsiniz; bu normaldir. Doğrulamayı Sheet’te yeni satır oluşmasıyla yapın.

---

## 5) Sütunlar (HEADERS) ve Anlamları

| Alan                    | Açıklama                                                    |
| ----------------------- | ----------------------------------------------------------- |
| `submission_date`       | Gönderim tarihi (örn. `new Date().toLocaleString('tr-TR')`) |
| `name`, `email`         | Kullanıcı bilgileri                                         |
| `experience`, `sector`  | Profil bilgileri                                            |
| `project_name`          | Projenin adı                                                |
| `success_story`         | Başarı hikayesi                                             |
| `human_challenge`       | İnsan dinamikleri                                           |
| `operational_challenge` | Operasyonel zorluk                                          |
| `biggest_crisis`        | En büyük kriz                                               |
| `total_quality_score`   | Toplam kalite puanı (0–100)                                 |
| `section_scores`        | Bölüm puanları (JSON string)                                |
| `completion_percentage` | Form tamamlanma yüzdesi                                     |
| `earned_badges`         | Kazanılan rozet sayısı                                      |
| `version`               | İstemci sürüm etiketi (örn. `gamified_v3.0_fixed`)          |

> **Yeni sütun eklemek** isterseniz: `HEADERS` listesine ekleyin → Script’i **kaydedip yeniden dağıtın** → Sheet ilk satırına otomatik başlık basılır.

---

## 6) Content Security Policy (CSP)

Sitenizde CSP kısıtı varsa **Apps Script’e bağlantıyı** izinli kılın. `connect-src`’a şu domainleri ekleyin:

```
https://script.google.com https://script.googleusercontent.com
```

### Örnek `<meta>` (statik sitede):

```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               connect-src 'self' https://script.google.com https://script.googleusercontent.com;">
```

### Örnek `next.config.js` (Next.js / Vercel):

```js
// next.config.js
module.exports = {
  async headers() {
    return [{
      source: '/(.*)',
      headers: [{
        key: 'Content-Security-Policy',
        value: "default-src 'self'; connect-src 'self' https://script.google.com https://script.googleusercontent.com"
      }]
    }];
  }
};
```

---

## 7) Hızlı Test

1. `/exec` URL’sini tarayıcıda aç → `status":"ok"` beklenir.
2. Kendi sitenizde (Google ana sayfasında değil) DevTools Console’da:

```js
fetch('https://script.google.com/macros/s/XXX/exec', {
  method:'POST',
  mode:'no-cors',
  headers:{'Content-Type':'application/x-www-form-urlencoded;charset=UTF-8'},
  body:new URLSearchParams({
    submission_date: new Date().toISOString(),
    name: 'Test', email:'t@t.com', experience:'3-5', sector:'web',
    project_name:'Deneme',
    success_story:'başarı', human_challenge:'insan', operational_challenge:'ops', biggest_crisis:'kriz',
    total_quality_score:'80',
    section_scores:'{"success":20,"human":20,"ops":20,"crisis":20}',
    completion_percentage:'100', earned_badges:'4', version:'gamified_v3.0_fixed'
  })
});
```

3. Sheet’te yeni satır oluştuysa entegrasyon tamamdır.

---

## 8) Sorun Giderme (FAQ)

* **Sheet’e yazmıyor**

  * Deploy ayarları: **Execute as = Me**, **Access = Anyone** olmalı.
  * `SPREADSHEET_ID` yanlış olabilir.
  * Yeni kodu dağıtmamış olabilirsiniz (Manage deployments → Edit → Save & Deploy).
  * Sitenizin **CSP**’sinde `connect-src` izinleri yoksa tarayıcı isteği engeller.

* **Google ana sayfasından fetch deneyince hata**

  * Google’ın CSP’si dış isteğe izin vermez. **Kendi sitenizden** test edin.

* **Yeni sütun ekledim ama görünmüyor**

  * Apps Script’te `HEADERS` listesine ekleyin → Kaydet → **yeniden dağıtın** → ilk satır başlıklar güncellenecek.

* **Yanıt göremiyorum**

  * `mode: 'no-cors'` ile beklenen bir durum. Başarıyı Sheet’te kontrol edin.

---

## 9) Güvenlik ve KVKK Notu

* Script “Execute as: **Ben**” çalışır; yazma izni sizin hesabınız üzerinden yapılır.
* Kişisel verilerin kullanım kapsamı KVKK metninde belirtilmiştir; gerekirse anonimleştirilir.
* Silme talepleri için Sheet üzerinde manuel temizlik yapabilirsiniz.

---

## 10) İletişim

* Soru/iyileştirme: **[anket@projegercegi.com](mailto:anket@projegercegi.com)**
* İstemci sürümü: `gamified_v3.0_fixed`

---

Hazır! Bu metni doğrudan **README.md** dosyanıza koyabilirsiniz.
