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

## LUẬT SỐ 1 — NGUYÊN VĂN TUYỆT ĐỐI

> Mọi nội dung trong DOCX — summary, verse, commentary — phải là chuỗi ký tự được **copy
> trực tiếp từ nguồn gốc** (ảnh hoặc OCR text). Không có ngoại lệ.

**Điều này có nghĩa là:**
- Trước khi viết bất kỳ dòng `chHead()`, `v()`, `com()`, `cp()` nào, AI phải đã **đọc
  nguồn tương ứng** và có chuỗi text gốc trong tay.
- Không được dùng trí nhớ, không được suy luận, không được tóm tắt, không được
  paraphrase, không được "điền vào" những gì có vẻ hợp lý.
- Nếu nội dung bị mờ hay thiếu một đoạn: để trống hoặc ghi `[illegible]`, không được tự bịa.

### Ba loại nội dung — ba nguồn tương ứng

| Nội dung | Nguồn | Hàm JS |
|----------|-------|--------|
| **Chapter summary** | Dòng bắt đầu bằng số `1 ...` ngay sau `CHAPTER X.` lần đầu, trước verse 1 | `chHead(num, summary)` |
| **Verse text** | Cột trái/phải của trang, các dòng bắt đầu bằng số verse | `v(num, text)` / `vmi(num, ...)` |
| **Commentary** | Toàn bộ text sau `CHAPTER X.` lần thứ hai trong trang | `com(ref, text)` / `cp(text)` |

---

## Bước 1 — Kiểm tra và chuẩn bị file

```bash
# Kiểm tra loại file
file /mnt/project/<tên-file>.pdf
ls -lh /mnt/project/<tên-file>.pdf

# --- Phương pháp A: Rasterize thành ảnh (cho PDF scan) ---
pdftoppm -jpeg -r 150 /mnt/project/<tên-file>.pdf /tmp/pages_p
ls /tmp/pages_p*.jpg | wc -l   # đếm số trang

# --- Phương pháp B: Giải nén nếu là ZIP chứa txt (Cassell's OCR) ---
unzip /mnt/project/<tên-file>.pdf -d /tmp/book/
ls /tmp/book/*.txt | wc -l   # đếm số trang txt
```

---

## Bước 2 — Đọc nội dung

Dùng **một trong hai phương pháp** tùy theo file:

### Phương pháp A — Đọc từ ảnh (view tool)

**Xem từng trang ảnh** để đọc text chính xác theo layout:

```
view /tmp/pages_p-01.jpg
view /tmp/pages_p-02.jpg
... (tiếp tục cho đến hết)
```

### Phương pháp B — Đọc từ OCR text (bash)

#### 2B-1. Đọc và trích xuất summary

Summary nằm giữa `CHAPTER X.` lần đầu và dòng đầu tiên của verse.

```bash
# Đọc toàn bộ trang đầu của chương
cat /tmp/book/PAGE.txt

# Summary nhận dạng: dòng bắt đầu bằng chữ số "1 ..." (không phải footnote)
# Footnote: "1 Heb. xxx", "2 Or, xxx" — BỎ
# Summary: "1 Tên người động từ..." (câu hoàn chỉnh có nghĩa) — GIỮ
```

#### 2B-2. Đọc và trích xuất verse text

```bash
cat /tmp/book/PAGE.txt
cat /tmp/book/PAGE+1.txt
# ...
```

**Nhận dạng verse:** dòng bắt đầu bằng số (`2 Wherefore his servants...`).
**Lọc bỏ cross-refs cột giữa:** `a Gen. xv. 4;` / `1 Heb. xxx` / `2 Sam. iii.` — **BỎ**.
**Verse bị ngắt dòng** — nối lại thành câu hoàn chỉnh.

#### 2B-3. Đọc và trích xuất commentary

```bash
# Trích commentary: bắt đầu từ "CHAPTER X." lần thứ hai
awk '/^CHAPTER [IVX]+\./{n++; if(n==2){found=1; next}} found{print}' /tmp/book/PAGE.txt \
  | grep -v "^[0-9]\{2,3\}$"   # bỏ số trang
```

Commentary thường **tràn sang trang tiếp theo** — phải đọc thêm cho đến khi gặp `CHAPTER N+1.`.

---

### Quy tắc XANH/ĐỎ (áp dụng cho cả 2 phương pháp)

