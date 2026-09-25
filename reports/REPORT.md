# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: `Lê Duy Phúc Lộc`

Công cụ gán nhãn đã dùng: CVAT

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

`Tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng đệm ở giữa, để hạn chế việc các frame rất gần nhau về thời gian nhưng có nội dung gần như giống nhau xuất hiện đồng thời ở train/pool và test.`

`Nếu chia ngẫu nhiên, các frame liên tiếp hoặc rất gần nhau trong cùng một đoạn video có thể bị chia sang cả train và test. Khi đó mô hình có thể gặp ở train những cảnh gần giống với cảnh trong test, làm cho kết quả test lạc quan hơn thực tế. Nói cách khác, AP50/recall có thể bị cao giả tạo vì test không còn độc lập tốt với dữ liệu dùng để chọn hoặc huấn luyện.`

`Vùng đệm thời gian giúp giảm hiện tượng này bằng cách tách các đoạn có tương quan thời gian cao. Điều này quan trọng hơn random split đối với video vì các frame liên tiếp không phải những mẫu độc lập.`

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

| vòng | model                                   | ảnh train | box train |  AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 |    F1 | R small | R medium | R large |
| ---: | --------------------------------------- | --------: | --------: | ----: | -------------------: | -----: | -----: | ----: | ------: | -------: | ------: |
|    0 | yolov8n cold start (COCO car+bus+truck) |         0 |         0 | 0.771 |                    — |  0.925 |  0.489 | 0.640 |   0.182 |    0.547 |   0.561 |

`Các số liệu chi tiết cho vòng 0 là AP50 0.7714, precision 0.9249, recall 0.4888, F1 0.6396. Tập test có 20 ảnh và 403 box tham chiếu, trong đó 14 box rất nhỏ bị bỏ qua.`

`Từ compare_round0.jpg, cold-start bỏ sót khá nhiều xe ở các vị trí xa/nhỏ, đặc biệt các xe có kích thước rất nhỏ ở phía xa hoặc vùng tối. Điều này phù hợp với recall theo kích thước:`

- Small: R = 0.1818
- Medium: R = 0.5473
- Large: R = 0.5610

`Tức là khả năng thu hồi xe nhỏ thấp hơn rất nhiều so với xe medium và large. Với 66 box small, model chỉ đạt recall khoảng 18.18%; trong khi medium đạt 54.73% và large đạt 56.10%.`

`Về câu hỏi “không khớp ở loại xe nào”: compare_round0.jpg chỉ hiển thị box, không hiển thị nhãn class riêng cho từng box, nên từ ảnh này không đủ bằng chứng để kết luận riêng rằng car, bus hay truck là class nào bị sai nhiều nhất. Kết luận chắc chắn hơn từ số liệu là xe nhỏ/xa là nhóm bị bỏ sót nhiều.`

`Một trường hợp cần người rà lại nhãn tham chiếu trước khi kết luận model sai là frame_0099. Khi quét độc lập trước khi xem pre-label, tôi đếm được 26 xe nhìn thấy, trong đó có hai xe màu đen bị cắt và mờ ở góc phải dưới ảnh, là những vị trí dễ bị AI bỏ sót hoặc vẽ sai.`

`Do đó, nếu reference chỉ có box ở một phần những xe này, không nên lập tức coi prediction là FP/FN; cần kiểm tra guideline và ảnh gốc trước.`

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

`Công thức:`

- score = W_U · U + W_A · A + W_D · D

`có thể hiểu là điểm ưu tiên của một frame được tạo từ ba thành phần:`

- U: uncertainty — mức độ bất định của model.
- A: tín hiệu liên quan đến ambiguity/độ khó của các prediction.
- D: tín hiệu đa dạng, giúp ưu tiên những frame mang thêm thông tin thay vì chỉ chọn các frame giống nhau.
- W_U, W_A, W_D: trọng số quy định mức đóng góp của từng thành phần.

