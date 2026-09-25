## Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Phạm Văn Nhật Trường

Công cụ gán nhãn đã dùng: CVAT Docker trên máy cá nhân

## 1. Dữ liệu và cách chia tập

Tập pool và tập test được chia theo thời gian vì camera đứng yên, nên các frame liên tiếp rất giống nhau và cùng một xe có thể ở trong nhiều ảnh kế tiếp. Theo `data/DATA.md`, ảnh pool gần nhất với ảnh test vẫn cách nhau khoảng 4.4 giây và giữa hai đoạn test có vùng đệm 4 giây, nên khi chọn ngẫu nhiên, một xe sẽ xuất hiện đồng thời ở cả huấn luyện và kiểm thử. Khi đó mô hình được chấm trên “xe nó đã từng thấy”, khiến số đo AP50, recall và F1 bị phóng đại lên, tức là rò rỉ dữ liệu (data leakage). Vì vậy chia theo trục thời gian giúp giữ “tương tự” ở trong cùng tập, để đánh giá khả năng tổng quát hóa thực tế.

Nếu chia ngẫu nhiên, mô hình dễ đạt điểm cao vì các ảnh gần nhau có cùng xe, cùng góc và cùng ánh sáng, nhưng đó không phải là đánh giá đúng về khả năng detect trên cảnh mới. Về mặt thực tế, đây là lí do bức tường ngăn giữa pool và test được đặt ở khoảng mắt trung tính, không phải chỉ để cho “đỡ khó” mà để tránh báo cáo sai về mô hình.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 từ `day8_round0_out/reports/rounds_table.md` là:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Dựa vào `day8_round0_out/outputs/compare_round0.jpg`, mô hình cold start không khớp với nhãn tham chiếu chủ yếu ở các xe tối, các xe ở cuối dải phân cách, các xe bị che nửa thân hoặc chỉ còn thấy đèn pha/đèn hậu, và các xe ở xa ở chân trời. Trong ảnh so sánh, nhiều box tham chiếu màu xanh lam vẫn còn trống vì model không phát ra các box tương ứng; ngược lại, nhiều box màu đỏ hoặc vàng trên hình mô hình cho thấy dự đoán sai hoặc đo không trúng với nhãn reference. Đây là nhóm xe “hơi tối, xa, hoặc che bởi xe khác” chứ không phải toàn bộ là xe sáng rõ ở giữa hình.

Độ phủ recall theo kích thước xe cho thấy rất rõ: recall small = 0.182, medium = 0.547, large = 0.561. Nói cách khác, mô hình tốt hơn với xe lớn/giữa hình và rất yếu với xe nhỏ/xa; điều này hợp với việc camera thấy phần lớn mô hình “đoán được” các xe sáng, rõ ràng, còn xe ở xa hoặc chỉ còn đèn bị bỏ gần hết. Một trường hợp cần kiểm tra lại nhãn tham chiếu trước khi kết luận mô hình sai là các xe ở mép xa hoặc xe nằm sau xe khác nhưng vẫn có phần thân nhận ra. Ví dụ như trong `BLIND_SCAN.md`, frame_0369.jpg được ghi nhận có “xe tối ở cuối dải phân cách, có một xe ở dải phân cách phải bị che khá nhiều bởi 1 xe khác”. Nếu trong test/reference, một box như vậy ở chân trời bị vẽ không nhất quán thì không nên kết luận ngay mô hình sai; phải rà lại nhãn tham chiếu trước khi kết luận đồ thị lỗi là của mô hình.

## 3. Chiến lược chọn mẫu

Công thức chọn frame là `score = W_U·U + W_A·A + W_D·D`, trong đó:
- `U` là mức bất định trung bình của 5 box khó nhất trong ảnh, tức các box mà model không chắc chắn giữa `0` và `1`;
- `A` là số box “mập mờ” nằm trong vùng 0.15 ≤ conf < 0.50, chuẩn hóa theo tối đa trong pool;
- `D` là độ đa dạng thời gian, tức khoảng cách đến frame đã chọn gần nhất, chuẩn hóa theo `DIVERSITY_CAP_S`.

`MIN_GAP_S` trong `tools/al_select.py` là 2.0 giây; nghĩa là nếu hai frame quá gần nhau dưới 2 s, chỉ chọn một, vì camera cố định nên khung gần nhau gần như trùng và việc gán nhãn cho cả hai tốn công mà học thêm ít. Điều này giúp mô hình không bị “nạp nhầm” vào các frame gần trùng lặp lại cùng góc nhìn.

