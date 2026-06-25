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

## Tổng quan

Skill này hướng dẫn cách đọc nội dung từ file PDF (kể cả PDF scan nhiều cột) và xuất ra
file `.docx` với định dạng đẹp, nhất quán — font Georgia, có running headers, verse numbers
đậm, commentary thụt lề, captions hình, footer đánh số trang.

---

## Bước 1 — Kiểm tra file PDF

```bash
# Kiểm tra loại file
file /mnt/project/<tên-file>.pdf
ls -lh /mnt/project/<tên-file>.pdf

# Rasterize toàn bộ trang thành ảnh JPEG 150dpi
pdftoppm -jpeg -r 150 /mnt/project/<tên-file>.pdf /tmp/pages_p

# Đếm số trang
ls /tmp/pages_p*.jpg | wc -l
```

---

## Bước 2 — Đọc nội dung từng trang

**Xem từng trang ảnh** bằng `view` tool để đọc text chính xác theo layout:

```
view /tmp/pages_p-01.jpg
view /tmp/pages_p-02.jpg
... (tiếp tục cho đến hết)
```

### Quy tắc XANH/ĐỎ (cho sách Kinh Thánh dạng 3 cột Cassell's 1860)

| Vùng | Màu | Hành động |
|------|-----|-----------|
| Running header (đầu trang) | 🟢 GIỮ | Dùng làm `runHdr(left, center, right)` |
| Chapter heading + summary italic | 🟢 GIỮ | Dùng làm `chHead(num, summary)` |
| Cột TRÁI — Bible text (verses) | 🟢 GIỮ | `v(num, text)` |
| Cột PHẢI — Bible text tiếp | 🟢 GIỮ | `v(num, text)` |
| Commentary cuối trang (full width) | 🟢 GIỮ | `com(ref, text)` hoặc `cp(text)` |
| Image captions | 🟢 GIỮ | `cap(text)` |
| Cột GIỮA — marginal cross-refs | 🔴 BỎ | Ví dụ: "a Gen. xv. 4;", "1 Heb. something" |
| Số trang gốc (489, 490...) | 🔴 BỎ | Bỏ hoàn toàn |
| "BEFORE CHRIST XXXX" riêng lẻ | 🔴 BỎ | Bỏ (chỉ giữ ở title page) |

---

## ⚠️ LỖI HAY GẶP KHI ĐỌC NỘI DUNG — ĐỌC KỸ TRƯỚC KHI CODE

### Lỗi 1 — Thiếu số "1" đầu dòng trong chapter summary

Phần summary in nghiêng đầu mỗi chương **luôn bắt đầu bằng số "1"** chỉ verse đầu tiên,
tiếp theo là các số verse khác (5, 8, 11...). Ví dụ nguyên văn:

```
1 Abishag cherisheth David in his extreme age. 5 Adonijah, David's darling,
usurpeth the kingdom. 11 By the counsel of Nathan, 15 Bath-sheba moveth the king...
```

**Lỗi hay mắc:** Bỏ số "1" ở đầu, viết thẳng vào `chHead()` như sau:
```javascript
// ❌ SAI — thiếu "1" đầu dòng
chHead('I', 'Abishag cherisheth David in his extreme age. 5 Adonijah...')
```

**Đúng phải là:**
```javascript
// ✅ ĐÚNG — giữ nguyên số "1" đầu dòng
chHead('I', '1 Abishag cherisheth David in his extreme age. 5 Adonijah...')
```

**Quy tắc:** Luôn đọc thật kỹ dòng đầu tiên của summary — nếu thấy số (thường là "1"),
phải giữ nguyên trong chuỗi truyền vào `chHead()`.

---

### Lỗi 2 — Thiếu đoạn commentary không có số tham chiếu

Phần commentary cuối trang đôi khi có **một đoạn văn mở đầu không có số tham chiếu**
(không phải "1—4." hay "5." mà là plain text chạy thẳng), rồi mới đến các footnote
đánh số. Ví dụ nguyên văn I Kings Chapter I:

```
CHAPTER I.
1—4.  These two Books of Kings, which, in the Hebrew manuscripts...
```

Nhưng phía trên dòng "1—4." đó còn có đoạn không đánh số:

