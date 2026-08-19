# Nhật ký thực hiện bài tập phân tích độ trễ

## 10/08/2026 — Thiết lập môi trường và làm quen với dữ liệu

Mục tiêu đầu tiên của em là tạo môi trường làm việc ổn định trước khi phân tích. Em tạo thư mục project `Measure_latency`, virtual environment `.venv`, cài `numpy`, `pandas`, `matplotlib`, Jupyter và chọn đúng kernel của `.venv` cho notebook.

Vấn đề gặp phải là PowerShell không cho chạy `.venv\Scripts\Activate.ps1` do Execution Policy, đồng thời máy có nhiều bản Python nên có nguy cơ chọn nhầm môi trường. Em xử lý bằng `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` cho phiên hiện tại và kiểm tra lại đường dẫn Python trước khi tiếp tục.

Sau đó em đọc được `requests.csv` và `tokens.csv`, dùng `shape`, `head()` và `info()` để xem cấu trúc. `requests.csv` có 12.040 dòng, 7 cột; `tokens.csv` có 1.829.816 dòng, 3 cột. Qua bước này chưa thấy giá trị null, nhưng em nhận ra rằng không có null chưa có nghĩa dữ liệu đã sạch: vẫn cần kiểm tra duplicate, logic thời gian, trạng thái request và quan hệ giữa request-level với token-level data.

## 10/08/2026 — Kiểm tra `request_id` và duplicate

Em bắt đầu audit `requests.csv` bằng tính duy nhất của `request_id`. Kết quả cho thấy 12.040 dòng nhưng chỉ có 12.000 ID khác nhau; 80 dòng thuộc các ID bị lặp [ô 07].

Ban đầu em chưa xoá ngay các dòng lặp vì cùng `request_id` chưa đủ để kết luận duplicate. Kiểm tra sâu hơn cho thấy có đúng 40 ID bị lặp, mỗi ID xuất hiện hai lần và cả 80 dòng đều trùng hoàn toàn trên 7 cột [ô 08]. Từ đó em mới có cơ sở coi 40 lần xuất hiện thứ hai là bản sao dư và tạo một dataframe làm việc riêng, vẫn giữ dữ liệu gốc để đối chiếu.

Em cũng kiểm tra nhóm `cancelled` thay vì áp dụng ngay quy tắc `status == ok`. Tất cả 136 request `cancelled` đều có token đầu ra, TTFT dương và TTFT không vượt E2E [ô 15], nên tại thời điểm này em giữ chúng trong tập ứng viên cho phân tích TTFT.

## 11/08/2026 — Hoàn thành Câu 1: audit `requests.csv`

Sau khi loại 40 bản sao dư, `requests_work` còn 12.000 request duy nhất [ô 09]. Phân bố trạng thái gồm 11.689 `ok`, 136 `cancelled` và 175 `error` [ô 10].

Một hướng xử lý đơn giản là chỉ giữ `status == ok`, nhưng kiểm tra dữ liệu cho thấy cách đó không phù hợp với bộ này. Toàn bộ `error` có `output_tokens = 0`, trong khi toàn bộ `ok` và `cancelled` đều có token đầu ra [ô 12, ô 17]. Vì TTFT được định nghĩa là thời gian đến token đầu tiên, em loại 175 request `error` khỏi phân tích TTFT nhưng vẫn giữ 136 `cancelled`.

Tập cuối có 11.825 request dùng được cho TTFT, gồm 11.689 `ok` và 136 `cancelled` [ô 18–19]. Các kiểm tra đối soát không còn duplicate ID, không có `output_tokens <= 0`, `ttft_ms <= 0`, `ttft_ms > e2e_ms`, `prompt_tokens <= 0` hoặc `arrival_ts_ms < 0`.

Điểm em chưa trả lời được ở bước này là nguyên nhân tạo bản ghi lặp, ý nghĩa chính xác của `ttft_ms` ở request `error`, và cơ chế tạo ra một vài request lỗi có `e2e_ms - ttft_ms` rất lớn.