Ba ảnh trong lô và một ảnh khác từ `reports/SELECTION.md` chứng minh cách cân nhắc này:
- `frame_0182.jpg` (rank 1, score 0.9591, t = 72.8 s): đây là ảnh rõ nhất, nhiều xe sáng và box đã rõ; chọn đầu tiên vì nó đa dạng và “dễ kiểm chứng” về độ đúng.
- `frame_0326.jpg` (rank 4, score 0.9155, t = 130.4 s): cảnh có 39 box và 15 box mơ hồ; đây là một trường hợp đông xe nhưng còn rõ, rất hữu ích để mô hình học các tồn tại ở mật độ trung bình-cao.
- `frame_0331.jpg` (rank 5, score 0.9154, t = 132.4 s): gần với `frame_0326` và `frame_0330` trong cùng khoảng 2 giây, nên dù score cao vẫn được chọn vì nó bổ sung biến thể góc nhìn/xe, không phải chỉ là gần trùng.
- `frame_0368.jpg` (score 0.9003, t = 147.2 s, rank 9, `selected = False`): điểm cao nhưng không chọn, vì nó gần với chuỗi `frame_0369`/`frame_0372`; nếu lấy cả ba, mô hình học được rất ít hơn so với việc dùng một cảnh đậm độ và một cảnh khác có phương án xe ở vị trí mới.

Vì vậy, “điểm bất định” không tự chứng minh ảnh đó sẽ cải thiện mô hình. Nó chỉ cho biết model lo lắng ở những box đó. Chỉ khi box mơ hồ đó là xe thật, được sửa nhãn đúng, và không quá lặp lại một cảnh gần trùng, nó mới có khả năng giúp mô hình tốt hơn. Nếu một ảnh có score cao nhưng quá gần một ảnh đã chọn, hoặc là cảnh lặp lại không có thêm thông tin, thì không phải là ảnh “hữu ích” dù score cao.

## 4. Các vòng học chủ động (active learning)

Bảng các vòng từ `day8_round1_out/reports/rounds_table.md` là:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 302 | 0.546 | -0.225 | 1.000 | 0.159 | 0.274 | 0.000 | 0.128 | 0.634 |

Từ `outputs/round1_diff.md`, mức sửa nhãn của vòng 1 là:
- 12 ảnh được chọn và sửa;
- model đề xuất 169 box, sau khi sửa còn 302 box;
- giữ nguyên 156 box, sửa 5 box, xoá 8 box FP, thêm 141 box FN;
- accept rate = 92%.

Như vậy, so với cold start, AP50 giảm từ 0.771 xuống 0.546, tức giảm 0.225 (-29%). So với cùng tập test, mô hình sau fine-tune trở nên “chặt hơn” về precision (1.000), nhưng recall tụt mạnh (0.159 so với 0.489), nên nó ít phát hiện xe hơn. Cụ thể: `R small` từ 0.182 xuống 0.000, `R medium` từ 0.547 xuống 0.128, trong khi `R large` tăng từ 0.561 lên 0.634. Kết luận: mô hình học thêm nhạy hơn với xe lớn/đèn sáng rõ, nhưng lại đánh mất khả năng phát hiện xe nhỏ và xe tối ở xa.

Dựa vào `compare_round*.jpg`, một ca đổi rõ sau fine-tune là các xe lớn ở làn ngoài cùng và vùng sáng rõ đã được model phát hiện nhiều hơn, nhưng đồng thời các xe tối ở xa hoặc bị che nửa thân bị bỏ hẳn. Điều này là hợp lý với các số liệu: precision tăng nhưng recall giảm; model “đặt cược” vào những xe rõ và bỏ qua nhóm khó. Đây là một kết quả cần kiểm tra lại xem có phải do nhãn sai trong reference hay do mô hình learning quá mạnh.

Phân biệt các nguồn quan sát:
- `BLIND_SCAN.md` là quan sát độc lập trước khi xem pre-label; nó chỉ ra một số vị trí khó, chẳng hạn frame_0369.jpg có 31 xe nhìn thấy bằng mắt, và nhấn mạnh các vị trí “xe tối ở cuối dải phân cách” và “một xe ở dải phân cách bị che bởi xe khác”.
- `REVIEW_LOG.csv` là nhật ký sửa nhãn sau khi so với pre-label, ghi các sửa như `update`, `add`, `deleted` theo quy tắc guideline. Ví dụ frame_0270.jpg thêm bbox cho xe con ở góc xa, frame_0331.jpg dịch chuyển/thu phóng bbox để bao trọn xe, frame_0369.jpg xóa bbox thừa vì xe nhòe/lóa.
- `outputs/round1_diff.md` là số liệu thực tế sau khi sửa, xuất hiện `added` 141 và `deleted` 8, và chỉ sau đó mới có vòng train 1. Đây là dữ liệu nhãn sau khi đã được người rà, không phải “mô hình tạo rồi coi là đúng”.