```
The period embraced in these records is from B.C. 1015—562, or rather
more than four hundred and fifty years. The authorship has been ascribed
by some to Jeremiah, whilst others attribute the work of compilation to Ezra...
```

**Lỗi hay mắc:** Chỉ chép phần "1—4." trở đi, bỏ mất đoạn mở đầu không có số.

**Đúng phải là:** Dùng `cp()` cho đoạn không có số, rồi `com()` cho các đoạn có số:
```javascript
// ✅ ĐÚNG
C.push(ch('CHAPTER I.'));
C.push(cp('The period embraced in these records is from B.C. 1015—562, or rather more than four hundred and fifty years...'));
C.push(com('1—4.', 'These two Books of Kings, which, in the Hebrew manuscripts...'));
C.push(com('5.', 'Adonijah, who was the fourth son of David...'));
```

**Quy tắc:** Khi đọc phần commentary, **đọc từ đầu tiêu đề "CHAPTER X."** rồi kéo xuống —
nếu có text nào xuất hiện **trước** footnote đánh số đầu tiên, phải dùng `cp()` để giữ lại.

---

### Lỗi 3 — Commentary trải dài nhiều trang: phải đọc đến khi gặp CHAPTER N+1

**Đây là lỗi gốc rễ quan trọng nhất và không có ngoại lệ.**

#### Quy tắc tuyệt đối

> **Commentary của CHAPTER N bắt đầu từ text `CHAPTER N.` ở phần bottom và kết thúc ngay trước text `CHAPTER N+1.` ở phần bottom. Nếu chưa thấy `CHAPTER N+1.` thì chưa được dừng, dù đã sang trang tiếp theo.**

Tất cả text trong phần bottom của các trang, từ `CHAPTER N.` cho đến (không gồm) `CHAPTER N+1.`, đều thuộc commentary của Chapter N — lấy **nguyên văn đầy đủ**, không bỏ bất kỳ footnote nào.

#### Tại sao lỗi này xảy ra

Bố cục trang Cassell's 1860: phần bottom (commentary) của một trang không nhất thiết kết thúc cùng chương với Bible text phần trên. Commentary CHAPTER N thường **tràn sang 1–3 trang tiếp theo**, xen kẽ dưới Bible text của CHAPTER N+1, N+2...

```
Trang 1:                      Trang 2:                    Trang 3:
[Bible text Ch. I v.1–18]     [Bible text Ch. I v.19–25] [Bible text Ch. II v.1–14]
─────────────────────────     ────────────────────────    ───────────────────────────
CHAPTER I.          ← bắt đầu  [không có heading]          [không có heading]
1. The previous...              3, 4. Ahaziah sends...      9—12. After Elijah...
2. Ekron was...                 5—8. The messengers...      13—16. The third captain...
                                                            17. As Ahaziah...   ← kết thúc Ch.I
                                                            CHAPTER II.         ← bắt đầu Ch.II
                                                            1. Gilgal is...
```

#### Cách làm đúng

**Bước 1 — Khi gặp `CHAPTER N.` trong phần bottom:** bắt đầu thu thập commentary, tiếp tục đọc hết trang đó.

**Bước 2 — Sang trang tiếp theo:** đọc phần bottom. Nếu **không có** heading `CHAPTER N+1.` → toàn bộ nội dung phần bottom vẫn thuộc Chapter N, tiếp tục thu thập.

**Bước 3 — Lặp lại Bước 2** cho mỗi trang tiếp theo cho đến khi thấy `CHAPTER N+1.` trong phần bottom.

**Bước 4 — Ghi vào code** tất cả các `com()` theo đúng thứ tự, nguyên văn đầy đủ.

