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

Bước 0 cho thấy ba vấn đề. **(1)** Cột `Acc_LV` không dùng được: tương quan với d(`Speed_LV`)/dt chỉ 0,045, trong khi cột gia tốc của xe theo sau là 0,929. Mình thay bằng gia tốc dẫn xuất từ `Speed_LV` đã làm mượt (cửa sổ 5 bước, đạo hàm trung tâm, `a_lead`) và dùng nó để khai thác ca phanh. **(2)** Có 383 quỹ đạo trùng hệt nhau ở các id xa nhau (9.993 → 9.610 quỹ đạo sạch); nếu để nguyên, một bản sao của quỹ đạo huấn luyện có thể rơi vào tập test và làm điểm test đẹp giả (rò rỉ dữ liệu). **(3)** Log 10 Hz, mỗi quỹ đạo 70–109 bước (trung vị 106), các khung liền kề gần như giống hệt nhau, nên phải chia theo **quỹ đạo** chứ không theo khung: nếu chia theo khung thì khung kề nhau nằm ở cả train lẫn test. Mình dùng split seed 2191, giữ lại 2.883 quỹ đạo để test (30 %). Hai điều đáng nhớ cho các câu sau: `Spatial_Gap` khớp đúng Pos_LV − Pos_FAV − 4,5 m (lệch tối đa 3,6e-11 m) nên đáng tin; còn xe dẫn trong log phanh rất nhẹ (gia tốc đã làm mượt ở phân vị 0,1 chỉ là −2,77 m/s²).

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã: 0/600 đoạn khác, 0 đoạn không-giống-bit. Bản refactor float32 (đổi thứ tự phép tính): **600/600** đoạn không giống từng bit (lệch tối đa 4,0e-4 m/s²) nhưng **0/600** đoạn vượt dung sai (atol = rtol = 1e-3). Lý do: phép cộng số thực không có tính kết hợp ((0,1+0,2)+0,3 = 0,6000000000000001 còn 0,1+(0,2+0,3) = 0,6), nên so từng bit sẽ báo lỗi giả; dung sai 1e-3 lớn hơn sai khác do thứ tự phép tính (4,0e-4) nhưng nhỏ hơn nhiều so với một thay đổi hành vi thật (1,98 m/s²). Thêm AEB (TTC < 1,5 s thì phanh −6 m/s²) làm lệch **10/600** đoạn (83, 141, 188, ...). Mở đoạn 83: chỉ 3/78 khung (75–77, cuối đoạn) lệch; ở đó gap 5,0 → 4,4 m, tiến gần ~3,4 m/s, TTC 1,48 → 1,44 s < 1,5, lệnh IDM cũ đã là −4,4 và AEB ép thành −6,0. Mọi khung còn lại có TTC lớn hơn nên AEB không kích hoạt và hai bản giống nhau, vì vậy AEB chỉ lệch ở khung TTC thấp. Phát nhanh nhất: 0 trễ, 0 rớt trên 59.460 khung, 0 đoạn đổi. Phát theo thời gian thật: **581 khung trễ + 573 khung rớt**, 376/600 đoạn đổi (lệch tới 5,9 m/s²). Nhanh nhất không có hạn chót nên một khung tốn 140 ms vẫn "xong"; thời gian thật thì khung đó trễ, các khung đến lúc bộ điều khiển còn bận bị rớt và phải giữ lệnh cũ, nên lỗi trễ hạn chót chỉ lộ ra ở thời gian thật.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới: tốc độ cruise 12 m/s (do MSSV), khoảng cách 2,76–59,3 m (60 giá trị, p2–p98 của log) × tiến gần 0–12 m/s (45 giá trị) = **2.700 ô**. Không ai tránh được: **2,2 %** ô khi độ trễ 0 và **7,4 %** khi 0,6 s; bộ mô phỏng được kiểm chứng bằng xe phanh hết cỡ (9 m/s²) đâm đúng 2,2 % / 7,4 % ô, 0 ô lệch ở cả hai độ trễ (đường đỏ trùng đường cyan trong `scenarios_idm.png`). Vùng đâm của IDM tăng từ 2,2 % lên 7,4 % (+5,2 điểm, gấp 3,3 lần), và nó trùng hoàn toàn vùng bất khả thi (đâm mà tránh được = 0,0 % ở cả hai độ trễ): IDM chỉ đâm ở chỗ ai cũng đâm. Ngược lại constant-velocity đâm 75,4 % / 70,2 % ô tránh được và EgoMLP 79,9 % / 73,9 % (seed xấu nhất): đó mới là thất bại của bộ điều khiển. Log **không** phủ vùng khó: khai thác được 614 ca xe dẫn phanh (6,4 % của 9.610 quỹ đạo), phanh mạnh nhất trung vị −2,5, p99 −4,2 m/s²; người lái theo sau phanh mạnh nhất cỡ **3,6 m/s²** (phân vị 99,9) trong khi bộ chấp hành làm được 9, và **8,6 %** lưới đòi phanh mạnh hơn 3,6. Các chấm trắng từ log dồn ở góc tiến gần ≲ 4 m/s, gần như trống ở tiến gần > 6 m/s.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?


