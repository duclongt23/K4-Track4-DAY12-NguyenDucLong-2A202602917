# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Nguyễn Đức Long |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602917 |
| GitHub | duclongt23 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 2191 / 7896 (các seed huấn luyện: 7896–7900) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,269 | 20,2 (1 lần chạy, không có seed) | BLOCK (R2, R3, R4, R6) |
| IDM | 1,513 | 0,07 (2/2883) | (baseline) |
| EgoMLP | 0,761 (0,758–0,767) | 20,1–24,6 (trung bình 22,2) | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0,624 (0,622–0,626) | 0,4–2,0 (trung bình 1,05) | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP+ (sau khi sửa) | 0,809 (±0,014 qua seed) | 0,21–0,24 (6–7 trên 2883) | RELEASE (7/7 quy tắc qua) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Cột Acc_LV không đáng tin vì tương quan với gia tốc tính từ Speed_LV chỉ 0,045, trong khi xe theo sau đạt 0,929. Vì vậy, mình dùng gia tốc tính từ tốc độ đã làm mượt. Dữ liệu có 383 quỹ đạo trùng nhau, dễ gây rò rỉ giữa train và test. Ngoài ra, các frame liên tiếp ở 10 Hz rất giống nhau nên mình chia theo quỹ đạo thay vì từng frame. Với seed 2191, tập test có 2.883 quỹ đạo (30%)

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã cho kết quả giống hoàn toàn (0/600 đoạn lệch). Bản float32 khác bit ở 600/600 đoạn nhưng không đoạn nào vượt dung sai 1e-3, vì sai khác lớn nhất chỉ 4,0e-4. Điều này cho thấy không nên so số thực bằng từng bit.

Thêm AEB làm lệch 10/600 đoạn. Ví dụ đoạn 83 chỉ lệch ở 3 frame cuối (75–77), khi TTC xuống dưới 1,5 s và AEB ép phanh −6 m/s². Phát nhanh nhất không có frame trễ hoặc rớt, nhưng phát theo thời gian thật có 581 frame trễ, 573 frame rớt, làm thay đổi 376/600 đoạn. Như vậy, chạy nhanh nhất có thể che mất lỗi xử lý không kịp thời gian.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới gồm 2.700 ô, với tốc độ 12 m/s, khoảng cách 2,76–59,3 m và tốc độ tiến gần 0–12 m/s. Vùng không thể tránh va chạm tăng từ 2,2% lên 7,4% khi độ trễ tăng từ 0 lên 0,6 s (+5,2 điểm). Kết quả mô phỏng khớp hoàn toàn với biên bất khả thi (0 ô lệch). IDM chỉ đâm ở vùng không thể tránh, còn constant-velocity và EgoMLP vẫn đâm nhiều ở vùng có thể tránh.

Log có 614 ca phanh (6,4%), mức phanh mạnh nhất của người lái khoảng 3,6 m/s², trong khi 8,6% lưới cần phanh mạnh hơn. Các ca trong log chủ yếu có tốc độ tiến gần dưới 4 m/s, nên dữ liệu chưa bao phủ tốt những tình huống nguy hiểm.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?


| Bộ điều khiển | L2 vòng hở (m) | Hạng L2 | Va chạm vòng kín (%) | Hạng va chạm |
|---|---|---|---|---|
| constant-velocity | 1,269 | 3 | 20,2 | 3 |
| IDM | 1,513 | 4 | 0,07 | 1 |
| EgoMLP | 0,761 | 2 | 22,2 (20,1–24,6) | 4 |
| PercMLP | 0,624 | 1 | 1,05 (0,4–2,0) | 2 |

Có 4 cặp đảo thứ hạng: constant-velocity/IDM, constant-velocity/EgoMLP, IDM/EgoMLP và IDM/PercMLP.

Điều đáng chú ý nhất là IDM có L2 tệ nhất nhưng lại ít va chạm nhất (2/2.883). IDM không cố bắt chước người lái mà phản ứng theo khoảng cách và tốc độ xe trước. Ngược lại, EgoMLP dự đoán khá tốt nhưng không nhìn thấy xe dẫn và khoảng cách nên không biết lúc nào cần phanh. Kết quả này cho thấy L2 thấp chưa chắc đã lái an toàn.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP có tỷ lệ va chạm dao động 20,1–24,6% qua 5 seed, chênh 4,6 điểm, dù L2 chỉ dao động 0,758–0,767. Khoảng tin cậy Wilson 95% của seed tốt nhất là [18,7; 21,6]% và seed xấu nhất là [23,1; 26,2]%, không chồng nhau.

