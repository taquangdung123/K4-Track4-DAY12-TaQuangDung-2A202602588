# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Tạ Quang Dũng |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602588 |
| GitHub | taquangdung123 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split 1537 / model 3398, 3399, 3400, 3401, 3402 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.266 | 19.7 | BLOCK (R2, R3, R4, R6) |
| IDM | 1.542 | 0.1 | (baseline) |
| EgoMLP | 0.763 ±0.001 | 12.9 .. 36.5 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0.621 ±0.002 | 0.3 .. 2.4 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP+ (sau khi sửa) | 0.829 | 0.3 .. 0.3 | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Dữ liệu thô có 992.679 dòng, 9.993 quỹ đạo; 383 quỹ đạo trùng hệt nhau bị loại, còn 9.610. Cột `Acc_LV` gần như vô dụng: tương quan với đạo hàm của `Speed_LV` chỉ 0,045, trong khi cột gia tốc của xe sau khớp 0,929. Vì vậy em dùng gia tốc xe dẫn tự tính từ `Speed_LV` đã làm mượt (`a_lead`) để khai thác ca phanh. Các bản trùng nằm ở những id cách xa nhau, nên chia ngẫu nhiên có thể đưa cùng một quỹ đạo vào cả train lẫn test và làm điểm test đẹp giả. Chia theo **quỹ đạo** (không theo khung hình) vì các khung liên tiếp ở 10 Hz gần như giống nhau; chia theo khung thì mô hình đã "thấy" gần đúng cửa sổ test khi huấn luyện. Ngoài ra `gap` khớp với hiệu vị trí trừ chiều dài xe 4,5 m (sai lệch tối đa 3,6e-11 m), nên cột gap đáng tin.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã: 0/600 đoạn khác, giống từng bit. Bản refactor float32 khác bit ở cả 600/600 đoạn nhưng 0 đoạn vượt dung sai (atol = rtol = 1e-3, lệch lớn nhất 1,2e-4). Đó là do làm tròn float32 và thứ tự cộng ((0,1+0,2)+0,3 = 0,6000000000000001 ≠ 0,6), không phải do đổi hành vi, nên so `==` là phép thử sai. Thêm AEB làm lệch 6/600 đoạn (ví dụ 129, 253, 310, 331, 547). Mở đoạn 129: chỉ 4 khung 82–85 lệch, đúng lúc TTC rơi xuống dưới 1,5 s (gap 5,11 m, closing 3,42 m/s, TTC 1,49 s → 1,36 s); lệnh đổi từ −4,44 thành −6,0 m/s² vì AEB kích hoạt. Phát "fastest" có 0 khung trễ/rớt vì không có hạn chót; phát theo thời gian thật có 589 khung trễ + 578 khung rớt trên 60.410 khung, làm 380/600 đoạn đổi kết quả. Đó là lỗi thời gian mà phát nhanh nhất che mất.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới cắt vào ở tốc độ cruise 12,0 m/s: 60 khoảng cách (p2..p98 của gap trong log) × 45 tốc độ tiến gần (0..12 m/s) = 2.700 ô. Với t_react = 0 s có 2,2 % ô bất khả thi; với 0,6 s tăng lên 7,4 %. Vùng đâm của IDM tăng đúng 2,2 % → 7,4 % và trùng khít biên vật lý (0 ô lệch ngoài biên ở cả hai độ trễ), nghĩa là IDM chỉ đâm ở nơi không ai tránh được. Người lái trong log phanh mạnh nhất cỡ 3,6 m/s² (phân vị 99,9), trong khi bộ chấp hành làm được 9 m/s²; 9 % lưới đòi phanh mạnh hơn mức đó. Em khai thác được 614 ca xe dẫn phanh (6,4 % số quỹ đạo, đỉnh giảm tốc p50 −2,5, p99 −4,2 m/s²), nhưng chúng tập trung ở vùng dễ; vùng gap nhỏ, closing lớn của lưới gần như không có trong log.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở | hạng | Va chạm vòng kín | hạng |
|---|---|---|---|---|
| const-vel | 1,266 m | 3 | 19,7 % | 3 |
| IDM | 1,542 m | 4 | 0,1 % | 1 |
| EgoMLP | 0,763 m | 2 | 24,3 % | 4 |
| PercMLP | 0,621 m | 1 | 1,4 % | 2 |

Các cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất vì nó không bắt chước người lái mà giữ khoảng cách theo luật riêng, thận trọng hơn; nhưng nó phản ứng với gap và tốc độ xe dẫn nên trong vòng kín gần như không đâm (0,1 %). EgoMLP chỉ nhìn tốc độ của chính nó: dự báo trơn nên L2 tốt hơn const-vel, nhưng nó không thấy khoảng cách hay việc xe dẫn phanh. Vì vậy khi tự lái nó không biết lúc nào cần phanh (67,0 % va chạm trên lát cắt xe dẫn phanh).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP dao động từ 12,9 % đến 36,5 % va chạm giữa 5 seed, gần gấp ba lần. Với n = 2.883 quỹ đạo, khoảng Wilson 95 % của seed xấu nhất là [34,8 %, 38,3 %], rất hẹp; seed tốt nhất (12,9 %) nằm xa ngoài khoảng đó, nên khoảng tin cậy của hai seed không chồng nhau. Nghĩa là sai khác giữa các seed là thật (đến từ huấn luyện), không phải nhiễu thống kê của tập test: khoảng tin cậy của một seed không phản ánh độ bất định do huấn luyện. Nếu chỉ chạy một seed, em có thể báo "12,9 %" hoặc "36,5 %" tuỳ may rủi; báo seed đẹp sẽ đánh giá thấp rủi ro gần 3 lần. Vì vậy cổng dùng seed xấu nhất (R2) và yêu cầu dải ≤ 1 điểm (R5).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Người lái trong log giữ thời khoảng cách trung vị 1,87 s. Em hiệu chỉnh IDM của xe sau theo khoảng cách Wasserstein-1 tới phân bố thời khoảng cách: T = 1,4 s cho W1 = 0,35, trong khi xe sau không phản ứng cho 0,87, nên xe sau đã hiệu chỉnh chân thực hơn rõ rệt.