`MIN_GAP_S dùng để tránh chọn nhiều frame quá gần nhau về thời gian. Nếu một frame vừa được chọn và frame tiếp theo chỉ cách nó một khoảng nhỏ hơn MIN_GAP_S, frame đó có thể bị bỏ qua để dành ngân sách cho một thời điểm khác.`

`Ba frame trong reports/SELECTION.md:`

| Frame            |  Score | Thời điểm | Bằng chứng                     |
| ---------------- | -----: | --------: | ------------------------------ |
| `frame_0182.jpg` | 0.9591 |     72.8s | rank 1, 28 boxes, 18 ambiguous |
| `frame_0369.jpg` | 0.9324 |    147.6s | rank 2, 43 boxes, 16 ambiguous |
| `frame_0331.jpg` | 0.9154 |    132.4s | rank 5, 47 boxes, 18 ambiguous |

`Các frame này đều thuộc 12 frame được chọn và có mức score/ambiguity cao.`

`Một frame khác dùng để thể hiện cách cân nhắc ảnh gần trùng là frame_0368.jpg. Frame này có score 0.9003, rank 9, thời điểm 147.2s, nhưng chỉ cách frame_0369.jpg 0.4 giây. Vì frame_0369.jpg có score cao hơn (0.9324), chọn cả hai sẽ tiêu tốn hai vị trí cho hai ảnh rất gần nhau. Do đó ưu tiên frame_0369.jpg để tăng độ phủ theo thời gian.`

`Một trường hợp khác là frame_0392.jpg: score chỉ 0.8874, rank 15, nhưng U=0.9747 và có 12 ambiguous. Vì vậy frame này vẫn đáng xem dù không nằm ở đầu bảng score.`

`Điểm bất định không chứng minh rằng ảnh đó chắc chắn sẽ cải thiện model. Nó chỉ cho biết frame có đặc điểm mà chiến lược selection đánh giá là đáng để con người kiểm tra. Chỉ sau khi con người sửa/ xác nhận nhãn và huấn luyện lại mới có thể kiểm tra xem việc bổ sung dữ liệu có thực sự cải thiện metric trên test hay không. Chính SELECTION.md cũng xác định score/U/A/D là tín hiệu chọn frame chứ không phải ground truth.`

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

`Tập test cố định gồm 20 ảnh, 403 box tham chiếu, bỏ qua 14 box có chiều cao dưới 16 px; IoU threshold = 0.5, còn P/R/F1 được tính tại confidence 0.25.`

| Vòng | Model                    | Ảnh train | Box train |   AP50 | Δ AP50 so cold start |      P |      R |     F1 | R small | R medium | R large |
| ---: | ------------------------ | --------: | --------: | -----: | -------------------: | -----: | -----: | -----: | ------: | -------: | ------: |
|    0 | YOLOv8n cold start       |         0 |         0 | 0.7714 |                    — | 0.9249 | 0.4888 | 0.6396 |  0.1818 |   0.5473 |  0.5610 |
|    1 | YOLOv8n fine-tune vòng 1 |        12 |       333 | 0.4814 |          **-0.2900** | 1.0000 | 0.0620 | 0.1168 |  0.0000 |   0.0473 |  0.2683 |

`Vòng 0 được xác nhận bằng metrics_round0.json; vòng 1 dùng 12 ảnh và 333 box train, train 50 epochs.`

`Mức độ sửa pre-label ở vòng 1`

`Model ban đầu đề xuất 169 box trên 12 ảnh. Sau khi tôi rà sửa, tập train có 333 box. Tổng hợp:`

| Hành động               | Số box |
| ----------------------- | -----: |
| Giữ nguyên (`accepted`) |    120 |
| Chỉnh sửa (`edited`)    |     33 |
| Xóa (`deleted`)         |     16 |
| Thêm mới (`added`)      |    180 |
| Tổng box sau sửa        |    333 |

`AP50 thay đổi so với cold start:`
- AP50 round 0 = 0.7714
- AP50 round 1 = 0.4814

=> Δ AP50 = 0.4814 - 0.7714 = -0.2900

`Cold start có recall small/medium/large lần lượt là 0.1818 / 0.5473 / 0.5610.`