## 12/08/2026 — Bắt đầu Câu 2: khảo sát phân phối TTFT

Em chuyển sang tập `requests_ttft` gồm 11.825 request. Các quantile ban đầu cho thấy median = 130,90 ms, Q25 = 48,38 ms, Q75 = 210,98 ms, Q95 = 433,15 ms và max = 2036,46 ms [ô 20].

Khoảng Q25–median và median–Q75 khá gần nhau, nhưng phía giá trị lớn kéo dài xa hơn nhiều. Em chưa loại các giá trị TTFT lớn vì chưa có bằng chứng chúng là lỗi. Bước tiếp theo là quan sát hình dạng thực nghiệm bằng histogram với nhiều cách chia bin thay vì kết luận từ một biểu đồ duy nhất.

## 12/08/2026 — Hoàn thành Câu 2: cấu trúc phân phối TTFT

Em thử histogram với nhiều số bin, đếm theo các khoảng TTFT cố định và biến đổi log. Cấu trúc hai vùng vẫn xuất hiện khi thay cách chia bin: vùng `<60 ms` có 3.401 request (28,76%), vùng `60–80 ms` chỉ có 486 request (4,11%), còn vùng `>=80 ms` có 7.938 request (67,13%) [ô 36]. Vì vậy em mô tả phân phối là lệch phải, có long tail và có cấu trúc bimodal rõ.

Em từng kiểm tra xem một lognormal duy nhất có thể mô tả toàn bộ TTFT hay không. Biến đổi log vẫn không tạo ra một phân phối một đỉnh gần Normal, nên hướng mô tả bằng một lognormal duy nhất bị loại.

Em cũng thử xem traffic cao hoặc request `cancelled` có giải thích được hai mode hay không. Tỷ trọng hai vùng TTFT không thay đổi đủ mạnh trong giai đoạn traffic cao 20–35 phút, còn `cancelled` chỉ chiếm tỷ lệ nhỏ, nên hai yếu tố này không giải thích tốt cấu trúc quan sát được.

Scatter theo `prompt_tokens` cho kết quả khác nhau rõ giữa hai vùng. Với TTFT `<60 ms`, Pearson correlation prompt–TTFT khoảng -0,0243, gần như không có quan hệ tuyến tính. Với nhánh `>=80 ms`, correlation khoảng 0,8576 và fitted line có slope khoảng 0,22385 ms/token, tương đương khoảng 22,38 ms trên mỗi 100 prompt token [ô 32–33].

Em giữ giả thuyết kỹ thuật ở mức phỏng đoán: có thể tồn tại hai chế độ xử lý khác nhau, chẳng hạn một nhánh có prefix/cache reuse và một nhánh phải làm nhiều prefill hơn. Dữ liệu không có cache, prefill, batching, scheduler hay worker nên em không coi đây là kết luận.

## 13/08/2026 — Câu 3: point estimates và confidence interval

Em tính các point estimate trên 11.825 request: mean = 161,449 ms, p50 = 130,900 ms, p90 = 324,866 ms và p99 = 737,139 ms [ô 39]. Mean cao hơn median và p99 cao hơn nhiều so với p50, phù hợp với tail phải dài đã thấy ở Câu 2.

Đối với mean, em dùng standard error và xấp xỉ Normal cho sampling distribution của mean. Kết quả 95% CI là [158,756; 164,142] ms [ô 41]. Qua bước này em hiểu rõ hơn sự khác nhau giữa SD và SE: SD mô tả độ phân tán giữa request, còn SE mô tả độ bất định của mean.

Em không dùng CI hẹp của mean để kết luận mean là TTFT “điển hình”, vì một số trung bình vẫn có thể che mất cấu trúc bimodal của phân phối.

## 15/08/2026 — Bootstrap cho p50, p90, p99 và chốt Câu 3

