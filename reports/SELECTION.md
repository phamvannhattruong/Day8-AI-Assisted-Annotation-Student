# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box: 
- 1) `frame_0182.jpg` — score 0.9591, t = 72.8 s, rank 1. Đây là khung rõ nhất trong top 50: nhiều xe sáng rõ, ít box mơ hồ và có tỷ lệ dự đoán cao nhất, nên là anchor cho tuyến đường ban đêm.
- 2) `frame_0369.jpg` — score 0.9324, t = 147.6 s, rank 2. Cảnh này có nhiều xe và độ chồng lớp nhất định nhưng vẫn giữ score cao, đồng thời cho thấy model ổn với lưu lượng giao thông cao ở cuối video.
- 3) `frame_0380.jpg` — score 0.9170, t = 152.0 s, rank 3. Chúng ta chọn khung này vì nó nằm trong cùng chuỗi thời gian gần với `frame_0369` và `frame_0372`, nên hợp để kiểm tra ảnh gần trùng và tránh chọn quá nhiều frame cùng một cú quay.
- 4) `frame_0326.jpg` — score 0.9155, t = 130.4 s, rank 4. Có `n_boxes = 39`, `n_ambiguous = 15`, cho thấy một cảnh khá đông nhưng vẫn rõ, nên rất thích hợp để biểu diễn tình huống traffic density cao mà model vẫn giữ độ tin cậy.
- 5) `frame_0331.jpg` — score 0.9154, t = 132.4 s, rank 5. Đây là một “trường hợp gần trùng” với `frame_0326` và `frame_0330` trong cùng khoảng 2 giây; chọn thêm khung này giúp bù đắp cho sự chồng lặp, đồng thời kiểm tra tính ổn định khi model xử lý cùng một góc nhìn nhưng các xe di chuyển khác nhau.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet: 
- `frame_0182.jpg` — rank 1, score 0.9591, t = 72.8 s, được chọn và là ví dụ rõ nhất trong contact sheet: đường cao tốc ban đêm, nhiều xe sáng rõ, model dễ và chắc chắn phát hiện box.
- `frame_0326.jpg` — rank 4, score 0.9155, t = 130.4 s, thuộc top selected set và thể hiện cảnh đông xe nhưng không quá mơ hồ; trong CSV, `n_boxes=39`, `n_ambiguous=15` vẫn nằm trong vùng “dùng để chọn”.
- `frame_0331.jpg` — rank 5, score 0.9154, t = 132.4 s, nằm ngay sát `frame_0326` nhưng có vị trí xe khác; hai khung này cùng cho thấy model chọn cả các cảnh tương tự để tăng độ phủ sóng của lưu lượng xe mà không quá tập trung vào một điểm. 

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do: 
- `frame_0368.jpg` — score 0.9003, t = 147.2 s, rank 9 nhưng `selected = False`. Đây là một ví dụ tốt cho “điểm cao nhưng không chọn”: khung này rất gần với nhóm `frame_0369`/`frame_0372` trong cùng vùng thời gian, nên dù cao điểm vẫn bị loại để tránh lặp lại đặc điểm tương tự và giữ ngân sách cho các tình huống khác.

Điều phép chọn này chưa chứng minh về chất lượng mô hình: 
- Lô đã chọn mới dựa trên 50 khung đầu của ranking và 12 frame model chọn, chưa có đánh giá trên tập ngẫu nhiên hay trên các trường hợp khó như xe ẩn trong bóng tối, phản quang sáng lóa, hoặc các xe quá chồng lấp. Vì vậy, chọn này chỉ chứng minh rằng mô hình đang hoạt động tốt trên các “cảnh dễ và đại diện” ở đầu video, chưa chứng minh chất lượng tổng thể trên toàn bộ đoạn video hoặc trên các trường hợp khúc xạ/bóng đen khó.
