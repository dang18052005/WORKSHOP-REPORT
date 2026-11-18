---
title: "Theo dõi đơn hàng với Amazon CloudWatch"
date: 2026-11-13
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

## API đã nhận đơn, nhưng đơn hàng đang ở đâu?

Chào mọi người!

Trong một hệ thống xử lý bất đồng bộ, phản hồi tiếp nhận thành công chưa trả lời được câu hỏi: đơn hàng đã hoàn tất hay vẫn đang chờ xử lý?

Đây là vấn đề mình muốn tìm hiểu thêm khi xây dựng luồng xử lý đơn hàng trên AWS.

Bài viết lấy cảm hứng từ [bài giới thiệu các thực hành serverless với Lambda Powertools](https://aws.amazon.com/blogs/opensource/simplifying-serverless-best-practices-with-lambda-powertools/) của **Tom McCarthy**, đăng trên **AWS Open Source Blog ngày 22/06/2020**. Bài đề cập đến log có cấu trúc, metric và tracing để hỗ trợ quan sát ứng dụng.

Ở đây, mình tập trung vào log và metric, sử dụng ví dụ riêng cho đề tài đơn hàng. Dữ liệu và tình huống dưới đây là minh họa, không phải kết quả đo thực tế.

## 1. Tình huống cần điều tra

Giả sử người dùng gửi yêu cầu tạo đơn hàng `ORDER-OBS-01`. API trả về mã đơn hàng và thông báo đã tiếp nhận, nhưng một lúc sau trạng thái vẫn chưa chuyển sang hoàn tất.

Luồng xử lý dự kiến:

```text
Client
  |
  v
API tiếp nhận
  |
  v
Hàng đợi SQS
  |
  v
Lambda xử lý
  |
  v
DynamoDB
```

Mình cần phân biệt các khả năng:

- Yêu cầu đã vào hàng đợi nhưng consumer chưa nhận.
- Consumer đã nhận nhưng đang chờ một bước xử lý.
- Một lần xử lý đã lỗi và công việc đang được thử lại.
- Thông điệp đã chuyển sang dead-letter queue.
- Nghiệp vụ đã hoàn tất nhưng phần truy vấn trạng thái chưa phản ánh đúng.

Một dòng log chung chung như `Processing failed` sẽ khó giúp xác định đơn hàng nào gặp vấn đề và lỗi nằm ở bước nào.

## 2. Thiết kế log để tìm được đúng đơn hàng

Powertools Logger hỗ trợ xuất log JSON, bổ sung thông tin thực thi Lambda và đặt correlation ID. Khi triển khai, cần kiểm soát các trường được ghi để hạn chế đưa dữ liệu nhạy cảm vào log.

Tham khảo: [Powertools for AWS Lambda — Logger](https://docs.aws.amazon.com/powertools/python/latest/core/logger/).

Với workshop, mình đề xuất các trường sau:

| Trường | Mục đích trong thiết kế |
|--------|------------------------|
| orderId | Xác định đơn hàng nghiệp vụ. |
| correlationId | Liên kết các bước thuộc cùng luồng xử lý. |
| messageId | Xác định thông điệp đang được xử lý. |
| function_request_id | Phân biệt từng lần thực thi Lambda. |
| service | Xác định thành phần phát log. |
| event | Mô tả mốc xử lý, chẳng hạn order_received hoặc order_completed. |
| errorType | Phân loại lỗi khi có sự cố. |

Mã đơn hàng có thể giữ nguyên qua nhiều lần thử lại, trong khi mã lần thực thi Lambda thay đổi. Việc lưu cả hai giúp phân biệt một đơn hàng với các lần xử lý của nó.

Ví dụ log khi hoàn tất:

```json
{
  "level": "INFO",
  "service": "order-worker",
  "event": "order_completed",
  "orderId": "ORDER-OBS-01",
  "correlationId": "FLOW-OBS-01",
  "messageId": "MESSAGE-OBS-01",
  "processing_ms": 240,
  "end_to_end_ms": 1840
}
```

Các giá trị thời gian trên chỉ là dữ liệu giả lập để minh họa cấu trúc log.

Trong thiết kế này, API đưa `correlationId` vào thông điệp và consumer đọc lại để ghi log. Bật CloudWatch không tự tạo ra quy ước liên kết nghiệp vụ này.

Khi xử lý batch, mỗi thông điệp cần có đúng ngữ cảnh log của riêng nó, tránh sử dụng nhầm mã đơn hàng từ thông điệp trước.

## 3. Tra cứu bằng CloudWatch Logs Insights

CloudWatch Logs Insights hỗ trợ chọn trường, lọc và sắp xếp log bằng ngôn ngữ truy vấn. Nên chọn các log group liên quan cùng khoảng thời gian cần điều tra để giới hạn dữ liệu quét.

Tham khảo: [CloudWatch Logs Insights query syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html).

Với cấu trúc JSON ở trên, truy vấn minh họa là:

```text
fields @timestamp, @log, service, event, orderId, correlationId, errorType
| filter orderId = "ORDER-OBS-01"
| sort @timestamp asc
| limit 200
```

Cách sử dụng dự kiến:

1. Mở CloudWatch Logs Insights.
2. Chọn log group của API và consumer.
3. Chọn khoảng thời gian gửi đơn hàng.
4. Chọn Logs Insights QL và chạy truy vấn.
5. Đối chiếu các mốc xuất hiện trong kết quả.

Truy vấn này giả định các trường JSON được nhận diện đúng. Nếu có hơn 200 dòng phù hợp, cần điều chỉnh giới hạn hoặc thu hẹp khoảng thời gian.

Không thấy log hoàn tất chưa đủ để kết luận đơn hàng bị mất. Cần kiểm tra cả phạm vi truy vấn, cấu hình ghi log và trạng thái lưu trong cơ sở dữ liệu.

## 4. Dashboard cần theo dõi những gì?

Ngoài việc tìm từng đơn hàng, mình cần nhìn được tình trạng chung của luồng xử lý.

### Nhóm metric của Lambda

| Metric | Ý nghĩa |
|--------|---------|
| Errors | Số lần thực thi kết thúc bằng lỗi hàm. |
| Duration | Thời gian chạy của hàm. |
| Throttles | Số yêu cầu thực thi bị giới hạn. |

`Duration` không bao gồm toàn bộ thời gian đơn hàng chờ trong hàng đợi.

Tham khảo: [AWS Lambda metrics](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-metrics-types.html).

### Nhóm metric của SQS

| Metric | Ý nghĩa |
|--------|---------|
| ApproximateNumberOfMessagesVisible | Số thông điệp đang sẵn sàng để nhận. |
| ApproximateNumberOfMessagesNotVisible | Số thông điệp đã được nhận nhưng chưa bị xóa hoặc hết thời gian ẩn. |
| ApproximateAgeOfOldestMessage | Tuổi thông điệp chưa xử lý lâu nhất được metric phản ánh. |

Các giá trị SQS có tính xấp xỉ. Tuổi thông điệp cũng có những ngoại lệ với thông điệp lỗi lặp lại, nên không đại diện chính xác cho thời gian chờ của mọi đơn hàng.

Tham khảo: [Available CloudWatch metrics for Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-available-cloudwatch-metrics.html).

### Cách đọc kết hợp trong workshop

Nếu số thông điệp chờ và tuổi thông điệp cùng tăng, mình sẽ kiểm tra tốc độ tiếp nhận, năng lực consumer và các bước xử lý chậm.

Nếu `Errors` thấp nhưng đơn hàng vẫn tồn đọng, mình sẽ tiếp tục kiểm tra giới hạn đồng thời, trạng thái message và lỗi ở cấp từng bản ghi.

Đây là hướng điều tra, chưa phải kết luận nguyên nhân chỉ từ một biểu đồ.

## 5. Lambda không báo lỗi có nghĩa mọi đơn đều thành công?

Không nhất thiết.

Khi sử dụng partial batch response, handler có thể trả về danh sách bản ghi thất bại mà không ném exception cho toàn bộ lượt thực thi. Vì vậy, cần phân biệt lỗi của một lần gọi hàm với lỗi của từng đơn hàng.

Tham khảo: [Handling errors for an SQS event source in Lambda](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html).

Trong workshop, mình dự kiến ghi riêng các mốc:

- `order_processing_failed`: một lần xử lý đơn gặp lỗi.
- `order_completed`: nghiệp vụ đã hoàn tất.
- `duplicate_operation_skipped`: bỏ qua tác động đã được thực hiện.

Số dòng log lỗi không đồng nghĩa với số đơn hàng thất bại cuối cùng. Một đơn có thể lỗi ở lần đầu nhưng hoàn tất sau retry.

Khi lập báo cáo, mình sẽ đối soát theo `orderId` và trạng thái cuối cùng.

## 6. Đo đúng thời gian của một đơn hàng

Để so sánh với phương án đồng bộ, mình đề xuất tách ba phép đo:

| Phép đo | Điểm bắt đầu và kết thúc |
|---------|--------------------------|
| Phản hồi API | Client gửi yêu cầu → client nhận phản hồi. |
| Xử lý tại worker | Worker bắt đầu → worker kết thúc một lần xử lý. |
| Hoàn tất toàn luồng | Client gửi yêu cầu đầu tiên → đơn hàng hoàn tất theo tiêu chí đã định. |

Trong ví dụ log, `processing_ms` và `end_to_end_ms` là hai phép đo khác nhau.

Để đo toàn luồng, cần giữ lại mốc bắt đầu qua các lần retry và thống nhất cách xử lý chênh lệch đồng hồ giữa các thành phần.

Nếu client dùng polling để phát hiện trạng thái hoàn tất, số đo còn bao gồm độ trễ do chu kỳ polling. Cách đo này phải được ghi rõ trong báo cáo.

Mình chưa có dữ liệu thực nghiệm để đưa ra kết luận về hiệu năng trong bài viết này.

## 7. Bài lab mình muốn thực hiện

Mình đề xuất thử với dữ liệu giả lập:

1. Gửi một đơn hàng bình thường và tìm đủ các mốc xử lý.
2. Tăng thời gian xử lý tại consumer để quan sát tồn đọng.
3. Tạo lỗi tạm thời rồi kiểm tra log của các lần retry.
4. Tạo lỗi kéo dài và theo dõi thông điệp trong DLQ.
5. Gỡ lỗi, phục hồi thông điệp và đối chiếu trạng thái đơn hàng.

Minh chứng cần lưu gồm truy vấn, log liên quan, ảnh dashboard và bảng trạng thái theo mã đơn hàng.

Mục tiêu của mình là từ một `orderId` có thể giải thích được: đơn đang ở bước nào, đã gặp lỗi gì và cuối cùng được xử lý ra sao.

## Tài liệu tham khảo

1. [AWS Open Source Blog — bài về thực hành serverless với Powertools, Tom McCarthy, 22/06/2020](https://aws.amazon.com/blogs/opensource/simplifying-serverless-best-practices-with-lambda-powertools/).
2. [Powertools for AWS Lambda — Logger](https://docs.aws.amazon.com/powertools/python/latest/core/logger/).
3. [CloudWatch Logs Insights — Query syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html).
4. [AWS Lambda — Types of metrics](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-metrics-types.html).
5. [Amazon SQS — Available CloudWatch metrics](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-available-cloudwatch-metrics.html).
6. [AWS Lambda — Handling errors for an SQS event source](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html).