```javascript
// ✅ ĐÚNG — gom đủ tất cả footnotes từ mọi trang cho đến khi gặp CHAPTER II.
C.push(ch('CHAPTER I.'));
C.push(com('1.', 'The previous Book concluded with the intimation that Ahaziah...'));  // trang 1
C.push(com('2.', 'Ekron was the most northern of the five cities...'));                // trang 1
C.push(com('3, 4.', 'Ahaziah sends to consult this deity as to...'));                  // trang 2
C.push(com('5—8.', 'The messengers did not recognise Elijah; but when...'));           // trang 2
C.push(com('9—12.', 'After Elijah had revealed to the messengers...'));                // trang 3
C.push(com('13—16.', 'The third captain takes quite a different attitude...'));        // trang 3
C.push(com('17.', 'As Ahaziah had no son, he was succeeded by Jehoram...'));           // trang 3

// ❌ SAI — dừng sớm sau trang 1, bỏ mất toàn bộ footnotes 3,4 đến 17
C.push(ch('CHAPTER I.'));
C.push(com('1.', 'The previous Book...'));
C.push(com('2.', 'Ekron was...'));
// ← THIẾU: 3,4 / 5—8 / 9—12 / 13—16 / 17
```

#### Checklist trước khi code một chương

- [ ] Đã đọc phần bottom tất cả trang từ khi gặp `CHAPTER N.` đến khi gặp `CHAPTER N+1.`?
- [ ] Số lượng `com()` trong code có khớp với số footnote đếm được trong ảnh không?
- [ ] Không có footnote nào bị bỏ qua giữa chừng không?

---

## Bước 3 — Build DOCX bằng JavaScript

Vì file thường lớn (20+ chương), **chia script thành nhiều file** rồi `require()` nối nhau.
Mỗi file tối đa ~80,000 bytes để tránh giới hạn bash.

### Helper functions chuẩn

```javascript
'use strict';
const {Document, Packer, Paragraph, TextRun, AlignmentType,
       BorderStyle, PageNumber, Footer} = require('docx');
const fs = require('fs');
const F = 'Georgia';

// Đường kẻ ngang phân cách
const hRule = () => new Paragraph({
  spacing: {before: 200, after: 200},
  border: {bottom: {style: BorderStyle.SINGLE, size: 4, color: '999999'}},
  children: [new TextRun('')]
});

// Running header 3 phần (trái/giữa/phải)
const runHdr = (l, c, r) => new Paragraph({
  alignment: AlignmentType.CENTER,
  spacing: {before: 0, after: 80},
  children: [
    new TextRun({text: l, italics: true, size: 18, font: F}),
    new TextRun({text: `    ${c}    `, bold: true, size: 18, font: F}),
    new TextRun({text: r, italics: true, size: 18, font: F})
  ]
});

// Chapter heading + summary
// ⚠️  summary phải giữ nguyên số "1" ở đầu dòng (xem mục LỖI HAY GẶP #1)
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

// Verse thông thường
const v = (num, text, ital) => new Paragraph({
  alignment: AlignmentType.JUSTIFIED,
  spacing: {before: 80, after: 80},
  indent: {hanging: 380, left: 380},
  children: [
    new TextRun({text: `${num} `, bold: true, size: 22, font: F}),
    new TextRun({text, size: 22, font: F, italics: ital || false})
  ]
});

// Verse với chữ in nghiêng xen kẽ: vmi(num, normal, italic, normal, italic, ...)
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

// Commentary có số tham chiếu
// ⚠️  Nếu có đoạn không có số trước com() đầu tiên, dùng cp() (xem mục LỖI HAY GẶP #2)
const com = (ref, text) => new Paragraph({
  alignment: AlignmentType.JUSTIFIED,
  spacing: {before: 60, after: 60},
  indent: {hanging: 360, left: 360},
  children: [
    new TextRun({text: `${ref}  `, bold: true, size: 20, font: F}),
    new TextRun({text, size: 20, font: F})
  ]
});

// Commentary đoạn văn thông thường (không có số tham chiếu)
const cp = t => new Paragraph({
  alignment: AlignmentType.JUSTIFIED,
  spacing: {before: 80, after: 80},
  children: [new TextRun({text: t, size: 20, font: F})]
});

// Tiêu đề chapter commentary (có border top)
const ch = t => new Paragraph({
  alignment: AlignmentType.CENTER,
  spacing: {before: 320, after: 120},
  border: {top: {style: BorderStyle.SINGLE, size: 4, color: '888888'}},
  children: [new TextRun({text: t, bold: true, size: 22, font: F})]
});

// Caption hình ảnh
const cap = t => new Paragraph({
  alignment: AlignmentType.CENTER,
  spacing: {before: 80, after: 200},
  children: [new TextRun({text: t, bold: true, size: 19, font: F})]
});
```

