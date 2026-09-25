# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Tú Anh

Công cụ gán nhãn đã dùng: CVAT

## 1. Dữ liệu và cách chia tập

Dữ liệu là ảnh đường cao tốc ban đêm từ camera cố định. Các ảnh gần nhau theo thời gian thường có cùng xe xuất hiện trong vài giây. Pool và test được chia theo thời gian, có vùng đệm nhằm giảm việc cùng xe xuất hiện ở cả hai tập. Theo `data/DATA.md`, có 268 ảnh pool, 20 ảnh test, 112 ảnh vùng đệm bị loại; ảnh pool gần test nhất vẫn cách 4.4 giây.

Nếu chia ngẫu nhiên, mô hình có thể học một chiếc xe rồi được chấm trên ảnh gần như giống hệt của xe đó. Rò rỉ dữ liệu khiến điểm test có thể cao hơn khả năng nhận diện tình huống mới. Tôi giữ nguyên test, chỉ dùng 12 ảnh được chọn từ pool để sửa nhãn và train vòng 1.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 chép từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Theo `outputs/metrics_round0.json`, AP50 chính xác là 0.7714. Recall xe nhỏ 0.1818 thấp hơn xe vừa 0.5473 và xe lớn 0.5610, cho thấy xe nhỏ, thường ở xa, bị bỏ sót nhiều hơn. Tại confidence 0.25 có 197 TP, 16 FP và 206 FN so với tham chiếu.

Trong `outputs/compare_round0.jpg`, frame_0350 có khung đỏ ở cụm xe bên trái gần vùng đèn pha sáng, rộng hơn các khung tham chiếu riêng trong khu vực đó. Cần kiểm tra khung có ôm vùng sáng hoặc nhiều xe hay không. Ở frame_0250, mép phải có khung tham chiếu bao vùng rất tối và phần xe bị cắt; cần phóng to để xác nhận vật thể và biên khung trước khi kết luận model bỏ sót. Nhãn test do mô hình tạo, chưa được người rà từng box, nên không khớp tham chiếu chưa chắc là dự đoán sai. Tôi ghi nhận nghi vấn, không sửa nhãn test.

## 3. Chiến lược chọn mẫu

Công thức là `score = 0.5·U + 0.3·A + 0.2·D`. Theo `tools/al_select.py`, U là trung bình độ bất định của tối đa năm box phân vân nhất; confidence gần 0.5 cho độ bất định cao. A là số box có confidence từ 0.15 đến dưới 0.50, chuẩn hóa theo số lớn nhất trong pool. D là khoảng cách tới ảnh đã gán gần nhất, giới hạn ở 10 giây rồi chuẩn hóa. Vòng đầu chưa có ảnh đã gán nên D = 1 cho mọi ảnh, chưa phân biệt thứ hạng giữa các ảnh.

`MIN_GAP_S = 2.0` yêu cầu hai ảnh được chọn trong lô cách nhau ít nhất 2 giây, giảm việc rà các ảnh gần trùng. Điều này không bảo đảm ảnh cách đúng 2 giây đã hoàn toàn khác nhau.

Trong `reports/SELECTION.md`, tôi dẫn frame_0182.jpg (hạng 1, 72.8 giây, điểm 0.9591, 18 box mơ hồ), frame_0099.jpg (hạng 8, 39.6 giây, điểm 0.9063, 14 box mơ hồ) và frame_0107.jpg (hạng 14, 42.8 giây, điểm 0.8876, 15 box mơ hồ). Cả ba được chọn. Frame_0372.jpg có điểm 0.9101, hạng 6, nhưng không chọn vì cách frame_0369.jpg đã chọn chỉ 1.2 giây. Rà nhiều box mơ hồ cần thêm công, nên tránh ảnh gần trùng giúp sử dụng thời gian hợp lý hơn.

Điểm bất định không chứng minh ảnh đó sẽ cải thiện mô hình. Dự đoán phân vân có thể liên quan tới ánh sáng, biên xe khó xác định hoặc nhãn không nhất quán. Phải đánh giá sau train trên cùng test để biết kết quả.

## 4. Các vòng học chủ động (active learning)

Bảng chép từ `reports/rounds_table.md`. Test có 20 ảnh, 403 box tham chiếu được tính điểm, bỏ qua 14 box cao dưới 16 pixel. Ngưỡng IoU là 0.5; P, R và F1 tính tại confidence 0.25.

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 312 | 0.346 | -0.426 | 1.000 | 0.012 | 0.025 | 0.000 | 0.007 | 0.073 |

Vòng 0 chưa train bằng nhãn tôi sửa. Vòng 1 dùng 12 ảnh, 312 box, train 50 epochs với kích thước ảnh 960. Theo `outputs/round1_diff.md`, từ 169 box gợi ý có 143 accepted, 13 edited, 13 deleted và 156 added. Tổng sau sửa là 143 + 13 + 156 = 312 box. Đây là thống kê đối chiếu hình học giữa hai bộ nhãn, không phải bản ghi từng thao tác chuột trong CVAT.