| Vùng | Màu | Hành động |
|------|-----|-----------|
| Running header (đầu trang) | 🟢 GIỮ | `runHdr(left, center, right)` |
| Chapter heading + summary italic | 🟢 GIỮ | `chHead(num, summary)` |
| Cột TRÁI — Bible text (verses) | 🟢 GIỮ NGUYÊN VĂN | `v(num, text)` |
| Cột PHẢI — Bible text tiếp | 🟢 GIỮ NGUYÊN VĂN | `v(num, text)` |
| Commentary cuối trang (full width) | 🟢 GIỮ NGUYÊN VĂN | `com(ref, text)` hoặc `cp(text)` |
| Image captions | 🟢 GIỮ | `cap(text)` |
| Cột GIỮA — marginal cross-refs | 🔴 BỎ | `a Gen. xv. 4;`, `1 Heb. something` |
| Số trang gốc (489, 490...) | 🔴 BỎ | Bỏ hoàn toàn |
| `BEFORE CHRIST XXXX` riêng lẻ | 🔴 BỎ | Bỏ (chỉ giữ ở title page) |
| Dòng `i`, `!`, `e` đơn lẻ | 🔴 BỎ | Artifact của OCR |
| Running header `Rebellion of Moab. II. KINGS, I.` | 🔴 BỎ | Trong commentary |

---

## ⚠️ LỖI HAY GẶP — ĐỌC KỸ TRƯỚC KHI CODE

### Lỗi 1 — Thiếu số "1" đầu dòng trong chapter summary

Phần summary **luôn bắt đầu bằng số "1"**, tiếp theo là các số verse khác.

```
1 Abishag cherisheth David in his extreme age. 5 Adonijah, David's darling,
usurpeth the kingdom. 11 By the counsel of Nathan, 15 Bath-sheba moveth the king...
```

```javascript
// ❌ SAI — thiếu "1" đầu dòng
chHead('I', 'Abishag cherisheth David in his extreme age. 5 Adonijah...')

// ✅ ĐÚNG — giữ nguyên số "1" đầu dòng
chHead('I', '1 Abishag cherisheth David in his extreme age. 5 Adonijah...')
```

**Quy tắc:** Luôn đọc thật kỹ dòng đầu tiên của summary — nếu thấy số (thường là "1"), phải giữ nguyên.

---

### Lỗi 2 — Thiếu đoạn commentary không có số tham chiếu

Commentary đôi khi có **một đoạn văn mở đầu không có số tham chiếu** thực sự là entry độc lập, đứng trước các footnote đánh số.

```
The period embraced in these records is from B.C. 1015—562...   ← đoạn plain độc lập
CHAPTER I.
1—4.  These two Books of Kings...                                ← mới có số
```

```javascript
// ✅ ĐÚNG — dùng cp() cho đoạn không có số (đã xác nhận là entry độc lập)
C.push(ch('CHAPTER I.'));
C.push(cp('The period embraced in these records is from B.C. 1015—562, or rather more than four hundred and fifty years...'));
C.push(com('1—4.', 'These two Books of Kings, which, in the Hebrew manuscripts...'));
C.push(com('5.', 'Adonijah, who was the fourth son of David...'));
```

**Quy tắc:** Khi đọc commentary, đọc **từ đầu tiêu đề "CHAPTER X."** — nếu có text nào trước footnote đánh số đầu tiên, dùng `cp()`. Tuy nhiên, **trước khi dùng `cp()`, phải xác nhận đây là entry độc lập** chứ không phải phần bị cắt của `com()` trước (xem Lỗi 3).

---

### Lỗi 3 — Commentary bị cắt đôi bởi layout 2 cột (QUAN TRỌNG)

**Đây là lỗi tinh vi nhất trong sách Cassell's 1860.** OCR đọc tuyến tính từ trên xuống,
nên commentary của một entry (ví dụ `1—4.`) có thể bị **chia làm 2 mảnh** nằm ở 2 vị trí
khác nhau trong file OCR vì layout 2 cột của trang gốc.

#### Tại sao xảy ra

Trang Cassell's có 2 cột Bible text ở trên, và commentary full-width ở dưới. Nhưng do
layout 2 cột, OCR đôi khi đọc như sau:

```
[Cột trái - Bible verses 1-18]   [Cột phải - Bible verses 19-34]
─────────────────────────────────────────────────────────────────
CHAPTER I.                        The period embraced...     ← MẢNH 2 của 1—4.
1—4. These two Books of Kings...  "Covered him with clothes."
                                  5—8. Adonijah, who was...
```

OCR tuyến tính sẽ xuất ra: `CHAPTER I.` → `1—4. These two Books...` → verse 19-34 → `The period embraced...` → `"Covered him with clothes."` → `5—8....`