### Cấu trúc multi-file

```
iiking_p1.js  ← helpers + C[] + chapters I–IX  → exports {C, helpers...}
iiking_p2.js  ← require('./iiking_p1') + chapters X–XVII → exports base
iiking_p3.js  ← require('./iiking_p2') + chapters XVIII–XXV + BUILD DOC
```

**File cuối (p3.js) — phần build:**

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
            new TextRun({text: 'Cassell\'s Illustrated Family Bible (1860) — Book Name  |  p.', size: 18, font: F, color: '888888'}),
            new TextRun({children: [PageNumber.CURRENT], size: 18, font: F, color: '888888'})
          ]
        })]
      })
    },
    children: C
  }]
});

Packer.toBuffer(doc).then(buf => {
  fs.writeFileSync('/mnt/user-data/outputs/Cassells_1860_BOOKNAME_full.docx', buf);
  console.log('Done! Paragraphs:', C.length);
});
```

### Chạy và validate

```bash
node /home/claude/iiking_p3.js

# Validate kết quả
python3 /mnt/skills/public/docx/scripts/office/validate.py \
  /mnt/user-data/outputs/Cassells_1860_BOOKNAME_full.docx
```

---

## Quy tắc quan trọng

1. **Đọc tất cả trang ảnh** trước khi build — không đoán nội dung
2. **Script tối đa ~80,000 bytes** — nếu lớn hơn, chia nhỏ thêm file
3. **Proper names KHÔNG dịch** — giữ nguyên tiếng Anh (Solomon, Elijah, Samaria...)
4. **Verse numbers in bold** — dùng `v(num, text)` không phải plain text
5. **Bỏ footnote refs** — các ký hiệu như `a`, `b`, `1 Heb.`, `Or,` trong cột giữa
6. **vmi()** cho câu mở đầu chapter có chữ cái trang trí to (drop cap)
7. **Validate PASSED** trước khi present_files
8. **Summary bắt đầu bằng "1 ..."** — giữ nguyên số "1" trong `chHead()` (xem LỖI #1)
9. **cp() trước com() đầu tiên** nếu có đoạn plain text không đánh số (xem LỖI #2)

---

## Cấu trúc file đầu ra chuẩn

```
TITLE PAGE
  Book title (56pt bold)
  Subtitle
  Before Christ XXX (italic)
  hRule

CHAPTER I
  runHdr(left, center, right)           ← từ running header trang ảnh
  chHead(I, "1 Summary line. 5 ...")    ← ⚠️ giữ số "1" đầu dòng
  cap("IMAGE CAPTION")                  ← nếu có hình
  vmi(1, "Opening drop-cap text...")    ← verse 1 với chữ to
  v(2, "Normal verse text...")
  ...
  ch("CHAPTER I.")                      ← bắt đầu commentary
  cp("Intro text không có số...")       ← ⚠️ nếu có đoạn plain trước footnotes
  com("1—4.", "Specific note...")
  com("5.", "Another note...")

CHAPTER II
  hRule                                  ← phân cách chương
  ...
```

---

## Ví dụ thực tế đã làm

| File | Chương | Paragraphs | Output |
|------|--------|-----------|--------|
| I Kings (Vol.2) | 22 | 1,046 | `Cassells_1860_I_KINGS_full.docx` |
| II Kings (Vol.2) | 25 | 946 | `Cassells_1860_II_KINGS_full.docx` |

---

## Xử lý lỗi thường gặp

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `Command argument exceeds 100,000 bytes` | Script quá lớn trong bash | Dùng `create_file` rồi `node file.js` |
| `unbalanced parenthesis` regex | Ký tự đặc biệt trong pattern | Escape hoặc tách regex |
| `Paragraphs: 0` khi validate | Chưa copy file vào outputs | Kiểm tra đường dẫn output |
| Text bị lẫn footnotes | pdftotext trộn 2 cột | Đọc từ ảnh, không dùng text thô |
| Summary thiếu số đầu dòng | Quên giữ "1" trong chHead() | Xem mục LỖI HAY GẶP #1 |
| Commentary thiếu đoạn mở đầu | Bỏ text plain trước footnotes | Xem mục LỖI HAY GẶP #2 |
