# Bài tập tháng 8 — Đọc một tập số đo độ trễ

**Giao ngày:** 01/8/2026 **Hạn nộp:** 22/8/2026
**Trình bày:** 15 phút + hỏi đáp, 10:30 ngày 25/8/2026
**Ngân sách thời gian dự kiến:** khoảng 12 giờ mỗi tuần

---

## Bối cảnh

Hai tệp kèm theo là bản ghi độ trễ của một hệ thống phục vụ mô hình ngôn ngữ trong khoảng một giờ vận hành. Dữ liệu đã được sinh ra để phục vụ bài tập này, nhưng cấu trúc và các vấn đề trong đó mô phỏng những gì gặp phải khi đo hệ thống thật.

**`requests.csv`** — mỗi dòng một request

| Cột | Ý nghĩa |
|---|---|
| `request_id` | định danh |
| `arrival_ts_ms` | thời điểm request đến, tính bằng ms từ đầu cửa sổ đo |
| `prompt_tokens` | số token đầu vào |
| `output_tokens` | số token sinh ra |
| `ttft_ms` | time to first token — từ lúc gửi đến lúc nhận token đầu tiên |
| `e2e_ms` | tổng thời gian từ lúc gửi đến lúc nhận token cuối |
| `status` | trạng thái kết thúc của request |

**`tokens.csv`** — mỗi dòng một token sinh ra sau token đầu tiên

| Cột | Ý nghĩa |
|---|---|
| `request_id` | khớp với bảng trên |
| `token_index` | thứ tự token trong request |
| `itl_ms` | inter-token latency — khoảng cách tới token liền trước |

Công cụ: chỉ cần `numpy`, `pandas`, `matplotlib`. Không cần GPU, không cần cài gì phức tạp.

---

## Sáu câu hỏi

### 1. Bộ dữ liệu này đang ở tình trạng nào?

Trước khi tính bất cứ con số nào, hãy kiểm tra dữ liệu. Có bao nhiêu request thực sự dùng được để phân tích độ trễ? Em loại bỏ những gì, và vì sao?

Với mỗi thứ em loại hoặc sửa, nêu bằng chứng cụ thể trong dữ liệu, không nêu bằng nguyên tắc chung.

### 2. TTFT phân phối như thế nào?

Mô tả phân phối của `ttft_ms`. Dạng gì, tham số ước lượng bao nhiêu, và **dựa vào đâu mà em kết luận như vậy**.

Nếu em thấy phân phối có cấu trúc bên trong, hãy mô tả cấu trúc đó và đề xuất một cơ chế kỹ thuật có thể sinh ra nó. Câu trả lời cho phần này không nằm trong bảng dữ liệu, nó nằm ở chỗ em nghĩ hệ thống đã làm gì.

### 3. Các con số tóm tắt

Báo cáo mean, p50, p90, p99 của TTFT, mỗi con số kèm khoảng tin cậy 95%.

Nếu em cho rằng một trong các con số này gây hiểu nhầm khi báo cáo cho người khác, nói rõ con số nào và vì sao.

### 4. Cần bao nhiêu số đo?

Nếu chỉ đo được một số lượng request hạn chế, cần khoảng bao nhiêu request để ước lượng p99 của TTFT với sai số trong khoảng ±10%?

Không trích công thức. Trả lời bằng một thí nghiệm chạy trên chính bộ dữ liệu này, và trình bày cách em thiết kế thí nghiệm đó.

### 5. Một hình

Vẽ **đúng một hình** thể hiện điều quan trọng nhất trong bộ dữ liệu này, kèm ba câu giải thích vì sao em chọn hình đó thay vì các hình khác em đã thử.

Một hình. Không phải một bảng bốn ô. Việc phải chọn là phần chính của câu hỏi.

### 6. Điều em chưa xác định được

Liệt kê những câu hỏi em đặt ra mà dữ liệu hiện có không trả lời được, và giải thích vì sao không. Nếu em cần thêm cột dữ liệu nào để trả lời, ghi rõ cột đó là gì.

Mục này được chấm ngang trọng số với năm mục trên. Một bài nộp không có mục này sẽ được trả lại.

---

## Sản phẩm nộp

Một repo git gồm:

1. **`bao_cao.md`** — 3 đến 5 trang. Mỗi con số nêu trong báo cáo phải ghi kèm số ô (cell) trong notebook đã sinh ra nó, dạng `[ô 12]`.
2. **`phan_tich.ipynb`** — notebook đã chạy, còn nguyên output, các ô đánh số tăng dần theo thứ tự chạy.
3. **`hinh.png`** — hình ở câu 5.
4. **`nhat_ky.md`** — tối thiểu 8 mục có ngày tháng, mỗi mục ghi hôm đó thử gì và cái gì không chạy được.

### Cách phát biểu

Mọi khẳng định trong báo cáo phải được gắn nhãn:

- **[đo được]** — có số liệu và có ô notebook tương ứng
- **[suy ra]** — kết luận rút từ số liệu, phải nêu rõ giả định
- **[phỏng đoán]** — chưa kiểm chứng được

Không dùng "luôn luôn", "không bao giờ", "chắc chắn" cho các phát biểu về dữ liệu.

---

## Chú ý

**Về công cụ AI.** Được dùng, không phải xin phép. Nhưng buổi trình bày sẽ có phần hỏi đáp về những lựa chọn cụ thể trong bài, và em cần giải thích được vì sao chọn cách này thay vì cách khác. Đoạn nào em không giải thích được thì tốt hơn hết là đừng đưa vào.

**Về buổi trình bày.** Sau phần hỏi đáp, em sẽ nhận **một bộ dữ liệu thứ hai** cùng định dạng và có 30 phút, mở laptop, để trả lời ba câu hỏi về nó. Nói trước để em chuẩn bị đúng hướng: thứ giúp em qua phần này là hiểu bộ thứ nhất, không phải là có sẵn đoạn mã chạy được trên bộ thứ nhất.

**Về việc bị tắc.** Sẽ có một điểm kiểm tra vào cuối tuần thứ hai. Ngoài mốc đó ra, không ai nhắc và không ai hỏi thăm. Nếu tắc thì chủ động hỏi, kèm theo: đã thử gì, thông báo lỗi nói gì, và giả thuyết hiện tại là gì.

---

## Điểm kiểm tra tuần 2

Gửi một tin nhắn, không cần trang trọng, gồm: đã trả lời được câu nào, đang tắc ở đâu, và một con số bất kỳ em đã tính ra được. Mục đích là để biết em còn đang đi hay đã dừng, không phải để chấm.