`Sau fine-tune, recall lần lượt còn 0 / 0.0473 / 0.2683.`

`Như vậy cả ba nhóm đều xấu đi, không có nhóm kích thước nào tốt lên theo recall. Medium giảm mạnh nhất, từ 0.5473 xuống 0.0473.`

`Tức AP50 giảm 0.2900 điểm, tương đương giảm khoảng 37.6% so với AP50 ban đầu.`

`Một ca kết quả thay đổi sau fine-tune: frame_0050`

- Reference: 19 box
- Cold start: TP 11, FP 2, FN 7
- Round 1: TP 1, FP 0, FN 17

`Sau fine-tune, số prediction đúng giảm từ 11 xuống 1, trong khi số false negative tăng từ 7 lên 17. Đây là một trường hợp xấu đi rõ rệt.`

Phân biệt quan sát độc lập, lỗi pre-label và kết quả sau train

1. Quan sát độc lập — BLIND_SCAN.md

`Trước khi xem pre-label, tôi đã quét frame_0099 độc lập và ghi nhận:`

- Có 26 xe nhìn thấy bằng mắt.
- Có hai vị trí dễ bị AI bỏ sót hoặc vẽ sai: hai xe màu đen bị ảnh cắt và mờ ở góc phải dưới.

`Đây là quan sát của người, chưa bị ảnh hưởng bởi prediction của model.`

2. Lỗi pre-label đã sửa — REVIEW_LOG.csv và round1_diff.md

`Sau khi xem pre-label, review log ghi nhận:`

- Hai xe tối ở góc dưới bên phải: edited, sửa box sát thân xe.
- Một xe vàng giữa ảnh: edited.
- Một xe sát mép dưới ảnh: added, vì AI bỏ sót.
- Một box sau xe bus bên trái: deleted, vì AI đang box cả 3 xe ở khu vực này.

`Đây là các thay đổi đối với pre-label trước khi tạo dữ liệu train, không phải kết quả của model sau fine-tune.`

`Tổng thể round1 có 33 box chỉnh sửa, 16 box xóa và 180 box thêm mới.`

3. Kết quả model sau train — compare_round1.jpg và metrics_round1.json

`Sau khi train trên 12 ảnh/333 box, model đạt:`

- AP50 = 0.4814
- Precision = 1.0000
- Recall = 0.0620
- F1 = 0.1168

Một ca khó theo guideline

frame_0099 là một ca khó tiêu biểu.

Blind scan ghi nhận hai xe màu đen ở góc phải dưới, vừa bị cắt bởi mép ảnh vừa mờ, là hai vị trí dễ bị AI bỏ sót hoặc vẽ sai.

Trong review:

- hai xe này được chỉnh box để sát thân xe;
- một xe sát mép dưới được thêm mới vì AI bỏ sót.

Vì vậy đây là trường hợp cần áp dụng guideline cẩn thận: xe bị cắt mép ảnh hoặc khó nhìn không nên tự động xem là lỗi của model/reference; cần kiểm tra ảnh gốc và quy tắc đặt box trước khi quyết định giữ, sửa hay thêm box.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

Kết quả so với cold start

`So với cold start, kết quả vòng 1 giảm rõ rệt:`

| Chỉ số        | Cold start | Vòng 1 |    Thay đổi |
| ------------- | ---------: | -----: | ----------: |
| AP50          |     0.7714 | 0.4814 | **-0.2900** |
| Precision     |     0.9249 | 1.0000 |     +0.0751 |
| Recall        |     0.4888 | 0.0620 | **-0.4268** |
| F1            |     0.6396 | 0.1168 | **-0.5228** |
| Recall small  |     0.1818 | 0.0000 | **-0.1818** |
| Recall medium |     0.5473 | 0.0473 | **-0.5000** |
| Recall large  |     0.5610 | 0.2683 | **-0.2927** |

`Precision lại tăng từ 0.9249 lên 1.0000, nhưng đây đi cùng với recall giảm rất mạnh.`