Nếu chỉ chọn seed tốt, EgoMLP trông ngang constant-velocity (20,2%), nhưng seed xấu lại tệ hơn rõ rệt. PercMLP cũng dao động 0,38–2,05%, khiến một số seed qua R2 còn số khác thì rớt. Vì vậy, mình nghĩ không thể đánh giá mô hình chỉ bằng một lần huấn luyện mà phải xem cả độ ổn định giữa các seed.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Tỷ lệ ô của lưới cắt vào có **va chạm bất kỳ** (trước hoặc sau), %, kèm phần va chạm phía sau trong ngoặc:

| Bộ điều khiển | t=0,0: xe sau phát lại | t=0,0: xe sau phản ứng | t=0,6: xe sau phát lại | t=0,6: xe sau phản ứng |
|---|---|---|---|---|
| constant-velocity | 77,6 (sau 0,0) | 77,6 (sau 0,0) | 77,6 (sau 0,0) | 77,6 (sau 0,0) |
| IDM | 75,3 (sau 73,0) | 2,2 (sau 0,0) | 75,7 (sau 68,3) | 7,4 (sau 0,0) |
| EgoMLP | 82,1 (sau 0,0) | 82,1 (sau 0,0) | 81,3 (sau 0,0) | 81,3 (sau 0,0) |
| PercMLP | 78,0 (sau 68,4) | 12,3 (sau 0,0) | 75,0 (sau 53,0) | 23,3 (sau 0,0) |

Mình chọn T = 1,4 s, cho W1 = 0,35 so với 0,86 khi xe sau không phản ứng, tức khớp log hơn khoảng 2,5 lần.

Khi xe sau chỉ phát lại, IDM bị tính rất nhiều va chạm từ phía sau (73,0% và 68,3%) vì xe sau không phản ứng khi IDM phanh. Khi cho xe sau phản ứng, tỷ lệ va chạm của IDM giảm còn 2,2% và 7,4%. Điều này cho thấy cách mô phỏng các xe xung quanh có thể làm thay đổi hoàn toàn kết luận về độ an toàn của bộ điều khiển.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi R2–R6. Cụ thể, R2 có cận trên 2,6% (ngưỡng 1,07%), R3 là 8,9% (ngưỡng 5,6%), R4 là 15,8% (ngưỡng 2%), R5 dao động 1,7 điểm (ngưỡng 1), còn R6 là 23,3% (ngưỡng 9,4%).

Mình thấy có hai nguyên nhân chính: dữ liệu thiếu tình huống nguy hiểm và mô hình bị tích lũy sai số khi tự lái. Để sửa, mình bổ sung 197.100 cửa sổ từ 20.000 kịch bản cắt vào, rồi chạy 2 vòng DAgger để mô hình học thêm từ chính những trạng thái nó gặp phải.

Sau khi sửa, PercMLP+ qua cả 7 quy tắc: R2 còn 0,2% (cận trên 0,5%), R3 còn 0,6%, R4 còn 0,4%, R5 dao động 0,21–0,24% và R6 còn 7,9%. Tuy nhiên, nó chưa tốt hơn IDM: tỷ lệ va chạm trên log vẫn là 0,21–0,24% so với 0,07%. L2 cũng tăng từ 0,624 lên 0,809 dù mô hình an toàn hơn.

Mình chưa chọn thay IDM ngay. Trước tiên nên thử shadow mode trên dữ liệu mới, kiểm tra thêm tình huống ngoài vùng huấn luyện và giữ IDM làm phương án dự phòng.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Mình thấy bài lab vẫn có một số giới hạn quan trọng.

Thứ nhất, mô phỏng chỉ xét chuyển động dọc, chưa có đánh lái, chuyển làn hay cảm biến thật. Vì vậy, kết quả chưa phản ánh hết các rủi ro khi lái ngoài đường.

Thứ hai, hành vi các xe xung quanh vẫn là giả định. Chỉ cần thay xe sau từ phát lại sang phản ứng, tỷ lệ va chạm của IDM đã giảm từ 75,7% xuống 7,4%. Điều này cho thấy kết luận phụ thuộc khá nhiều vào cách mô phỏng.

Thứ ba, dữ liệu còn hẹp, chủ yếu là các tình huống phanh nhẹ (giới hạn khoảng 3,6 m/s²). Khi thử ở 20 m/s, PercMLP+ vẫn có thể đâm tới 2,4% ô tránh được.

Cuối cùng, kết quả chỉ dựa trên một lần chia dữ liệu và 2.883 quỹ đạo test; IDM chỉ có 2 ca va chạm nên vẫn còn bất định thống kê. Theo mình, cần thử thêm nhiều dữ liệu, nhiều seed, các tình huống phức tạp hơn và chạy shadow mode hoặc HIL trước khi cân nhắc triển khai thực tế.
