# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:

Trong 50 dòng đứng đầu outputs/selection_round1.csv, nếu chỉ có ngân sách rà 5 ảnh, tôi ưu tiên:
| Thứ tự rà | Frame            |      Score |  Thời điểm | Lý do                                                                                                                                  |
| --------- | ---------------- | ---------: | ---------: | -------------------------------------------------------------------------------------------------------------------------------------- |
| 1         | `frame_0182.jpg` | **0.9591** |  **72.8s** | Score cao nhất trong top 50; `U=0.9182`, `A=1.0000`, có **28 box và 18 box ambiguous**, nên có nhiều trường hợp cần kiểm tra thủ công. |
| 2         | `frame_0369.jpg` | **0.9324** | **147.6s** | Score rất cao, `U=0.9315`, 43 box và 16 ambiguous. Đây là frame có nhiều đối tượng và độ bất định cao.                                 |
| 3         | `frame_0331.jpg` | **0.9154** | **132.4s** | `A=1.0000`, **47 box và 18 ambiguous**. Ưu tiên vì số lượng trường hợp cần rà lớn.                                                     |
| 4         | `frame_0312.jpg` | **0.9100** | **124.8s** | `A=1.0000`, 37 box và **18 ambiguous**; bổ sung một thời điểm khác trong video để tránh chỉ tập trung vào một đoạn.                    |
| 5         | `frame_0099.jpg` | **0.9063** |  **39.6s** | Score cao (`U=0.9460`) và nằm ở đoạn thời gian khác, giúp mẫu 5 frame phủ nhiều giai đoạn của video hơn.                               |

Một quyết định có xét ảnh gần trùng: frame_0369.jpg được chọn thay vì frame_0368.jpg. Hai frame chỉ cách nhau 0.4 giây và đều nằm trong top 10, nhưng frame_0369 có score 0.9324 so với 0.9003 của frame_0368. Vì vậy không nên dùng cả hai ngân sách cho hai frame rất gần nhau; chọn frame_0369 để giữ độ đa dạng thời gian.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet: 

1. frame_0182.jpg — rank 1, score 0.9591, 28 boxes, 18 ambiguous, selected=True.
2. frame_0369.jpg — rank 2, score 0.9324, 43 boxes, 16 ambiguous, selected=True.
3. frame_0331.jpg — rank 5, score 0.9154, 47 boxes, 18 ambiguous, selected=True.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do: 

Frame có điểm cao nhưng không được chọn: frame_0368.jpg.

- Rank: 9
- Score: 0.9003
- Thời điểm: 147.2s
- U=0.9339
- 33 boxes
- 14 ambiguous
- selected=False

Mặc dù score cao, chỉ cách frame_0369.jpg 0.4 giây. Vì frame_0369.jpg đã được chọn và có score cao hơn (0.9324), việc chọn thêm frame_0368.jpg sẽ làm ngân sách tập trung vào hai frame rất gần nhau thay vì phủ rộng các thời điểm khác.

Một frame điểm thấp vẫn đáng xem: frame_0392.jpg vì nó vẫn được model chọn dù chỉ rank 15, score 0.8874. Điểm không cao bằng top đầu nhưng U=0.9747 là cao nhất trong các frame được chọn, trong khi A=0.6667 và có 12 ambiguous. Điều này cho thấy chỉ nhìn score tổng hợp không nên là tiêu chí duy nhất khi rà.

Điều phép chọn này chưa chứng minh về chất lượng mô hình: 

Phép chọn này chỉ chứng minh rằng các frame được ưu tiên có mức độ bất định/độ khó cao theo tiêu chí scoring đã thiết kế; nó chưa chứng minh model có độ chính xác cao hay thấp.

Cụ thể, score, U, A, D, số box và số ambiguous là tín hiệu để chọn frame cần con người kiểm tra, không phải ground truth. Muốn đánh giá chất lượng mô hình cần có nhãn chuẩn/ground truth và tính các metric phù hợp như precision, recall, IoU/mAP, hoặc so sánh prediction với annotation sau khi rà thủ công.

Ngoài ra, việc chọn các frame có score cao có thể tạo selection bias: mẫu được rà chủ yếu là những frame mà chính scoring system cho là khó, nên kết quả của 5 frame này không đại diện cho toàn bộ 268 frame của Round 1.