Em dùng percentile bootstrap với 5.000 lần resample, mỗi sample có cùng kích thước 11.825 request. Kết quả: p50 có 95% CI [128,800; 133,040] ms, p90 có CI [318,280; 332,580] ms và p99 có CI [692,340; 779,844] ms [ô 42].

Relative CI width lần lượt khoảng 3,239% cho p50, 4,402% cho p90 và 11,871% cho p99 [ô 43]. Điều này cho thấy uncertainty tăng rõ khi đi sâu vào tail.

Kết luận của em cho Câu 3 là mean dễ gây hiểu nhầm nhất nếu được gọi là latency “điển hình”; p50 ít nhạy tail hơn nhưng đứng một mình vẫn che bimodality. P90 và p99 hữu ích để mô tả tail, trong đó p99 có bất định lớn hơn. Em không loại outlier chỉ để làm các khoảng tin cậy đẹp hơn.

## 16/08/2026 — Câu 4: thí nghiệm sample size cho p99

Em dùng 11.825 TTFT làm empirical population và p99 toàn bộ tập, 737,139 ms, làm reference [ô 45]. Một trial lấy ngẫu nhiên `n` request, tính sample p99 và coi là thành công nếu relative error không vượt ±10%. Em tự chọn criterion khoảng 95% simulated measurement runs thành công; đây là tiêu chí thiết kế của em, không phải đề bài cho sẵn.

Một thử nghiệm ban đầu với `n = 500` tình cờ cho sai số nhỏ hơn 10%, nhưng khi lặp 1.000 trials thì success rate chỉ còn 60,90% [ô 46–47]. Kết quả này giúp em nhận ra không thể suy ra sample size từ một lần lấy mẫu duy nhất.

Khi subsample **không hoàn lại** từ chính cửa sổ dữ liệu hiện tại, coarse/fine scan và kiểm tra nhiều seed đưa vùng ngưỡng về khoảng 2.400–2.500 request [ô 48–52]. Đây từng là kết luận tạm thời của em.

Sau đó em kiểm tra sensitivity với sampling **có hoàn lại** để mô phỏng các measurement run mới từ empirical distribution. Kết quả thay đổi đáng kể: `n = 2500` chỉ đạt 92,76%, còn `n = 3000` đạt khoảng 95,18% [ô 53]. Kiểm tra cuối qua 5 random seed cho mean success rate tại `n = 2800`, `2900`, `3000`, `3100`, `3200` lần lượt khoảng 94,30%, 94,98%, 95,24%, 95,60% và 95,99% [ô 54].

Vì vậy em chốt câu trả lời chính là **khoảng 3.000 request**, không phải “chính xác 3.000”. Kết quả khoảng 2.500 request được giữ như sensitivity khi subsample không hoàn lại và có finite-population effect.

## 17/08/2026 — Câu 5: lựa chọn đúng một hình

Em thử bốn cách biểu diễn trước khi chốt hình cuối:

- Scatter `prompt_tokens × ttft_ms` trên thang tuyến tính: giữ được quan hệ với prompt và long tail, nhưng vùng latency thấp bị nén và overplotting khá mạnh.
- Histogram `ln(TTFT)`: làm bimodality rõ hơn nhưng mất hoàn toàn thông tin về prompt length.
- Hexbin `prompt_tokens × ttft_ms`: xử lý overplotting tốt nhưng làm các request hiếm trong long tail kém nổi bật hơn và cần thêm colorbar.
- Scatter `prompt_tokens × ttft_ms` với trục TTFT logarithmic: vẫn giữ toàn bộ request, làm rõ vùng latency thấp, nhánh phía trên và long tail trong một hình.

Em kiểm tra thêm bằng cách chia prompt length thành các khoảng. Median TTFT của vùng `<60 ms` chủ yếu quanh 34–36 ms trong các khoảng có đủ quan sát, trong khi median của nhánh `>=80 ms` tăng rõ khi prompt dài hơn [ô 59]. Điều này giúp em tin rằng pattern trên hình không chỉ là hiệu ứng thị giác do dùng log scale.