Kết quả: **"The period embraced..."** xuất hiện SAU các verses 19-34 trong file OCR, khiến
AI nhầm tưởng đó là một đoạn `cp()` riêng biệt, trong khi thực ra nó là **phần tiếp của
`1—4.`** bị cột phải cắt xen vào.

#### Cách nhận biết

Đọc OCR của **2 trang liên tiếp** quanh đầu chapter. Nếu thấy:
- Trang N có `CHAPTER X.` + `1—4. [đoạn A]`
- Trang N+1 có `[đoạn B không có số tham chiếu]` + `"Quoted phrase."` + `5—8. [...]`

Thì `[đoạn B]` rất có thể là **phần còn lại của `1—4.`**, không phải entry mới.

#### Kiểm tra xác nhận

```bash
# Đọc 2 trang đầu của chapter để xác định vị trí thực của đoạn text
cat /tmp/book/PAGE1.txt
cat /tmp/book/PAGE2.txt
# Tìm xem đoạn text không có số tham chiếu nằm trước hay sau 1—4.
```

Nếu `[đoạn B]` xuất hiện **sau** `1—4.` trong cùng một block commentary → gộp vào `1—4.`.
Nếu `[đoạn B]` xuất hiện **trước** `1—4.` → dùng `cp()` riêng (đây là Lỗi 2 hợp lệ).

#### Cách xử lý đúng

```javascript
// ❌ SAI — tách thành cp() riêng vì thấy nó không có số tham chiếu
C.push(ch('CHAPTER I.'));
C.push(cp('The period embraced in these records is from B.C. 1015-562...'));
C.push(com('1—4.', 'These two Books of Kings...'));

// ✅ ĐÚNG — gộp vào com('1—4.') vì đây là phần 2 của cùng entry, bị layout cắt xen
C.push(ch('CHAPTER I.'));
C.push(com('1—4.', 'These two Books of Kings, which, in the Hebrew manuscripts, were, '
  + 'for a long time, given as one continuous work... [đoạn A] '
  + 'The period embraced in these records is from B.C. 1015-562... [đoạn B]'));
```

**Quy tắc:** Trước khi viết `cp()` cho bất kỳ đoạn nào không có số tham chiếu, **phải đọc
lại OCR raw** để xác nhận đoạn đó thực sự là entry độc lập hay là phần bị cắt của entry
trước. Không được đoán chỉ từ nội dung.

---

### Lỗi 4 — Commentary trải dài nhiều trang: phải đọc đến khi gặp CHAPTER N+1

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
C.push(com('3, 4.', 'Ahaziah sends to consult this deity as to the probability...'));  // trang 2
C.push(com('5—8.', 'The messengers did not recognise Elijah; but when...'));           // trang 2
C.push(com('9—12.', 'After Elijah had revealed to the messengers...'));                // trang 3
C.push(com('13—16.', 'The third captain takes quite a different attitude...'));        // trang 3
C.push(com('17.', 'As Ahaziah had no son, he was succeeded by Jehoram...'));           // trang 3
// Đây mới thấy CHAPTER II. → dừng Ch.I, bắt đầu Ch.II

// ❌ SAI — dừng sớm sau trang 1, bỏ mất toàn bộ footnotes 3,4 đến 17
C.push(ch('CHAPTER I.'));
C.push(com('1.', 'The previous Book...'));
C.push(com('2.', 'Ekron was...'));
// ← THIẾU: 3,4 / 5—8 / 9—12 / 13—16 / 17
```

#### Checklist trước khi code một chương

- [ ] Đã đọc phần bottom tất cả trang từ khi gặp `CHAPTER N.` đến khi gặp `CHAPTER N+1.`?
- [ ] Số lượng `com()` trong code có khớp với số footnote đếm được không?
- [ ] Không có footnote nào bị bỏ qua giữa chừng không?

---

### Lỗi 5 — OCR ngắt dòng giữa câu commentary, phần đuôi bị nhầm là cross-ref cột giữa

**Đây là lỗi tinh vi, xảy ra ngay trong một câu commentary, không liên quan đến tràn trang.**

#### Tại sao xảy ra

OCR đọc tuyến tính và ngắt dòng tùy vị trí trên trang. Một câu commentary dài — đặc biệt câu kết thúc bằng tham chiếu Scripture như `2 Kings x. 32, 33; xiii. 3-7.` — có thể bị tách thành hai dòng:

```
"threshed Gilead :-on this and the following verse consult 2
Kings x.32 ,33 ;
xiii .3-7.
5But Iwill send a fire...
```

Dòng `xiii .3-7` xuất hiện riêng lẻ, trông y hệt một marginal cross-ref cột giữa (dạng `a Gen. xv. 4;`, `1 Heb. xxx`) — vốn phải bỏ theo quy tắc XANH/ĐỎ. Kết quả: AI bỏ nó đi, khiến câu commentary bị cụt mất phần đuôi.

#### Cách nhận biết

Trước khi lọc bỏ bất kỳ dòng ngắn nào dạng `Tên.sách .số` trong vùng commentary, hỏi:

> **Câu commentary ngay trên dòng này đã kết thúc bằng dấu chấm chưa?**

- **Chưa có dấu chấm** → dòng đó là phần **tiếp của câu**, không phải cross-ref — **GIỮ LẠI và nối vào**.
- **Đã có dấu chấm** → mới được xem xét bỏ theo quy tắc XANH/ĐỎ bình thường.

#### Ví dụ thực tế (AMOS, Chapter I, commentary số 3)

```
# OCR raw — trang 2
...consult 2
Kings x.32 ,33 ;
xiii .3-7.
5But Iwill send a fire...
```

```javascript
// ❌ SAI — cắt câu sớm, bỏ "xiii. 3-7" vì trông giống cross-ref
com('3.', '...on this and the following verse consult 2 Kings x. 32, 33.')

