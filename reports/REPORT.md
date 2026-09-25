# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: CHU THÁI HÒA 

Công cụ gán nhãn đã dùng: CVAT

Sao chép file này thành `reports/REPORT.md` rồi điền vào các chỗ ĐIỀN. Mọi con số phải truy được
từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json` hoặc
`outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Camera đứng cố định nên một chiếc xe có thể xuất hiện trong nhiều frame liên tiếp trong vài giây. Vì vậy pool và test set được tách theo trục thời gian, có vùng đệm ở giữa, để các frame gần như cùng một cảnh hoặc cùng một xe không xuất hiện ở cả hai phía. Cách này giảm rò rỉ theo thời gian và đo gần hơn khả năng tổng quát sang đoạn video khác.

Nếu chia ngẫu nhiên, các frame rất giống nhau có thể vừa được dùng để chọn/sửa nhãn vừa nằm trong test. Khi đó số đo trên test sẽ bị lệch tăng, thường đẹp hơn thực tế, vì model gặp lại cùng bối cảnh, vị trí xe và điều kiện ánh sáng thay vì dự đoán trên dữ liệu thời gian độc lập.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Ở `compare_round0.jpg`, model không khớp nhiều box xe ở xa hoặc bị tối/chồng lấn: các xe nhỏ chỉ còn đèn thường bị bỏ sót, còn một số xe vừa/lớn có box lệch hoặc model không bao phủ đủ thân xe. Ví dụ ở `frame_0050`, cold start có TP 11, FP 2, FN 7; các box nhỏ phía xa và box ở vùng tối là phần khó khớp.

Recall theo kích thước cho thấy model tìm xe nhỏ kém rõ rệt: R small chỉ 0.182, trong khi R medium là 0.547 và R large là 0.561. Vì vậy xe càng xa/càng nhỏ càng dễ bị bỏ sót. Tuy nhiên, trước khi kết luận một box là lỗi model, cần người rà lại nhãn tham chiếu, nhất là các xe bị che, bị cắt ở mép ảnh hoặc chỉ còn đèn; nhãn test hiện chưa được người kiểm từng box.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

Công thức `score = W_U·U + W_A·A + W_D·D` cộng ba tín hiệu: `U` là độ bất định của model, `A` là mức mơ hồ/nhiều box khó quyết định cần rà, còn `D` là độ khác biệt so với các frame đã chọn. Theo hướng dẫn, trọng số tương ứng là 0.5, 0.3 và 0.2. `MIN_GAP_S` yêu cầu hai frame được chọn cách nhau ít nhất 2 giây để tránh dùng ngân sách cho các ảnh gần trùng từ camera cố định.

Ba frame trong `SELECTION.md` minh họa các ưu tiên: `frame_0182.jpg` hạng 1, score 0.9591, U 0.9182, A 1.0000, D 1.0000; `frame_0369.jpg` hạng 2, score 0.9324, U 0.9315, A 0.8889, D 1.0000; `frame_0380.jpg` hạng 3, score 0.9170, U 0.9340, A 0.8333, D 1.0000. Đây là các ảnh model không chắc, có nhiều box khó quyết định và đủ khác biệt để việc rà nhãn có giá trị.

Một frame khác là `frame_0372.jpg`, hạng 6, score 0.9101, U 0.9202, A 0.8333, D 1.0000. Điểm cao nhưng không chọn vì gần thời gian với `frame_0369.jpg` và cảnh gần như trùng, nên chi phí sửa cả hai có thể không đem lại thêm nhiều thông tin. Điểm bất định chỉ chứng minh model đang phân vân, không chứng minh ảnh đó chắc chắn sẽ cải thiện model: sự phân vân có thể do ảnh quá tối, xe chồng lấn, nhãn mơ hồ hoặc lỗi nhãn tham chiếu.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

Ở vòng 1, model đề xuất 169 box trên 12 ảnh. Sau khi rà bằng CVAT còn 331 box: giữ nguyên 146, chỉnh sửa 11, xóa 12 box giả và thêm 174 box bị thiếu; accept rate là 86%. `frame_0099.jpg` và `frame_0107.jpg` mỗi ảnh được thêm 12 box; `frame_0369.jpg` được thêm 22 box. Đây là bằng chứng pre-label bỏ sót nhiều xe, không phải bằng chứng rằng mọi nhãn tham chiếu đều tuyệt đối đúng.

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vòng 1 | 12 | 331 | 0.706 | -0.065 | 1.000 | 0.226 | 0.368 | 0.000 | 0.206 | 0.732 |