| Bộ điều khiển | L2 vòng hở (m) | Hạng L2 | Va chạm vòng kín (%) | Hạng va chạm |
|---|---|---|---|---|
| constant-velocity | 1,269 | 3 | 20,2 | 3 |
| IDM | 1,513 | 4 | 0,07 | 1 |
| EgoMLP | 0,761 | 2 | 22,2 (20,1–24,6) | 4 |
| PercMLP | 0,624 | 1 | 1,05 (0,4–2,0) | 2 |

Bốn cặp đổi thứ tự: (constant-velocity, IDM), (constant-velocity, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). Nổi bật nhất là IDM: **hạng 4 về L2 nhưng hạng 1 về va chạm**. L2 đo việc đoán đúng tốc độ của người lái trong 3 s tới; IDM có tham số cố định (T = 1,2 s, trong khi người lái giữ thời khoảng trung vị 1,85 s) nên đoán người lái kém, nhưng nó là luật vật lý phản ứng với khoảng cách và tốc độ tiến gần, nên khi tự lái chỉ đâm 2/2883 trường hợp. Đoán giống người không phải là an toàn. EgoMLP có L2 (0,761) tốt hơn constant-velocity (1,269) vì nó nắm được quán tính và xu hướng tốc độ của chính xe, nhưng **không thấy khoảng cách và xe dẫn** nên không có lý do để phanh: phanh gắt trung vị của nó chỉ 0,08 m/s² (IDM 0,9; PercMLP 0,52), kết quả đâm 22,2 %, tệ hơn cả constant-velocity (20,2 %).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP dao động **20,1–24,6 %** va chạm qua 5 seed (20,33; 23,14; 22,72; 20,08; 24,63), rộng 4,6 điểm, trong khi L2 vòng hở gần như đứng yên (0,758–0,767, độ lệch chuẩn 0,003). Khoảng tin cậy Wilson 95 % (n = 2.883) của seed tốt nhất (7899) là [18,7; 21,6] % và của seed xấu nhất (7900) là [23,1; 26,2] %: **không chồng nhau**, nên chênh lệch này là khác biệt thật giữa các lần huấn luyện chứ không phải nhiễu lấy mẫu. Nếu chỉ chạy seed 7899, mình sẽ báo "EgoMLP (20,1 %) ngang hoặc hơn constant-velocity (20,2 %, CI [18,8; 21,7])"; chỉ chạy seed 7900 thì lại báo "tệ hơn rõ rệt". Với PercMLP còn nguy hiểm hơn: 0,38–2,05 % (CI [0,21; 0,68] và [1,59; 2,63]). Quy tắc R2 cho phép cận trên ≤ 0,07 + 1 = 1,07 %, nên chỉ chạy seed 7897 (cận trên 0,68 %) hoặc 7899 (0,94 %) thì PercMLP qua R2, còn 7896, 7898, 7900 thì rớt. Chỉ một lần chạy sẽ khiến quyết định phát hành phụ thuộc vào vận may của seed; vì vậy cổng lấy seed xấu nhất và có thêm R5 (độ ổn định qua seed).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Tỷ lệ ô của lưới cắt vào có **va chạm bất kỳ** (trước hoặc sau), %, kèm phần va chạm phía sau trong ngoặc:

| Bộ điều khiển | t=0,0: xe sau phát lại | t=0,0: xe sau phản ứng | t=0,6: xe sau phát lại | t=0,6: xe sau phản ứng |
|---|---|---|---|---|
| constant-velocity | 77,6 (sau 0,0) | 77,6 (sau 0,0) | 77,6 (sau 0,0) | 77,6 (sau 0,0) |
| IDM | 75,3 (sau 73,0) | 2,2 (sau 0,0) | 75,7 (sau 68,3) | 7,4 (sau 0,0) |
| EgoMLP | 82,1 (sau 0,0) | 82,1 (sau 0,0) | 81,3 (sau 0,0) | 81,3 (sau 0,0) |
| PercMLP | 78,0 (sau 68,4) | 12,3 (sau 0,0) | 75,0 (sau 53,0) | 23,3 (sau 0,0) |

`T` hiệu chỉnh là **1,4 s** (khoảng cách W1 giữa phân bố thời khoảng mô phỏng và log là 0,35; xe không bao giờ phản ứng là 0,86; T = 1,2 cho 0,347, gần như ngang), xe sau cách 18,8 m và trễ phản ứng 0,6 s. Vậy xe sau chân thực hơn xe không phản ứng khoảng 2,5 lần về thời khoảng, nhưng W1 vẫn khác 0. "Va chạm giả": xe sau phát lại giữ tốc độ log bất kể xe thử làm gì, nên khi IDM phanh gấp để khỏi đâm xe phía trước thì bị tính đâm từ phía sau (73,0 % / 68,3 % ô), một việc xe thật có thể phản ứng để tránh. constant-velocity và EgoMLP không phanh nên không bị tính đâm sau. Hệ quả: ở thế giới phát lại IDM (75,3 / 75,7 %) trông **ngang constant-velocity (77,6 %)**, còn ở thế giới phản ứng IDM chỉ 2,2 % / 7,4 % (đúng bằng vùng bất khả thi), kết luận "IDM tốt nhất" chỉ hiện ra khi các tác nhân khác phản ứng. Vì thế R6 dùng thế giới phản ứng.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi **R2** (seed xấu nhất 2,0 %, cận trên 2,6 % so với 0,07 + 1 = 1,07 %), **R3** (4,6 %, n = 173, cận trên 8,9 % so với 0,6 + 5 = 5,6 %), **R4** (15,8 % ô tránh được mà vẫn đâm so với 0,0 + 2 %), **R5** (0,4–2,0 %, rộng 1,7 điểm > 1) và **R6** (23,3 % so với 7,4 + 2 %). Có hai nguyên nhân gốc. *(a) Kịch bản log không chứa:* người lái trong log phanh ≤ 3,6 m/s² và 8,6 % lưới đòi hơn thế, nên mô hình bắt chước chưa thấy cắt vào gần, nhanh; thể hiện ở R4 và R6. *(b) Sai số tích luỹ khi tự lái:* ngay trên log không lạ, nó đâm 0,4–2,0 % với biến thiên gấp 5 lần giữa các seed; mô hình tự đi vào trạng thái người lái chưa từng đến, sai nhỏ cộng dồn (R2, R3, R5). Cách sửa: thêm 197.100 cửa sổ từ 20.000 cắt vào mô phỏng do IDM gắn nhãn (chữa (a)) và 2 vòng DAgger (cho mô hình tự lái trên log huấn luyện, gắn nhãn bằng dự báo IDM ở chính các trạng thái nó tự đi vào; chữa (b)). Kết quả: R2 0,2 % (cận trên 0,5 %), R3 0,6 %, R4 0,4 %, R5 0,21–0,24 %, R6 7,9 % so với 7,4 %, tức **7/7 quy tắc qua**. Nhưng PercMLP+ **không tốt hơn** IDM, chỉ "không kém hơn trong biên": trên log 0,21–0,24 % (6–7 ca) so với IDM 0,07 % (2 ca), CI xấu nhất [0,12; 0,50] so với [0,02; 0,25] chồng nhau; lưới 0,4 % so với 0,0 %; thế giới phản ứng 7,9 % so với 7,4 %; có nhiễu cảm biến 0,28–0,31 % so với 0,07 %; ở 20 m/s (ngoài dải huấn luyện ≤ 16 m/s) 0,3–2,4 % so với IDM 0,0 % (PercMLP chưa sửa: 24,4–30,6 %). Nó học từ nhãn của IDM nên trần của nó là thầy. Mình **chưa nên thay IDM**: L2 của nó còn xấu đi từ 0,624 lên 0,809 trong khi va chạm giảm, càng cho thấy điểm vòng hở không nói gì về an toàn. Trước khi quyết định, mình đề xuất chạy chế độ bóng (shadow mode) trên log thật mới, bổ sung kịch bản ngoài dải tốc độ, và giữ IDM làm phương án dự phòng; chỉ thay khi có lợi ích khác IDM mà bằng chứng đo được.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

