# Bản tin hôm nay — hệ thống tổng hợp tin tức tự động

Hệ thống này mỗi sáng tự động:
1. Lấy tin **trong ngày** từ Báo Chính phủ, Đại biểu Nhân dân, VnExpress và một số báo trong nước khác.
2. Loại bỏ những tin trùng nội dung giữa các báo, ưu tiên giữ nguồn chính thống.
3. Dùng Claude (AI của Anthropic) viết lại tóm tắt ngắn 3-4 dòng cho từng tin, tự phân loại theo chủ đề.
4. Hiển thị lên một trang web tĩnh, đọc mỗi tin trong khoảng 10 giây.

Toàn bộ chạy miễn phí trên GitHub Actions (không cần server riêng), chỉ tốn chi
phí rất nhỏ cho việc gọi Claude API.

---

## 1. Cấu trúc thư mục

```
news-digest/
├── .github/workflows/daily-update.yml   # lịch chạy tự động mỗi ngày
├── docs/                                # đây là trang web (GitHub Pages đọc từ đây)
│   ├── index.html
│   ├── style.css
│   ├── app.js
│   └── data/latest.json                 # dữ liệu tin tức, được cập nhật tự động mỗi ngày
├── scripts/                              # phần "bộ não" xử lý dữ liệu
│   ├── config.py         # danh sách nguồn tin — sửa ở đây nếu muốn thêm/bớt báo
│   ├── fetch_news.py      # lấy tin thô
│   ├── dedupe.py           # loại trùng
│   ├── summarize.py        # gọi AI tóm tắt
│   ├── check_sources.py    # công cụ kiểm tra nguồn tin còn sống hay không
│   └── main.py              # chạy toàn bộ theo thứ tự
└── requirements.txt
```

---

## 2. Triển khai lên GitHub (làm một lần)

### Bước 1 — Tạo repository mới
Vào GitHub → **New repository** → đặt tên (ví dụ `news-digest`) → chọn **Public**
(bắt buộc để dùng GitHub Pages miễn phí) → **Create repository**.

### Bước 2 — Đưa toàn bộ file lên
Cách dễ nhất nếu chưa quen dòng lệnh: mở repo vừa tạo → **Add file → Upload
files** → kéo thả **toàn bộ thư mục** `news-digest` (đã giải nén) vào — GitHub
sẽ tự giữ nguyên cấu trúc thư mục con. Viết commit message rồi bấm **Commit
changes**.

*(Nếu quen dùng GitHub Desktop hoặc dòng lệnh git, cách đó cũng hoàn toàn được.)*

### Bước 3 — Lấy Anthropic API key
Vào [console.anthropic.com](https://console.anthropic.com) → tạo tài khoản
(nếu chưa có) → mục **API Keys** → **Create Key** → copy key (dạng
`sk-ant-...`). Đây là "chìa khóa" để hệ thống gọi được Claude.

### Bước 4 — Lưu API key vào GitHub (bí mật, không public)
Trong repo → **Settings → Secrets and variables → Actions → New repository
secret**:
- Name: `ANTHROPIC_API_KEY`
- Secret: dán API key vừa copy

### Bước 5 — Bật GitHub Pages
**Settings → Pages** → mục **Build and deployment → Source**: chọn **Deploy
from a branch** → Branch: chọn `main`, thư mục chọn **/docs** → **Save**.
Sau khoảng 1 phút, GitHub sẽ cho bạn một đường link dạng:
`https://<tên-github-của-bạn>.github.io/news-digest/`

### Bước 6 — Chạy thử lần đầu
Vào tab **Actions** trên repo → chọn workflow **"Cập nhật bản tin hằng ngày"**
→ **Run workflow** → **Run workflow** (chạy thủ công, không cần đợi tới lịch
6 giờ sáng). Đợi khoảng 1-3 phút, refresh trang web ở Bước 5 là thấy tin.

Từ hôm sau, hệ thống tự chạy lúc **6:00 sáng giờ Việt Nam** mỗi ngày, không
cần bạn làm gì thêm.

---

## 3. Khi một nguồn tin bị "gãy"

Các báo tiếng Việt thỉnh thoảng đổi cấu trúc RSS hoặc chặn bot. Khi đó nguồn
đó sẽ tự động bị bỏ qua (không làm hỏng cả hệ thống), nhưng bạn nên sửa lại
sớm để không bị thiếu tin.

Chạy lệnh sau (cần cài Python ở máy, hoặc chạy trong Codespace của GitHub):

```bash
pip install -r requirements.txt
cd scripts
python check_sources.py
```

Kết quả sẽ liệt kê nguồn nào ✅ OK, nguồn nào ❌ LỖI. Với nguồn lỗi, tìm URL
RSS mới (thử tìm "`<tên báo>` rss feed" trên Google, hoặc mở trang báo tìm
biểu tượng RSS/cam) rồi sửa lại trong `scripts/config.py`.

> Lưu ý: trong `config.py`, những nguồn có `"verified": False` là URL suy đoán
> theo quy ước phổ biến của các báo Việt Nam, **chưa được xác nhận chắc chắn**
> — nên chạy `check_sources.py` ngay sau khi triển khai lần đầu để biết nguồn
> nào cần chỉnh.

---

## 4. Thêm nguồn tin mới

Mở `scripts/config.py`, thêm một dict vào danh sách `SOURCES`:

```python
{
    "name": "Tên báo",
    "type": "rss",              # hoặc "html" nếu báo không có RSS
    "url": "https://.../rss.xml",
    "priority": 4,               # 1-3 dành cho 3 nguồn ưu tiên cao nhất, còn lại dùng 4
    "verified": False,
},
```

---

## 5. Một vài giới hạn cần biết

- **Báo Chính phủ và Đại biểu Nhân dân** không có RSS công khai rõ ràng, nên
  hệ thống dò trực tiếp trang danh sách bài viết (`type: "html"`). Cách này
  hoạt động được nhưng dễ bị ảnh hưởng hơn RSS nếu báo đổi giao diện.
- Với các nguồn HTML, nếu không xác định được thời gian đăng bài, hệ thống sẽ
  **loại bỏ bài đó** thay vì liều hiển thị (tránh đăng nhầm tin cũ).
- Chi phí gọi Claude API rất thấp vì dùng model Haiku (rẻ nhất) và mỗi ngày
  chỉ xử lý vài chục tin; có thể xem chi phí thực tế tại
  [console.anthropic.com](https://console.anthropic.com) mục Usage.
- Muốn tóm tắt chất lượng cao hơn (đánh đổi chi phí cao hơn một chút), đổi
  `SUMMARY_MODEL` trong `config.py` từ `"claude-haiku-4-5-20251001"` sang
  `"claude-sonnet-5"`.

---

## 6. Chạy thử trên máy cá nhân (không bắt buộc)

```bash
pip install -r requirements.txt
export ANTHROPIC_API_KEY=sk-ant-xxxx      # Windows PowerShell: $env:ANTHROPIC_API_KEY="sk-ant-xxxx"
cd scripts
python main.py
```

File kết quả sẽ được ghi vào `docs/data/latest.json`. Mở
`docs/index.html` trực tiếp bằng trình duyệt để xem thử (một số trình duyệt
chặn `fetch()` khi mở file trực tiếp — nếu vậy, chạy `python -m http.server`
trong thư mục `docs` rồi mở `http://localhost:8000`).