Một ca khó theo `GUIDELINE_LABEL.md` là xe ở rất xa, chỉ còn hai chấm đèn, hoặc xe bị che một phần bởi xe khác. Theo guideline, nếu xe bị che một phần thì chỉ vẽ box cho phần nhìn thấy; nếu xe quá xa khiến box cao dưới 16 px thì box đó có thể bị bỏ qua trong chấm điểm; nếu xe bị nhoè do chuyển động thì vẫn vẽ box bao trọn vùng nhoè của thân xe. Chính sự nhất quán trong cách xử lý này mới tạo ra nhãn học tốt, tránh model học “vẽ box ôm cả vệt đèn ở ảnh này nhưng không ở ảnh khác”.

## 5. Kết luận và giới hạn

Kết quả vòng 1 so với cold start là không tốt hơn trên toàn tập test: AP50 giảm từ 0.771 xuống 0.546, dù precision tăng lên 1.0. Điều này cho thấy thêm nhãn chưa chắc đã cải thiện khả năng tổng quát hóa; model đang học cách tránh sai dương nhưng lại bỏ sót quá nhiều xe khó. Vì vậy, nếu chỉ xem AP50 thì không nên train tiếp ngay trong điều kiện hiện tại; cần dừng hoặc làm thêm một vòng “điều chỉnh mẫu và nhãn” để sửa chất lượng nhãn trước khi học tiếp.

Hai trường hợp còn yếu / bất định cho vòng sau:
1. Xe tối ở dải phân cách / xe bị che nửa thân: chi phí rà nhãn khoảng 2–5 box/ảnh, nhưng nguy cơ nhầm nhãn cao vì chỉ có phần thân nhìn thấy và dễ bị gộp với xe phía trước. Đây là trường hợp rất gần với các cảnh mô tả trong `BLIND_SCAN.md` và `REVIEW_LOG.csv`.
2. Xe rất xa, chỉ còn hai chấm đèn, hoặc xe chỉ xuất hiện 1–2 khung liên tiếp: chi phí rà nhãn thấp hơn, nhưng nguy cơ ảnh gần trùng cao vì cùng cái xe kéo dài nhiều frame, và quy định bỏ qua xe nhỏ dưới 16 px khiến bộ học dễ bị thiên về xe lớn hơn. Đây là nhóm khó nhất cho mô hình, và có thể cần chọn frame phân bố thời gian đều thay vì chỉ lấy các frame “đẹp” và giàu xe sáng.

Tập kiểm thử chỉ có 20 ảnh; theo `data/DATA.md`, những box dưới 16 px được bỏ qua và nhãn tham chiếu không phải là nhãn do con người rà lại từng box. Vì vậy, các chênh lệch AP50 nhỏ dưới khoảng 0.01 không đủ để kết luận mô hình tốt lên/xấu đi. Hơn nữa, vì test set là cực kỳ nhỏ, một vài nhãn sai có thể xô lệch AP50 đáng kể. Đó là lý do không nên coi `test/labels` như “ground truth tuyệt đối”; phải xem như “bộ so sánh hiện có” để đọc xu hướng, nhưng không kết luận chắc chắn về chất lượng mô hình trên toàn cảnh.

Nếu AP50 giảm, trước khi train tiếp tôi sẽ kiểm tra các điểm sau:
- xem lại `BLIND_SCAN.md` và `REVIEW_LOG.csv` để chắc rằng không có nhãn reference sai hoặc bbox được sửa sai cách;
- kiểm tra ảnh gần trùng và `MIN_GAP_S` để tránh chọn nhiều frame quá giống nhau;
- xem `compare_round1.jpg` có phải model đang “thích” xe lớn sáng mà bỏ những xe tối/xa không;
- kiểm tra xem drop do nhãn sai, do dữ liệu khó, hay do sample selection thiếu diversity; và nếu cần, sửa nhãn kỹ hơn ở các frame trọng tâm trước khi chạy vòng học tiếp.

Kết luận ngắn gọn: vòng 1 không chứng minh mô hình đã tiến bộ trên tập test, mà chỉ cho thấy mô hình cần tập trung vào xe khó, tối và xa, và nhãn cần được rà lại chặt chẽ hơn trước khi học tiếp.