AP50 giảm từ 0.7714 xuống 0.3458, giảm 0.4256 (42.56 điểm phần trăm). Vì mới có một vòng học lại, mức giảm so với vòng trước cũng là so với cold start. Recall xe nhỏ giảm 0.1818 xuống 0; xe vừa 0.5473 xuống 0.0068; xe lớn 0.5610 xuống 0.0732. Không nhóm nào cải thiện recall. Precision vòng 1 bằng 1.0 nhưng chỉ có 5 TP, 0 FP, 398 FN tại confidence 0.25; không thể dựa riêng precision để kết luận tốt hơn. F1 giảm từ 0.6396 xuống 0.0245.

Trong `outputs/compare_round1.jpg`, frame_0050 có cold start TP = 11, FP = 2, FN = 7, còn vòng 1 TP = 0, FP = 0, FN = 18. Xe lớn bị cắt ở mép dưới ảnh có khung khớp màu xanh ở cold start nhưng thành khung tham chiếu bị bỏ sót màu vàng ở vòng 1. Số trên ảnh đã áp dụng luật bỏ qua box quá nhỏ. Kết quả xấu đi kể cả với xe gần. Một giả thuyết cần kiểm tra là confidence sau train giảm khiến nhiều box không vượt ngưỡng 0.25; phải xem dự đoán gốc để xác nhận. Cũng cần kiểm tra nhãn train, ánh xạ lớp và cấu hình train, chưa thể khẳng định nguyên nhân chỉ từ bảng điểm.

Ba loại bằng chứng cần phân biệt: `BLIND_SCAN.md` ghi quan sát độc lập ở frame_0107.jpg là 27 xe, cụm ba đèn xa khó phân biệt một hay hai xe và hai đuôi xe ở mép ảnh. `REVIEW_LOG.csv` ghi thêm khung ở mép dưới frame_0107.jpg, chỉnh rộng car39 ở frame_0182.jpg và xóa một khung ở frame_0227.jpg. `round1_diff.md` thống kê nhãn sau sửa, trong đó frame_0107 có 23 box; khác ước lượng 27 xe bằng mắt nên cần đối chiếu lại ca khó, không sửa bản quét đã khóa cho khớp. Các file metrics và ảnh compare mới phản ánh mô hình sau train; thêm nhiều box không bảo đảm điểm tăng.

Ca khó là xe bị cắt ở mép: theo `GUIDELINE_LABEL.md`, chỉ khoanh phần còn trong ảnh, không vẽ bù phần khuất hoặc ôm vệt đèn phản chiếu. Với xe sát nhau phải dùng từng khung riêng. Ở frame_0227, nhật ký ghi xóa khung phía sau xe bên trái vì “AI gán 3 xe thành 1 box”. Đây là lỗi gộp nhiều xe vào một khung, trái quy tắc mỗi xe một box; khi rà lại cần kiểm tra các xe trong cụm đều có khung riêng phù hợp.

## 5. Kết luận và giới hạn

Vòng 1 chưa cải thiện: AP50 giảm 0.4256, recall giảm cả ba nhóm kích thước, FN tại ngưỡng 0.25 tăng từ 206 lên 398. Tôi chọn dừng train thêm để rà nguyên nhân trước. Cần kiểm tra box mới có ôm thân xe hay ánh sáng, mỗi xe có khung riêng, lớp đúng `car`/class 0, tọa độ và tên ảnh khớp, nhãn xuất CVAT đầy đủ, rồi đối chiếu cấu hình và log train. Frame_0227 và chênh lệch số xe ở frame_0107 là hai điểm cụ thể cần rà. Đây là việc đề xuất, chưa phải kết quả chẩn đoán đã hoàn thành.

Nếu làm vòng sau, tôi ưu tiên xe rất xa chỉ thấy cụm đèn và xe bị che hoặc cắt mép ảnh. Ca đầu cần phóng to để phân biệt xe với phản chiếu, tốn công và dễ không thống nhất; ca sau cần rà kỹ biên phần thân nhìn thấy. Chọn thời điểm cách nhau để giảm ảnh gần trùng; 2 giây chỉ là mức tối thiểu, vẫn cần xem cảnh có lặp hay không. Chưa đo thời gian rà thực tế nên không đưa số phút như kết quả đã đo.

Test chỉ có 20 ảnh của một camera, không đại diện mọi cảnh giao thông ban đêm. Bỏ qua box dưới 16 pixel khiến số đo không phản ánh đầy đủ xe rất nhỏ. Nhãn tham chiếu do mô hình tạo chưa được rà cũng có thể sai về số xe hoặc biên khung. Vì vậy cần xem cả ảnh, nhật ký và số đo; không sửa test hay sửa tay metrics để làm điểm tốt hơn.