AP50 vòng 1 là 0.7064, giảm 0.0650 so với cold start 0.7714 và giảm 0.0650 so với vòng trước. Precision tăng từ 0.9249 lên 1.0000 và FP giảm từ 16 xuống 0, nhưng recall giảm từ 0.4888 xuống 0.2258, F1 từ 0.6396 xuống 0.3684. Theo kích thước, xe lớn tốt lên từ R 0.5610 lên 0.7317, nhưng xe vừa xấu đi từ 0.5473 xuống 0.2061 và xe nhỏ xấu đi từ 0.1818 xuống 0.0000.

Một ca xấu đi trong `compare_round0.jpg` và `compare_round1.jpg` là `frame_0050`: cold start có TP 11, FP 2, FN 7, còn vòng 1 có TP 7, FP 0, FN 11. Model sau fine-tune bớt box sai nhưng bỏ sót thêm xe, nên precision đẹp hơn không đồng nghĩa kết quả tốt hơn tổng thể. Trước khi quy lỗi cho fine-tune, cần kiểm tra lại box train, đặc biệt các box mới thêm, cách đóng gói nhãn và sự khác nhau giữa nhãn tham chiếu chưa rà với ảnh thật.

Các bằng chứng cần phân biệt như sau: `BLIND_SCAN.md` là quan sát độc lập trước pre-label; ở `frame_0099.jpg` người quan sát đếm 24 xe và lưu ý hai xe chồng lấn cùng một xe bị lóa đèn. `REVIEW_LOG.csv` ghi thao tác người sửa, như thêm box ở `frame_0099.jpg`, thêm box ở `frame_0107.jpg` và thêm/sửa box xe buýt ở `frame_0392.jpg`. Các TP/FP/FN trong ảnh compare là kết quả model sau train, không phải bằng chứng rằng người đã sửa đúng mọi nhãn. Một ca khó theo guideline là hai xe chồng lấn khoảng 80%: phải vẽ hai box riêng nếu nhận ra hai thân xe, không gộp thành một box; xe bị cắt mép chỉ khoanh phần thân còn nhìn thấy và vệt sáng trên đường không phải xe.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

Vòng 1 không tốt hơn cold start theo AP50: 0.7064 so với 0.7714, giảm 0.0650. Vì recall và F1 cùng giảm mạnh, mình dừng ở vòng này để kiểm tra dữ liệu và nhãn trước khi train tiếp. Có thể tiếp tục một vòng sau chỉ khi đã xác minh các box mới thêm và chạy lại đúng gói dữ liệu.

Hai nhóm nên ưu tiên ở vòng sau là (1) xe xa chỉ còn hai chấm đèn, vì xe nhỏ hiện có R 0.000 và cần rà kỹ nhưng chi phí cao do khó phân biệt xe với đèn; (2) xe bị cắt mép hoặc chồng lấn, như các ca trong `frame_0099.jpg` và `frame_0182.jpg`, vì cần tách box riêng và dễ sai guideline. Nên tránh chọn hai frame cách nhau dưới `MIN_GAP_S` vì cảnh gần trùng làm tăng chi phí rà mà ít thêm thông tin.

Kết luận vẫn có giới hạn lớn: test chỉ có 20 ảnh, 14 box quá nhỏ bị bỏ qua, và nhãn tham chiếu do model tạo chưa được người rà thủ công. Do đó AP50 và recall chỉ là tín hiệu trên một test nhỏ, không phải chân lý tuyệt đối; recall bằng 0 của xe nhỏ có thể chịu ảnh hưởng của luật bỏ qua kích thước và chất lượng nhãn tham chiếu. Nếu AP50 giảm, trước khi train thêm cần kiểm tra ảnh train, số lượng và vị trí box sau khi export từ CVAT, các box mới thêm/xóa trong `round1_diff.md`, class id và việc đóng gói đúng 12 ảnh; sau đó đối chiếu lại các ca trong compare với nhãn tham chiếu và log rà nhãn.