`Vì vậy, với bằng chứng hiện tại, tôi dừng việc train thêm ngay trên cùng pipeline để kiểm tra nguyên nhân trước. Việc tiếp tục thêm dữ liệu mà chưa kiểm tra pipeline có thể làm tăng chi phí gán nhãn nhưng chưa giải quyết nguyên nhân khiến recall giảm.`

Hai ca nên ưu tiên ở vòng sau

- Ca 1 — xe nhỏ/xa:

`Cold start có R small = 0.1818, còn round 1 giảm xuống 0.0000. Đây là nhóm yếu nhất theo metric.`

`Chi phí rà nhãn có thể cao vì xe nhỏ khó nhìn, dễ phải phóng to và kiểm tra box sát kích thước thân xe. Nguy cơ gần-trùng cần kiểm soát bằng MIN_GAP_S, tránh chọn liên tiếp nhiều frame của cùng một đoạn.`

- Ca 2 — frame_0369.jpg:

`Đây là frame có lượng sửa nhãn lớn: model ban đầu đề xuất 14 box, sau khi người rà sửa thành 36 box, tức thêm 22 box.`

`Do đó đây là một trường hợp cho thấy pre-label bỏ sót nhiều đối tượng và đáng xem tiếp. Tuy nhiên chi phí rà cũng tương đối lớn vì phải kiểm tra tới 36 box sau sửa.`

Các giới hạn ảnh hưởng đến kết luận

1. Tập test chỉ có 20 ảnh

`Metric được tính trên 20 ảnh và 403 box tham chiếu, nên kết quả có thể bị ảnh hưởng đáng kể bởi một số frame khó. Vì vậy việc AP50 giảm là bằng chứng rõ ràng rằng vòng 1 không tốt hơn trên tập test này, nhưng chưa nên mở rộng kết luận thành “model chắc chắn kém hơn trên mọi dữ liệu”.`

2. Có luật bỏ qua xe quá nhỏ

`Có 14 box có chiều cao dưới 16 px bị bỏ qua khi đánh giá. Điều này đặc biệt quan trọng vì xe small đang là nhóm yếu nhất. Do đó metric hiện tại không phản ánh đầy đủ khả năng phát hiện những xe cực nhỏ này.`

3. Nhãn tham chiếu chưa được coi là chân lý tuyệt đối

`Một số trường hợp cần kiểm tra lại bằng mắt. Ví dụ BLIND_SCAN.md ghi nhận ở frame_0099 có 26 xe nhìn thấy, trong đó có hai xe màu đen bị cắt và mờ ở góc phải dưới, là các vị trí dễ bị AI bỏ sót hoặc vẽ sai.`

Nếu AP50 giảm, tôi sẽ kiểm tra gì trước khi train thêm?

`Tôi sẽ kiểm tra theo thứ tự:`

- Nhãn train sau khi sửa: kiểm tra số lượng box, tọa độ và box có nằm đúng trên xe hay không.
- Class mapping: kiểm tra mapping giữa car, bus, truck và class ID trong dữ liệu train.
- Format/convert label: kiểm tra quá trình chuyển annotation sang YOLO có làm sai tọa độ hoặc class hay không.
- Checkpoint: xác nhận model được đánh giá đúng là checkpoint sau fine-tune, không phải checkpoint khác.
- Train/test pipeline: kiểm tra preprocessing, image size và cách load dataset có giống giữa các vòng hay không.
- Metric/evaluation: xác nhận confidence threshold và IoU threshold được giữ giống cold start.
- Kiểm tra trực tiếp prediction trên một vài ảnh test để xem hiện tượng “precision 1.0 nhưng recall 0.062” có phải do model gần như không dự đoán box hay không.

`Lý do cần kiểm tra các bước này trước là vòng 1 đã có 333 box sau khi sửa, trong đó có tới 180 box được thêm mới, nhưng kết quả lại giảm mạnh. Vì vậy chưa thể kết luận rằng “thêm dữ liệu active learning không hiệu quả”; trước hết phải xác minh dữ liệu và pipeline fine-tune có đúng hay không.`