Em chọn scatter với trục Y logarithmic làm `hinh.png`. Em không tô màu “fast/slow path” và không gắn nhãn cache hit/miss vì nguyên nhân kỹ thuật vẫn chưa được dữ liệu chứng minh.

## 18/08/2026 — Câu 6: dùng `tokens.csv` để loại những câu hỏi thực ra đã trả lời được

Trước khi liệt kê “điều chưa biết”, em kiểm tra `tokens.csv` thay vì mặc định dữ liệu generation không đủ. File có 1.829.816 token records thuộc đúng 11.825 request TTFT; số dòng token của từng request khớp `output_tokens - 1`, `token_index` liên tục, không duplicate và `itl_ms` đều hợp lệ [ô 63–65].

Em thử xem TTFT cao có đi cùng generation chậm hay không. Correlation giữa TTFT và median ITL chỉ khoảng 0,0003, giữa TTFT và p90 ITL khoảng 0,0016 [ô 66]. ITL theo `token_index` cũng khá ổn định, còn high-ITL rate gần 1% giữa các nhóm generation length [ô 67, ô 76–77]. Vì các câu hỏi này đã được dữ liệu trả lời ở mức association tuyến tính, em loại chúng khỏi danh sách “chưa xác định được”.

Một kiểm tra khác là tái tạo `e2e_ms` bằng `ttft_ms + sum(itl_ms)`. Residual có median khoảng 3,241 ms và p95 khoảng 8,146 ms, nhưng có 108 request residual trên 1000 ms và max khoảng 10.919,831 ms [ô 68]. Ban đầu em phỏng đoán tail này có thể do `cancelled`, nhưng kiểm tra theo status cho thấy phần lớn residual lớn thuộc request `ok`, nên giả thuyết này bị bác bỏ [ô 69].

Em cũng thử liên hệ residual lớn với giai đoạn traffic cao 20–35 phút. Ở mức aggregate, tỷ lệ residual >1000 ms cao hơn khoảng 2,05 lần [ô 72], nhưng sau khi stratify theo output length thì khác biệt không còn nhất quán [ô 84]. Vì vậy em không kết luận traffic cao gây queueing.

Cuối cùng em giữ 5 câu hỏi thực sự chưa xác định được: cơ chế tạo hai pattern TTFT; đóng góp của queueing/batching/scheduling; bản chất phần E2E residual; khả năng generalize ngoài cửa sổ một giờ; và nguyên nhân 136 request bị `cancelled`.

Điều quan trọng nhất em học được ở Câu 6 là: “dataset không có cột X” chưa phải một lập luận đủ. Cần chỉ ra observation hiện có, các mechanism khác nhau vẫn cùng giải thích được observation đó, và telemetry nào sẽ giúp phân biệt chúng.

## 19/08/2026 — Rà soát cuối trước khi viết báo cáo

Em rà soát notebook theo đúng thứ tự từ trên xuống, giữ nguyên toàn bộ code và output phân tích. Phần chỉnh sửa ở bước này chỉ tập trung vào Markdown: thống nhất cách xưng “em”, làm rõ heading Câu 1–6 và giữ nguyên các tham chiếu ô đã dùng trong quá trình phân tích.

Em cũng rà lại nhật ký để bảo đảm phần Câu 4 phản ánh đúng kết luận cuối khoảng 3.000 request thay vì dừng ở kết luận tạm khoảng 2.500 request. Các thử nghiệm không thành công hoặc giả thuyết bị bác bỏ vẫn được giữ lại vì đây là phần thể hiện quá trình học và cách em đi đến lựa chọn cuối.

Sau bước này, em chưa viết `bao_cao.md`; mục tiêu tiếp theo là chỉ bắt đầu báo cáo sau khi kiểm tra lại hai file final và xác nhận cấu trúc bài nộp.