| t_react = 0,6 s | xe sau phát lại | xe sau phản ứng | va chạm phía sau (phát lại / phản ứng) |
|---|---|---|---|
| const-vel | 77,6 % | 77,6 % | 0,0 % / 0,0 % |
| IDM | 75,7 % | 7,4 % | 68,3 % / 0,0 % |
| EgoMLP | 80,8 % | 80,8 % | 0,0 % / 0,0 % |
| PercMLP | 76,2 % | 27,8 % | 49,0 % / 0,0 % |

Khi xe sau chỉ phát lại, nó không biết IDM vừa phanh để tránh xe cắt vào nên đâm vào đuôi: 68,3 % lưới là "va chạm giả". Trong thế giới đó IDM (75,7 %) trông tệ ngang const-vel (77,6 %), vì const-vel không phanh nên không bị đâm đuôi. Khi xe sau phản ứng, va chạm phía sau về 0 và IDM chỉ còn 7,4 %, đúng bằng vùng bất khả thi. Kết luận đảo ngược hoàn toàn: phát lại trừng phạt chính hành vi phanh an toàn.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn ở R2, R3, R4, R5, R6, với hai nguyên nhân gốc:
- **Kịch bản log không chứa:** trên lưới cắt vào nó đâm 20,4 % diện tích tránh được (R4, baseline 0,0 %) và 27,8 % khi xe sau phản ứng (R6, baseline 7,4 %).
- **Sai số tích luỹ khi tự lái:** trên log, seed xấu nhất có 2,4 % va chạm (cận trên 3,1 % > 0,1 % + 1 điểm, R2), và dải qua các seed là 0,3..2,4 % (R5).

Em sửa bằng hai cách:
- **Thêm kịch bản cắt vào:** 197.130 cửa sổ từ 20.000 cảnh cắt vào mô phỏng, nhãn do IDM, lấp đúng vùng log thiếu.
- **DAgger 2 vòng:** gắn nhãn IDM tại chính các trạng thái mô hình tự đi vào, dạy nó hồi phục từ sai số của mình.

Sau khi sửa, PercMLP+ đạt R2 0,3 % (cận trên 0,6 %), R4 1,9 %, R5 0,3..0,3 %, R6 9,3 % và được RELEASE; ở tốc độ chưa thấy 20 m/s nó còn đâm 0,1..1,3 % lưới, so với 25,1..38,6 % của PercMLP. Nhưng nó **không tốt hơn** IDM: vẫn kém ở mọi chỉ số (0,3 % so với 0,1 %; 1,9 % so với 0,0 %; 9,3 % so với 7,4 %), và vì học từ IDM nên khó vượt thầy. L2 vòng hở lại tăng từ 0,621 lên 0,829 m dù lái an toàn hơn, thêm một bằng chứng rằng L2 không đo an toàn. Em đề xuất chưa thay IDM: chạy PercMLP+ ở shadow mode bên cạnh IDM, thử thêm kịch bản (tốc độ cao, nhiều xe, nhiễu cảm biến), và chỉ phát hành nếu nó mang lại lợi ích rõ ràng (êm hơn, giống người hơn) mà an toàn không kém.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(1) **Chỉ một chiều dọc, không cảm biến thật**: không có chuyển làn, rẽ hay người đi bộ; nhiễu cảm biến chỉ là ±1 m / ±0,5 m/s giả định, nên tỷ lệ 0,3 % không nói gì về lỗi nhận thức thật như che khuất hay nhận nhầm. (2) **Xe dẫn không phản ứng và thế giới mô phỏng là giả định của ta**: biên bất khả thi dùng b_max = 9 m/s², xe cắt vào giữ tốc độ, xe sau là IDM với T = 1,4 s; nếu các giả định này sai thì việc phân loại "bất khả thi" hay "lỗi bộ điều khiển" cũng sai. (3) **Phân phối dữ liệu hẹp**: 9.610 quỹ đạo dài 7–11 s, tốc độ p99 ≈ 15,6–17,8 m/s, chỉ từ Argoverse 2; ở 20 m/s PercMLP đã sụp, chứng tỏ còn nhiều vùng chưa thấy. (4) **Cỡ mẫu**: 0,3 % trên 2.883 quỹ đạo không đủ chứng minh rủi ro cỡ 1e-6 mỗi giờ ở thực tế. Việc cần làm tiếp: kiểm thử trên log đa dạng hơn (thời tiết, đô thị, cao tốc), mô phỏng hai chiều với tác nhân phản ứng học từ dữ liệu, kiểm thử phần cứng trong vòng (HIL), lái bóng (shadow mode) và thử trên đường kín trước khi lên đường thật.