// ✅ ĐÚNG — đọc đến hết câu (dấu chấm sau "3-7")
com('3.', '...on this and the following verse consult 2 Kings x. 32, 33; xiii. 3-7.')
```

#### Checklist bổ sung cho từng `com()`

- [ ] Câu cuối cùng trong `com()` này kết thúc bằng dấu chấm chưa?
- [ ] Nếu chưa: đọc lại OCR dòng ngay kế tiếp — có phải phần đuôi bị ngắt không?

---

## Bước 3 — Build DOCX bằng JavaScript

Chỉ sau khi đã đọc xong nguồn của một chương mới được viết code cho chương đó.

### Helper functions chuẩn

```javascript
'use strict';
const {Document, Packer, Paragraph, TextRun, AlignmentType,
       BorderStyle, PageNumber, Footer} = require('docx');
const fs = require('fs');
const F = 'Georgia';

// Đường kẻ ngang phân cách chương
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
// ⚠️  summary phải giữ nguyên số "1" ở đầu dòng (xem LỖI #1)
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

// Verse thông thường — text = NGUYÊN VĂN
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
// Dùng cho verse 1 có drop cap hoặc chữ cái trang trí
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

// Commentary có số tham chiếu — ref và text = NGUYÊN VĂN
// ⚠️  Xem LỖI #3 trước khi quyết định dùng cp() hay gộp đoạn plain vào com() trước đó
// ⚠️  Xem LỖI #5: câu cuối phải kết thúc bằng dấu chấm — nếu chưa, đọc thêm OCR
const com = (ref, text) => new Paragraph({
  alignment: AlignmentType.JUSTIFIED,
  spacing: {before: 60, after: 60},
  indent: {hanging: 360, left: 360},
  children: [
    new TextRun({text: `${ref}  `, bold: true, size: 20, font: F}),
    new TextRun({text, size: 20, font: F})
  ]
});

// Commentary đoạn văn thông thường (không có số tham chiếu) — text = NGUYÊN VĂN
// ⚠️  Chỉ dùng khi đã XÁC NHẬN đây là entry độc lập, không phải phần bị cắt của com() trước
//     (đọc OCR raw để xác nhận — xem LỖI #2 và LỖI #3)
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

Vì file thường lớn (20+ chương), **chia script thành nhiều file** rồi `require()` nối nhau.
Mỗi file tối đa ~80,000 bytes để tránh giới hạn bash.

```
book_p1.js  ← helpers + C[] + chapters I–VIII    → exports {C, helpers...}
book_p2.js  ← require('./book_p1') + IX–XVI      → exports base
book_p3.js  ← require('./book_p2') + XVII–XXV + BUILD
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
node /home/claude/book_p3.js

python3 /mnt/skills/public/docx/scripts/office/validate.py \
  /mnt/user-data/outputs/Cassells_1860_BOOKNAME_full.docx
```

---

## Quy tắc quan trọng

