---
name: xuat-pdf
description: >
  Dùng skill này khi người dùng muốn đọc nội dung từ file PDF (đặc biệt là sách Kinh Thánh
  hoặc sách cũ dạng scan) rồi xuất ra file .docx có định dạng chuyên nghiệp.
  Trigger khi người dùng nói: "đọc file PDF rồi xuất ra docx", "xuất PDF ra Word",
  "đọc nội dung PDF và tạo file docx", "xuat-pdf", hoặc upload file .pdf và yêu cầu
  tạo file Word/docx từ nó. Skill này áp dụng đặc biệt tốt cho sách scan nhiều cột
  (như Cassell's Bible 1860) nhưng có thể dùng cho bất kỳ PDF nào.
---

# Skill: Xuất PDF ra DOCX

## Luật số 1 — NGUYÊN VĂN TUYỆT ĐỐI

> Mọi nội dung trong DOCX — summary, verse, commentary — phải là chuỗi ký tự được **copy
> trực tiếp từ OCR text**. Không có ngoại lệ.

**Điều này có nghĩa là:**
- Trước khi viết bất kỳ dòng `chHead()`, `v()`, `com()`, `cp()` nào, AI phải đã **đọc
  file OCR tương ứng** và có chuỗi text gốc trong tay.
- Không được dùng trí nhớ, không được suy luận, không được tóm tắt, không được
  paraphrase, không được "điền vào" những gì có vẻ hợp lý.
- Nếu OCR bị mờ hay thiếu một đoạn: để trống hoặc ghi `[illegible]`, không được tự bịa.

### Ba loại nội dung — ba nguồn OCR tương ứng

| Nội dung | Nguồn trong OCR | Hàm JS |
|----------|-----------------|--------|
| **Chapter summary** | Dòng bắt đầu bằng số `1 ...` ngay sau `CHAPTER X.` lần đầu, trước verse 1 | `chHead(num, summary)` |
| **Verse text** | Cột trái/phải của trang, các dòng bắt đầu bằng số verse | `v(num, text)` / `vmi(num, ...)` |
| **Commentary** | Toàn bộ text sau `CHAPTER X.` lần thứ hai trong cùng file | `com(ref, text)` / `cp(text)` |

---

## Bước 1 — Kiểm tra và giải nén file

```bash
file /mnt/project/<tên-file>.pdf
# Nếu là ZIP (Cassell's):
unzip /mnt/project/<tên-file>.pdf -d /tmp/book/
ls /tmp/book/*.txt | wc -l   # đếm số trang
```

---

## Bước 2 — ĐỌC OCR TRƯỚC, VIẾT CODE SAU

**Quy trình bắt buộc cho mỗi chương:**

### 2A. Đọc và trích xuất summary

Summary nằm giữa `CHAPTER X.` lần đầu và dòng đầu tiên của verse (thường là chữ cái
to viết hoa đơn lẻ như `N`, `A`, `T`, `O`...).

```bash
# Đọc toàn bộ trang đầu của chương
cat /tmp/book/PAGE.txt

# Trích phần summary: từ sau "CHAPTER X." đến trước verse 1
# Summary nhận dạng: dòng bắt đầu bằng chữ số "1 ..." (không phải footnote)
```

**Ví dụ thực tế — I Kings Ch. I, trang 1:**
```
CHAPTER I.
BEFORE CHRIST 1015.
1 Heb entered into days          ← BỎ (footnote cột giữa)
2 Heb. Let them seek.            ← BỎ (footnote cột giữa)
1 Abishag cherisheth David in his extreme age. 5 Adonijah, David's darling,
usurpeth the kingdom. 11 By the counsel of Nathan, 15 Bath-sheba moveth the king,
...
OW king David was old...         ← bắt đầu verse 1 (chữ N bị mất)
```
→ Summary là: `1 Abishag cherisheth David in his extreme age. 5 Adonijah...`

**Cách phân biệt summary với footnote:**
- Footnote: `1 Heb. xxx`, `2 Or, xxx`, `a Gen. xv. 4` — **BỎ**
- Summary: `1 Tên người động từ...` (câu hoàn chỉnh có nghĩa) — **GIỮ**

### 2B. Đọc và trích xuất verse text

Verse text là các dòng có số đầu dòng trong cột trái/phải, **không phải** cột giữa.

```bash
# Đọc từng trang của chương
cat /tmp/book/PAGE.txt
cat /tmp/book/PAGE+1.txt
# ...
```

**Nhận dạng verse:** dòng bắt đầu bằng số (`2 Wherefore his servants...`).

**Lọc bỏ cross-refs cột giữa:** các dòng dạng `a Gen. xv. 4;` / `1 Heb. xxx` / `2 Sam. iii.` — **BỎ**.

**Verse thường bị ngắt dòng trong OCR** — nối lại thành câu hoàn chỉnh:
```
2 Wherefore his servants said unto
him, Let there be sought...
→ "Wherefore his servants said unto him, Let there be sought..."
```

### 2C. Đọc và trích xuất commentary

Commentary bắt đầu ngay sau `CHAPTER X.` **lần thứ hai** trong file OCR.

```bash
# Lệnh chuẩn để trích commentary:
awk '/^CHAPTER [IVX]+\./{n++; if(n==2){found=1; next}} found{print}' /tmp/book/PAGE.txt \
  | grep -v "^[0-9]\{2,3\}$"   # bỏ số trang
```

Commentary thường **tràn sang trang tiếp theo** — phải đọc thêm:
```bash
# Trang kế tiếp: nếu không có "CHAPTER X." lần 2 → đây là commentary tiếp
cat /tmp/book/PAGE+1.txt
```

**Nhận dạng các đoạn commentary:**
- `1. Text...` hoặc `1-3. Text...` → `com('1.', 'Text...')` / `com('1–3.', 'Text...')`
- Đoạn mở đầu không có số tham chiếu → `cp('Text...')`
- Nhiều dòng liên tiếp cùng một đoạn → **nối lại thành 1 chuỗi**

**Lọc bỏ trong commentary:**
- Số trang đơn lẻ (`533`, `534`) — **BỎ**
- Running header (`Rebellion of Moab. II. KINGS, I.`) — **BỎ**
- Cross-refs lẫn vào (`a 2 Sam. viii. 2;`) — **BỎ**

---

## Bước 3 — Viết code từ OCR đã đọc

Chỉ sau khi đã đọc xong OCR của một chương mới được viết code cho chương đó.

### ❌ SAI — Viết từ trí nhớ / tự suy luận

```javascript
// Summary tự nghĩ:
chHead('I', 'Abishag cherisheth David. Adonijah usurpeth the kingdom.')

// Verse tự điền:
v(1, 'Now king David was old and stricken in years.')

// Commentary tự tóm tắt:
com('1.', 'The Moabites revolted. Ahaziah was injured and could not respond.')
```

### ✅ ĐÚNG — Copy từ OCR đã đọc

```javascript
// Summary — copy nguyên văn từ /tmp/iking/1.txt:
chHead('I', '1 Abishag cherisheth David in his extreme age. 5 Adonijah, David\'s darling, usurpeth the kingdom. 11 By the counsel of Nathan, 15 Bath-sheba moveth the king, 22 and Nathan secondeth her. 28 David reneweth his oath to Bath-sheba. 32 Solomon, by David\'s appointment, being anointed king by Zadok and Nathan, the people triumph. 41 Jonathan bringing these news, Adonijah\'s guests fly. 50 Adonijah, flying to the horns of the altar, upon his good behaviour is dismissed by Solomon.')

// Verse — copy nguyên văn từ OCR:
vmi(1, 'OW king David was old and stricken in years; and they covered him with clothes, but he gat no heat.')
v(2, 'Wherefore his servants said unto him, Let there be sought for my lord the king a young virgin: and let her stand before the king, and let her cherish him, and let her lie in thy bosom, that my lord the king may get heat.')

// Commentary — copy nguyên văn từ /tmp/iiking/1.txt:
com('1.', 'The previous Book concluded with the intimation that Ahaziah succeeded to the throne of Ahab his father, and that his reign was characterised by the exhibition of the same godless and profligate spirit. The Moabites, who had been subjected by David (2 Sam. viii. 2 ; xxiii. 20), and who, in the partition of the kingdom, had remained tributary to the kingdom of the Ten Tribes, revolted after the death of Ahab, and endeavoured to sever themselves from Israel.')
com('2.', 'Ekron was the most northern of the five cities of the Philistines, situated between Ashdod and Samaria, bordering on Judah in the low lands. Baal-zebub was the tutelary deity against vermin.')
```

---

## Bảng lọc nội dung — GIỮ / BỎ

| Vùng trong OCR | Hành động | Nhận dạng |
|----------------|-----------|-----------|
| Running header đầu trang | 🟢 GIỮ → `runHdr()` | Dòng đầu tiên mỗi trang |
| Chapter heading `CHAPTER X.` lần 1 | 🟢 GIỮ → `chHead()` | Dòng `CHAPTER` đầu tiên |
| Summary (dòng sau CHAPTER, trước verse 1) | 🟢 GIỮ NGUYÊN VĂN | Bắt đầu bằng `1 Tên...` |
| Verse text cột trái/phải | 🟢 GIỮ NGUYÊN VĂN | Số đầu dòng + câu hoàn chỉnh |
| Image captions | 🟢 GIỮ → `cap()` | Chữ hoa hoàn toàn, mô tả hình |
| Commentary sau CHAPTER lần 2 | 🟢 GIỮ NGUYÊN VĂN | `N.` hoặc `N–M.` + text dài |
| Cross-refs cột giữa | 🔴 BỎ | `a Gen. xv.`, `1 Heb.`, `Or,` |
| Số trang | 🔴 BỎ | Số đơn lẻ 2–3 chữ số |
| `BEFORE CHRIST XXXX` | 🔴 BỎ | Trừ title page |
| Dòng `i`, `!`, `e` đơn lẻ | 🔴 BỎ | Artifact của OCR |

---

## Helper functions chuẩn

```javascript
'use strict';
const {Document, Packer, Paragraph, TextRun, AlignmentType,
       BorderStyle, PageNumber, Footer} = require('docx');
const fs = require('fs');
const F = 'Georgia';

const hRule = () => new Paragraph({
  spacing: {before: 200, after: 200},
  border: {bottom: {style: BorderStyle.SINGLE, size: 4, color: '999999'}},
  children: [new TextRun('')]
});

const runHdr = (l, c, r) => new Paragraph({
  alignment: AlignmentType.CENTER,
  spacing: {before: 0, after: 80},
  children: [
    new TextRun({text: l, italics: true, size: 18, font: F}),
    new TextRun({text: `    ${c}    `, bold: true, size: 18, font: F}),
    new TextRun({text: r, italics: true, size: 18, font: F})
  ]
});

// summary = NGUYÊN VĂN từ OCR, kể cả tất cả các số
const chHead = (num, summary) => [
  new Paragraph({
    alignment: AlignmentType.CENTER,
    spacing: {before: 480, after: 80},
    children: [new TextRun({text: `CHAPTER ${num}.`, bold: true, size: 30, font: F})]
  }),
  new Paragraph({
    alignment: AlignmentType.CENTER,
    spacing: {before: 0, after: 220},
    children: [new TextRun({text: summary, italics: true, size: 20, font: F})]
  })
];

// text = NGUYÊN VĂN từ OCR, nối các dòng bị ngắt
const v = (num, text) => new Paragraph({
  alignment: AlignmentType.JUSTIFIED,
  spacing: {before: 80, after: 80},
  indent: {hanging: 380, left: 380},
  children: [
    new TextRun({text: `${num} `, bold: true, size: 22, font: F}),
    new TextRun({text, size: 22, font: F})
  ]
});

// Verse mở đầu chương có drop cap (chữ cái to bị OCR tách ra)
const vmi = (num, ...parts) => {
  const runs = [new TextRun({text: `${num} `, bold: true, size: 22, font: F})];
  for (let i = 0; i < parts.length; i++)
    if (parts[i]) runs.push(new TextRun({text: parts[i], size: 22, font: F, italics: i % 2 === 1}));
  return new Paragraph({
    alignment: AlignmentType.JUSTIFIED,
    spacing: {before: 80, after: 80},
    indent: {hanging: 380, left: 380},
    children: runs
  });
};

// ref và text = NGUYÊN VĂN từ OCR
const com = (ref, text) => new Paragraph({
  alignment: AlignmentType.JUSTIFIED,
  spacing: {before: 60, after: 60},
  indent: {hanging: 360, left: 360},
  children: [
    new TextRun({text: `${ref}  `, bold: true, size: 20, font: F}),
    new TextRun({text, size: 20, font: F})
  ]
});

// Đoạn commentary không có số ref — text = NGUYÊN VĂN từ OCR
const cp = t => new Paragraph({
  alignment: AlignmentType.JUSTIFIED,
  spacing: {before: 80, after: 80},
  children: [new TextRun({text: t, size: 20, font: F})]
});

const ch = t => new Paragraph({
  alignment: AlignmentType.CENTER,
  spacing: {before: 320, after: 120},
  border: {top: {style: BorderStyle.SINGLE, size: 4, color: '888888'}},
  children: [new TextRun({text: t, bold: true, size: 22, font: F})]
});

const cap = t => new Paragraph({
  alignment: AlignmentType.CENTER,
  spacing: {before: 80, after: 200},
  children: [new TextRun({text: t, bold: true, size: 19, font: F})]
});
```

---

## Cấu trúc script và build

Vì file thường lớn (20+ chương), **chia script thành nhiều file** rồi `require()` nối nhau.
Mỗi file tối đa ~80,000 bytes.

```
book_p1.js  ← helpers + C[] + chapters I–VIII   → exports {C, helpers...}
book_p2.js  ← require('./book_p1') + IX–XVI     → exports base
book_p3.js  ← require('./book_p2') + XVII–XXV + BUILD
```

**File cuối — phần build:**

```javascript
const doc = new Document({
  styles: {default: {document: {run: {font: F, size: 22}}}},
  sections: [{
    properties: {
      page: {
        size: {width: 12240, height: 15840},
        margin: {top: 1440, right: 1440, bottom: 1296, left: 1440}
      }
    },
    footers: {
      default: new Footer({
        children: [new Paragraph({
          alignment: AlignmentType.CENTER,
          children: [
            new TextRun({text: 'Cassell\'s Illustrated Family Bible (1860) — BOOK  |  p.', size: 18, font: F, color: '888888'}),
            new TextRun({children: [PageNumber.CURRENT], size: 18, font: F, color: '888888'})
          ]
        })]
      })
    },
    children: C
  }]
});

Packer.toBuffer(doc).then(buf => {
  fs.writeFileSync('/mnt/user-data/outputs/Cassells_1860_BOOK_full.docx', buf);
  console.log('Done! Paragraphs:', C.length);
});
```

```bash
node /home/claude/book_p3.js
python3 /mnt/skills/public/docx/scripts/office/validate.py \
  /mnt/user-data/outputs/Cassells_1860_BOOK_full.docx
```

---

## Cấu trúc output chuẩn mỗi chương

```
hRule()                              ← phân cách (bỏ cho chương I)
chHead(num, 'NGUYÊN VĂN summary')   ← đọc từ OCR
runHdr(left, center, right)          ← đọc từ OCR
cap('CAPTION HOA')                   ← nếu có hình
vmi(1, 'Verse mở đầu...')           ← OCR, drop cap
v(2, 'Verse text...')                ← OCR
v(3, '...')
...
ch('CHAPTER X.')                     ← tiêu đề commentary
cp('Đoạn mở đầu không ref...')      ← OCR
com('1.', 'NGUYÊN VĂN...')          ← OCR
com('2.', 'NGUYÊN VĂN...')          ← OCR
com('1–3.', 'NGUYÊN VĂN...')        ← OCR
```

---

## Quy tắc kỹ thuật

1. **Script tối đa ~80,000 bytes** — chia nhỏ nếu cần
2. **Proper names KHÔNG dịch** — Solomon, Elijah, Samaria...
3. **Verse 1 dùng `vmi()`** — OCR thường tách drop cap thành ký tự đơn
4. **Validate PASSED** trước khi `present_files`
5. **Bỏ số trang** (`489`, `533`...) khỏi mọi nội dung

## Xử lý lỗi thường gặp

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `Command argument exceeds 100,000 bytes` | Script quá lớn | Dùng `create_file` rồi `node file.js` |
| Summary thiếu số | Không đọc OCR, tự viết lại | Chạy `cat /tmp/book/PAGE.txt`, copy nguyên văn |
| Commentary sai hoàn toàn | Tự tóm tắt thay vì đọc OCR | Chạy `awk` trích commentary, copy nguyên văn |
| Verse bị thiếu/sai | Dùng trí nhớ | Đọc lại file txt từng trang, copy từng câu |
| Text bị lẫn footnotes | OCR trộn cột giữa vào | Lọc các dòng dạng `a Tên.sách.` và `1 Heb.` |
| `Paragraphs: 0` | Chưa copy file vào outputs | Kiểm tra đường dẫn writeFileSync |

## Thực tế đã làm

| File | Chương | Paragraphs | Output |
|------|--------|-----------|--------|
| I Kings (Vol.2) | 22 | 562 | `Cassells_1860_I_KINGS_full.docx` |
| II Kings (Vol.2) | 25 | 499 | `Cassells_1860_II_KINGS_full.docx` |
