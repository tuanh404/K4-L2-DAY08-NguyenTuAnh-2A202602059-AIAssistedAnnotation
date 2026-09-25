# Vì sao chọn lô này?

Nếu chỉ có ngân sách rà năm ảnh, tôi ưu tiên năm frame sau trong 50 dòng đầu của `outputs/selection_round1.csv`. Thời điểm tính bằng giây từ đầu video.

| Ưu tiên | Frame | Hạng CSV | Điểm | Thời điểm | Lý do |
| ---: | --- | ---: | ---: | ---: | --- |
| 1 | frame_0182.jpg | 1 | 0.9591 | 72.8 | Điểm cao nhất, có 18 box mơ hồ cần rà. |
| 2 | frame_0369.jpg | 2 | 0.9324 | 147.6 | U = 0.9315, có 16 box mơ hồ; thời điểm khác ảnh đầu. |
| 3 | frame_0380.jpg | 3 | 0.9170 | 152.0 | U = 0.9340, có 15 box mơ hồ, cách frame_0369 4.4 giây. |
| 4 | frame_0326.jpg | 4 | 0.9155 | 130.4 | Có 15 box mơ hồ, cách xa các thời điểm ưu tiên ở trên. |
| 5 | frame_0331.jpg | 5 | 0.9154 | 132.4 | Có 18 box mơ hồ; cách frame_0326 đúng 2 giây, đạt khoảng cách tối thiểu. |

Tôi không thêm frame_0372.jpg vì nó chỉ cách frame_0369.jpg 1.2 giây. Camera cố định nên rà cả hai có nguy cơ lặp cùng xe và tình huống, tăng công nhưng ít thêm thông tin. Cặp frame_0326 và frame_0331 đạt khoảng cách 2 giây nhưng vẫn cần lưu ý cảnh tương tự. Năm ảnh ưu tiên đều có `empty=False`; quyết định ở đây xét ảnh gần trùng, không dựa trên trường hợp không có dự đoán.

Ba frame thuộc lô 12 ảnh được chọn là frame_0182.jpg (hạng 1, 72.8 giây, score 0.9591, U = 0.9182, 18 box mơ hồ), frame_0099.jpg (hạng 8, 39.6 giây, score 0.9063, U = 0.9460, 14 box mơ hồ) và frame_0107.jpg (hạng 14, 42.8 giây, score 0.8876, U = 0.8752, 15 box mơ hồ). Cả ba có `selected=True` trong CSV. Hai ảnh frame_0099 và frame_0107 cách nhau 3.2 giây, đạt điều kiện tối thiểu. Các số này cho thấy model còn phân vân, chưa chỉ ra khung nào thực sự sai. Cột `n_boxes` trong CSV là số dự đoán dùng để tính điểm ở ngưỡng thấp, không phải số khung pre-label đưa sang CVAT.

Frame_0372.jpg có điểm cao nhưng không chọn: hạng 6, thời điểm 148.8 giây, score 0.9101, `selected=False`. Điểm cao hơn frame_0107.jpg nhưng nó quá gần frame_0369.jpg đã chọn ở 147.6 giây. Vì vậy không thể chỉ lấy 12 điểm cao nhất mà bỏ qua khoảng cách thời gian và công rà ảnh gần trùng.

Điểm cao chỉ phản ánh độ bất định, số box mơ hồ và yếu tố thời gian theo công thức chọn mẫu. Nó không chứng minh nhãn gợi ý sai hay bảo đảm sửa ảnh đó sẽ cải thiện mô hình. Kết quả còn phụ thuộc tính nhất quán của nhãn, độ đa dạng dữ liệu và quá trình huấn luyện. Thực tế AP50 vòng 1 giảm từ 0.7714 xuống 0.3458 trên cùng tập test; cần đánh giá sau train thay vì suy ra hiệu quả từ điểm chọn ảnh.