1. **Đọc tất cả nguồn** (ảnh hoặc txt) trước khi build — không đoán nội dung
2. **Script tối đa ~80,000 bytes** — nếu lớn hơn, chia nhỏ thêm file
3. **Proper names KHÔNG dịch** — Solomon, Elijah, Samaria...
4. **Verse 1 dùng `vmi()`** — OCR thường tách drop cap thành ký tự đơn
5. **Verse numbers in bold** — dùng `v(num, text)` không phải plain text
6. **Bỏ footnote refs cột giữa** — `a`, `b`, `1 Heb.`, `Or,`
7. **Summary bắt đầu bằng "1 ..."** — giữ nguyên số "1" trong `chHead()` (xem LỖI #1)
8. **`cp()` CHỈ sau khi xác nhận OCR** — trước khi dùng `cp()`, đọc OCR raw để xác nhận đây là entry độc lập, không phải phần bị cắt của `com()` trước (xem LỖI #2 và LỖI #3)
9. **Commentary tràn trang** — đọc tiếp cho đến khi gặp `CHAPTER N+1.` (xem LỖI #4)
10. **Câu cuối `com()` phải kết thúc bằng dấu chấm** — nếu chưa, đọc OCR dòng kế tiếp để tìm phần đuôi bị ngắt (xem LỖI #5)
11. **Validate PASSED** trước khi `present_files`

---

## Cấu trúc file đầu ra chuẩn

```
TITLE PAGE
  Book title (56pt bold)
  Subtitle
  Before Christ XXX (italic)
  hRule

CHAPTER I
  runHdr(left, center, right)           ← từ running header trang ảnh/OCR
  chHead(I, "1 Summary line. 5 ...")    ← ⚠️ giữ số "1" đầu dòng
  cap("IMAGE CAPTION")                  ← nếu có hình
  vmi(1, "Opening drop-cap text...")    ← verse 1 với chữ to
  v(2, "Normal verse text...")
  ...
  ch("CHAPTER I.")                      ← bắt đầu commentary
  cp("Intro text không có số...")       ← ⚠️ CHỈ dùng sau khi xác nhận là entry độc lập
  com("1—4.", "These two Books...")     ← ⚠️ gộp đủ cả phần bị layout 2 cột cắt xen
  com("5.", "Another note...")          ← ⚠️ câu cuối phải kết thúc bằng dấu chấm

CHAPTER II
  hRule                                  ← phân cách chương
  ...
```

---

## Ví dụ thực tế đã làm

| File | Phương pháp | Chương | Paragraphs | Output |
|------|-------------|--------|-----------|--------|
| I Kings (Vol.2) | OCR txt | 22 | 929 | `Cassells_1860_I_KINGS_full.docx` |
| II Kings (Vol.2) | OCR txt | 25 | 1,082 | `Cassells_1860_II_KINGS_full.docx` |
| I Kings (Vol.2) | Visual ảnh | 22 | 1,046 | `Cassells_1860_I_KINGS_full.docx` |
| II Kings (Vol.2) | Visual ảnh | 25 | 946 | `Cassells_1860_II_KINGS_full.docx` |

---

## Xử lý lỗi thường gặp

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `Command argument exceeds 100,000 bytes` | Script quá lớn trong bash | Dùng `create_file` rồi `node file.js` |
| `unbalanced parenthesis` regex | Ký tự đặc biệt trong pattern | Escape hoặc tách regex |
| Summary thiếu số | Không đọc nguồn, tự viết lại | Đọc lại, copy nguyên văn, giữ số "1" |
| Commentary sai hoàn toàn | Tự tóm tắt thay vì đọc nguồn | Đọc lại từ ảnh/txt, copy nguyên văn |
| Commentary thiếu đoạn cuối | Dừng sớm, không đọc hết trang | Xem LỖI #4, đọc đến khi gặp CHAPTER N+1 |
| Commentary thiếu đoạn mở đầu | Bỏ text plain trước footnotes | Xem LỖI #2, thêm `cp()` sau khi xác nhận |
| `cp()` sai chỗ — đoạn thực ra thuộc `com()` trước | Layout 2 cột cắt entry commentary | Xem LỖI #3, đọc OCR raw 2 trang liên tiếp |
| Cuối câu `com()` bị cụt, thiếu tham chiếu Scripture | OCR ngắt dòng giữa câu, phần đuôi bị nhầm là cross-ref | Xem LỖI #5, kiểm tra câu cuối có dấu chấm chưa |
| Verse bị thiếu/sai | Dùng trí nhớ | Đọc lại nguồn, copy từng câu |
| Text bị lẫn footnotes | OCR trộn cột giữa vào | Lọc các dòng dạng `a Tên.sách.` và `1 Heb.` |
| `Paragraphs: 0` khi validate | Chưa copy file vào outputs | Kiểm tra đường dẫn `writeFileSync` |
