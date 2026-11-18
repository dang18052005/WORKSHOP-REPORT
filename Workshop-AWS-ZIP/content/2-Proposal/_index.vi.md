---
title : "Bản đề xuất"
date : 2026-11-11
weight : 2
chapter : false
pre : " <b> 2. </b> "
---

# Thiết kế và triển khai kiến trúc xử lý đơn hàng event-driven chịu lỗi, có giám sát trên AWS, và đánh giá thực nghiệm

### 1. Tóm tắt

Đề tài thiết kế và triển khai một luồng xử lý đơn hàng theo hướng *event-driven* trên AWS. Hệ thống tách bước nhận đơn khỏi các tác vụ phía sau (ghi nhận đơn, gửi email xác nhận, thông báo cho quản trị viên) bằng hàng đợi *Amazon SQS* và hàng đợi lỗi (*DLQ*), chống xử lý trùng bằng *Amazon DynamoDB*, điều phối quy trình gửi email và thông báo bằng *AWS Step Functions*, *Amazon SES* và *Amazon SNS*, đồng thời theo dõi hệ thống bằng *Amazon CloudWatch*.

Sau khi triển khai, đề tài mô phỏng lỗi và tải để đo hiệu quả của kiến trúc, rồi so sánh với phương án xử lý đồng bộ.

### 2. Vấn đề

Trong một ứng dụng thương mại điện tử, bước đặt hàng kéo theo nhiều tác vụ phía sau. Nếu xử lý tất cả trong cùng một yêu cầu (đồng bộ), hệ thống gặp các vấn đề sau:

- *Chậm khi tải tăng:* thời gian phản hồi của API đặt hàng phụ thuộc vào tác vụ chậm nhất trong chuỗi xử lý.
- *Dễ mất đơn:* một dịch vụ phụ bị lỗi (ví dụ gửi email thất bại) có thể làm cả yêu cầu thất bại.
- *Gửi trùng:* khi hệ thống thử lại sau lỗi, cùng một đơn có thể bị xử lý hoặc gửi email nhiều lần.
- *Khó phát hiện sự cố:* nếu không có giám sát, quản trị viên chỉ biết lỗi khi khách hàng phản ánh.

Vì vậy cần một kiến trúc tách rời các bước xử lý, chịu được lỗi tạm thời, không xử lý trùng và có cơ chế cảnh báo kịp thời.

### 3. Giải pháp đề xuất

#### 3.1. Kiến trúc tổng thể

Luồng xử lý chính: *Nhận đơn → Đưa vào hàng đợi → Xử lý → Điều phối thông báo → Giám sát*.

| Thành phần | Dịch vụ AWS | Vai trò |
|---|---|---|
| Nhận đơn | Amazon API Gateway, AWS Lambda | Nhận yêu cầu đặt hàng và đưa đơn vào hàng đợi, trả phản hồi nhanh cho người dùng |
| Hàng đợi | Amazon SQS | Tách bước nhận đơn khỏi bước xử lý, làm đệm khi lượng đơn tăng |
| Hàng đợi lỗi | SQS Dead-Letter Queue (DLQ) | Giữ lại các đơn xử lý thất bại sau số lần thử lại cho phép, không làm mất đơn |
| Chống xử lý trùng | Amazon DynamoDB | Lưu mã đơn đã xử lý (idempotency) để một đơn chỉ được xử lý một lần |
| Điều phối quy trình | AWS Step Functions | Điều phối các bước gửi email xác nhận và thông báo, kèm cơ chế thử lại |
| Gửi email, thông báo | Amazon SES, Amazon SNS | Gửi email xác nhận cho khách hàng và thông báo cho quản trị viên |
| Giám sát | Amazon CloudWatch | Theo dõi số liệu, nhật ký và đặt cảnh báo (alarm) |

Toàn bộ hệ thống được triển khai ở region **Asia Pacific (Singapore) ap-southeast-1** bằng AWS SAM.

#### 3.2. Các cơ chế chịu lỗi

- *Tách rời bằng SQS:* đơn hàng được lưu vào hàng đợi trước khi xử lý, nên lỗi ở bước sau không làm mất đơn.
- *Thử lại và DLQ:* đơn xử lý lỗi được thử lại theo số lần cấu hình; nếu vẫn thất bại, đơn được chuyển sang DLQ để kiểm tra và xử lý lại.
- *Idempotency bằng DynamoDB:* trước khi xử lý, hệ thống kiểm tra mã đơn trong bảng; đơn đã xử lý sẽ không bị xử lý lại.
- *Điều phối bằng Step Functions:* mỗi bước (gửi email, thông báo) có trạng thái và cơ chế thử lại riêng, dễ theo dõi từng lần chạy.

#### 3.3. Giám sát và cảnh báo

CloudWatch theo dõi các chỉ số chính: độ dài hàng đợi, số thông điệp trong DLQ, số lần Lambda lỗi, thời gian xử lý và số lần Step Functions thất bại. Khi vượt ngưỡng (ví dụ DLQ có thông điệp), hệ thống gửi cảnh báo cho quản trị viên qua SNS.

#### 3.4. Kế hoạch thực nghiệm

Sau khi triển khai, đề tài chạy các kịch bản sau và so sánh với phương án xử lý đồng bộ:

| Kịch bản | Mục đích |
|---|---|
| Tăng dần lượng đơn (kiểm thử tải) | Đo thời gian phản hồi của API đặt hàng (trung bình, p95) |
| Giả lập lỗi ở bước xử lý | Đo tỉ lệ đơn bị mất khi có lỗi, kiểm tra đơn lỗi chuyển vào DLQ |
| Gửi lại cùng một đơn nhiều lần | Đếm số email gửi trùng, kiểm tra cơ chế idempotency |
| Giả lập lỗi dịch vụ gửi email | Đo thời gian từ lúc xảy ra lỗi đến lúc quản trị viên nhận cảnh báo |

#### 3.5. Chỉ số đánh giá

- Thời gian phản hồi của API đặt hàng (trung bình, p95)
- Tỉ lệ đơn bị mất khi có lỗi
- Số email gửi trùng
- Thời gian từ lúc xảy ra lỗi đến lúc quản trị viên nhận cảnh báo

### 4. Kết quả mong đợi

- Một hệ thống xử lý đơn hàng hoạt động trên AWS theo hướng event-driven, triển khai được bằng AWS SAM.
- Đơn hàng không bị mất khi một bước xử lý lỗi: đơn lỗi được giữ trong DLQ và có thể xử lý lại.
- Cùng một đơn không bị xử lý hoặc gửi email trùng nhờ cơ chế idempotency.
- Quản trị viên nhận được cảnh báo khi hệ thống có sự cố nhờ CloudWatch và SNS.
- Số liệu thực nghiệm so sánh kiến trúc event-driven với phương án xử lý đồng bộ qua các chỉ số ở mục 3.5.
- Tài liệu workshop hướng dẫn từng bước (chuẩn bị, SQS và DLQ, idempotency, Step Functions, giám sát, thực nghiệm, dọn dẹp tài nguyên).

### 5. Thời lượng và chi phí

Thời gian thực hiện workshop khoảng 2 đến 3 giờ. Phần lớn dịch vụ nằm trong free tier hoặc có chi phí rất thấp ở quy mô thử nghiệm. Cần thực hiện bước dọn dẹp tài nguyên ở cuối để tránh phát sinh phí, và đặt cảnh báo chi phí (AWS Budgets) từ đầu.