---
title: "Giới thiệu và kiến trúc"
date: "2026-10-03"
weight: 1
chapter: false
pre: " <b> 5.1 </b> "
---
## Tổng quan

Trong một ứng dụng thương mại điện tử, bước đặt hàng kéo theo nhiều tác vụ phía sau: ghi nhận đơn, gửi email xác nhận, thông báo cho quản trị viên. Nếu xử lý tất cả trong cùng một yêu cầu (đồng bộ), hệ thống dễ chậm khi lượng đơn tăng, và một dịch vụ phụ bị lỗi có thể làm mất đơn hoặc gửi email trùng.

Workshop này xây dựng luồng xử lý đơn hàng theo hướng **event-driven, chịu lỗi và quan sát được** trên AWS, sau đó đo đạc để so sánh với phương án xử lý đồng bộ.

## Bạn sẽ làm gì

- Tách bước nhận đơn khỏi bước xử lý bằng **Amazon SQS** và hàng đợi lỗi (**DLQ**)
- Chống xử lý trùng bằng **Amazon DynamoDB** (idempotency)
- Điều phối quy trình gửi email và thông báo bằng **AWS Step Functions**, **Amazon SES**, **Amazon SNS**
- Theo dõi hệ thống và nhận cảnh báo với **Amazon CloudWatch**
- Giả lập lỗi và tải để đánh giá hiệu quả của kiến trúc

## Kiến trúc

**Sơ đồ kiến trúc**
![Kiến trúc](/images/5-Workshop/5.1-Introduction/001.png)

**Giải thích sơ đồ**

1. **Nhận đơn.** Khách hàng gửi yêu cầu đặt hàng tới Amazon API Gateway. Hàm Lambda `receive-order` kiểm tra đơn, đưa vào hàng đợi SQS rồi trả mã 202 ngay, nên khách không phải chờ các tác vụ phía sau.
2. **Xử lý nền.** SQS giữ đơn làm bộ đệm khi lượng đơn tăng. Hàm Lambda `process-order` lấy đơn ra, ghi vào DynamoDB bằng thao tác ghi có điều kiện (mã đơn đã tồn tại thì bỏ qua) để một đơn chỉ được xử lý một lần, rồi khởi động Step Functions.
3. **Điều phối và thông báo.** Step Functions gọi hàm Lambda `send-email` để gửi email xác nhận qua Amazon SES, tự thử lại tối đa 3 lần nếu lỗi. Khi kết thúc, Amazon SNS gửi thông báo cho quản trị viên.
4. **Xử lý lỗi.** Đơn bị nhận quá 3 lần mà vẫn thất bại được SQS chuyển sang hàng đợi lỗi (DLQ), không bị mất và có thể đưa lại hàng đợi chính sau khi sửa nguyên nhân.
5. **Giám sát.** Amazon CloudWatch thu log và số liệu. Khi DLQ có tin, Step Functions thất bại hoặc Lambda lỗi, alarm kích hoạt SNS gửi cảnh báo cho quản trị viên.
6. **Đường đối chứng.** Đường `POST /orders-sync` xử lý đồng bộ: một hàm Lambda ghi DynamoDB và gửi email ngay trong yêu cầu. Đường này chỉ dùng để so sánh trong thực nghiệm.

| Thành phần | Nhiệm vụ | Giải quyết vấn đề nào |
|---|---|---|
| SQS | Đệm và giữ đơn | Phản hồi chậm, mất đơn |
| DynamoDB | Ghi có điều kiện theo mã đơn | Xử lý và gửi trùng |
| Step Functions | Điều phối, thử lại từng bước | Lỗi tạm thời ở bước gửi email |
| DLQ | Giữ đơn lỗi | Mất đơn khi lỗi kéo dài |
| CloudWatch và SNS | Giám sát và cảnh báo | Phát hiện sự cố chậm |

## Chỉ số đánh giá

- Thời gian phản hồi của API đặt hàng (trung bình, p95)
- Tỉ lệ đơn bị mất khi có lỗi
- Số email gửi trùng
- Thời gian từ lúc xảy ra lỗi đến lúc quản trị viên nhận cảnh báo

## Thời lượng và chi phí

Khoảng 2–3 giờ. Phần lớn dịch vụ nằm trong free tier hoặc có chi phí rất thấp ở
quy mô thử nghiệm. Hãy thực hiện bước dọn dẹp ở cuối để tránh phát sinh phí.