**(1) Chỉ một chiều dọc, không cảm biến thật.** Không có chuyển làn, đánh lái, nhiều xe, xe cắt vào là mô hình lý tưởng (xuất hiện tức thì, giữ nguyên tốc độ), và tri giác chỉ là nhiễu Gauss (1 m, 0,5 m/s) với bộ chấp hành lý tưởng; rủi ro thật nằm ngoài những giả định này nên "qua cổng" không suy ra an toàn. **(2) Thế giới mô phỏng là giả định của chúng ta.** Xe dẫn trong log không phản ứng với xe thử, xe sau là IDM chỉ hiệu chỉnh theo phân bố thời khoảng (T = 1,4 s, W1 0,35 ≠ 0, trễ 0,6 s tự chọn); chỉ đổi giả định về xe sau đã đưa IDM từ 75,7 % xuống 7,4 %, nên mọi kết luận về va chạm phụ thuộc giả định đó. **(3) Dữ liệu hẹp.** Một bộ log đô thị, tốc độ xe dẫn p99 chỉ 15,6 m/s, người lái phanh ≤ 3,6 m/s², 614 ca phanh nhẹ, lưới ở một tốc độ 12 m/s; ở 20 m/s PercMLP+ đâm tới 2,4 % ô tránh được, tức vượt biên R4 (2 điểm), cho thấy kết quả không khái quát ra ngoài vùng đã thấy. **(4) Thống kê và ngưỡng.** Một lần chia, 2.883 quỹ đạo test, IDM chỉ 2 va chạm và PercMLP+ 6–7, nên khoảng tin cậy rộng (cận trên 3,2 % trên 173 ca phanh); biên 1 điểm gấp khoảng 14 lần tỷ lệ của IDM (0,07 %) và là lựa chọn của đội, không phải sự thật vật lý. Để thu hẹp khoảng cách với thế giới thật cần: log từ nhiều thành phố và tốc độ, kịch bản ngang và nhiều xe với xe dẫn có phản ứng, mô hình tri giác và độ trễ thật, chạy bóng trên xe thật rồi kiểm tra trên bãi thử/HIL, nhiều lần chia và nhiều seed hơn, cùng bộ giám sát an toàn dự phòng khi phát hành dần.
