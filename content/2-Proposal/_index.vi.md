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

#### 2.1. Bối cảnh

Trong một ứng dụng thương mại điện tử, bước đặt hàng không chỉ là ghi một bản ghi vào cơ sở dữ liệu. Sau khi khách bấm đặt hàng, hệ thống còn phải lưu đơn, gửi email xác nhận, thông báo cho quản trị viên và có thể làm thêm nhiều việc khác (trừ kho, tạo vận đơn). Các việc này phụ thuộc vào những dịch vụ khác nhau, có độ trễ và độ tin cậy khác nhau.

Cách làm đơn giản nhất là xử lý toàn bộ trong cùng một yêu cầu (đồng bộ): API nhận đơn, lần lượt gọi từng dịch vụ, rồi mới trả kết quả cho khách. Cách này dễ viết nhưng bộc lộ nhiều hạn chế khi lượng đơn tăng hoặc khi một dịch vụ phụ gặp sự cố.

#### 2.2. Các vấn đề cụ thể

- *Chậm khi tải tăng:* thời gian phản hồi của API đặt hàng phụ thuộc vào tác vụ chậm nhất trong chuỗi xử lý. Khách phải chờ cả những việc không cần chờ (như gửi email).
- *Dễ mất đơn:* một dịch vụ phụ bị lỗi (ví dụ gửi email thất bại) có thể làm cả yêu cầu thất bại, trong khi đơn có thể đã được ghi một phần.
- *Kết quả không nhất quán:* API trả lỗi cho khách nhưng đơn đã nằm trong cơ sở dữ liệu. Khách bấm đặt lại và tạo ra đơn thứ hai.
- *Gửi trùng:* khi hệ thống thử lại sau lỗi, cùng một đơn có thể bị xử lý hoặc gửi email nhiều lần.
- *Khó phát hiện sự cố:* nếu không có giám sát, quản trị viên chỉ biết lỗi khi khách hàng phản ánh.
- *Khó mở rộng từng phần:* tất cả bước chạy chung một chỗ nên không thể điều chỉnh riêng khả năng xử lý của từng bước.

| Vấn đề | Hậu quả với doanh nghiệp | Biểu hiện có thể đo được |
|---|---|---|
| Phản hồi chậm | Khách bỏ giỏ hàng, trải nghiệm kém | Thời gian phản hồi trung bình và p95 tăng theo tải |
| Mất đơn khi có lỗi | Mất doanh thu, mất uy tín | Tỉ lệ đơn bị mất khi giả lập lỗi |
| Xử lý hoặc gửi trùng | Khách nhận nhiều email, dữ liệu sai | Số email gửi trùng cho cùng một đơn |
| Phát hiện sự cố chậm | Sự cố kéo dài mà không ai biết | Thời gian từ lúc lỗi đến lúc có cảnh báo |

#### 2.3. Vì sao chọn hướng event-driven

Kiến trúc event-driven tách bước nhận đơn khỏi các bước xử lý phía sau bằng một hàng đợi trung gian. API chỉ cần đưa đơn vào hàng đợi rồi trả phản hồi ngay. Các bước còn lại chạy nền, có thể thử lại độc lập, và đơn lỗi được giữ lại thay vì biến mất. Đổi lại, kiến trúc này phức tạp hơn và có hai điểm cần xử lý cẩn thận: hàng đợi tiêu chuẩn có thể giao một tin nhắn nhiều hơn một lần, và kết quả xử lý trở thành bất đồng bộ. Vì vậy đề tài kết hợp thêm cơ chế chống xử lý trùng (idempotency) và giám sát.

#### 2.4. Mục tiêu của đề tài

1. Thiết kế kiến trúc xử lý đơn hàng bất đồng bộ trên AWS, triển khai được bằng mã (AWS SAM).
2. Bảo đảm đơn không bị mất khi một bước xử lý lỗi, và không bị xử lý hoặc gửi email trùng.
3. Có giám sát và cảnh báo để quản trị viên biết sự cố kịp thời.
4. Đánh giá bằng thực nghiệm và so sánh với phương án xử lý đồng bộ trong cùng điều kiện.
5. Viết tài liệu hướng dẫn từng bước để người khác làm lại được.

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
- *Phản hồi một phần theo lô (partial batch response):* khi một lô tin nhắn có tin lỗi, chỉ tin lỗi quay lại hàng đợi, các tin còn lại được xóa.

#### 3.3. Giám sát và cảnh báo

CloudWatch theo dõi các chỉ số chính: độ dài hàng đợi, số thông điệp trong DLQ, số lần Lambda lỗi, thời gian xử lý và số lần Step Functions thất bại. Khi vượt ngưỡng (ví dụ DLQ có thông điệp), hệ thống gửi cảnh báo cho quản trị viên qua SNS.

| Cảnh báo | Điều kiện | Ý nghĩa |
|---|---|---|
| DLQ không rỗng | Có ít nhất 1 tin trong DLQ | Có đơn xử lý thất bại hoàn toàn, cần kiểm tra |
| Step Functions thất bại | Có ít nhất 1 lần chạy thất bại trong 1 phút | Quy trình gửi email hoặc thông báo lỗi sau mọi lần thử lại |
| Lambda xử lý đơn lỗi | Có ít nhất 1 lỗi trong 1 phút | Hàm xử lý đơn gặp lỗi, đơn sẽ được thử lại |

Ngoài cảnh báo, một dashboard CloudWatch hiển thị độ sâu hàng đợi, DLQ, thời gian xử lý của Lambda, số lần chạy Step Functions và độ trễ của API.

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

#### 3.6. Luồng xử lý chi tiết

1. Khách gửi yêu cầu đặt hàng tới API Gateway; hàm Lambda kiểm tra dữ liệu, đưa đơn vào SQS và trả mã **202 (Accepted)** ngay.
2. Lambda xử lý nhận tin nhắn từ SQS, ghi khóa chống trùng vào DynamoDB bằng thao tác ghi có điều kiện (mỗi `orderId` chỉ được ghi một lần), lưu đơn và khởi động state machine.
3. Step Functions gọi hàm gửi email qua Amazon SES, thử lại tối đa 3 lần với thời gian chờ tăng dần.
4. Thành công: gửi thông báo "đơn đã xử lý" cho quản trị viên qua SNS. Thất bại sau mọi lần thử: gửi cảnh báo qua SNS và kết thúc ở trạng thái lỗi.
5. Tin nhắn không xử lý được sau số lần nhận cho phép (`maxReceiveCount = 3`) được SQS chuyển sang DLQ; alarm CloudWatch phát hiện và báo cho quản trị viên.

Các quyết định thiết kế chính:

| Quyết định | Lý do |
|---|---|
| Dùng hàng đợi SQS tiêu chuẩn | Thông lượng cao; có thể giao một tin nhiều hơn một lần (at-least-once) nên cần chống trùng ở phía xử lý |
| `maxReceiveCount = 3` | Cho phép thử lại lỗi tạm thời nhưng không để tin lỗi vòng lặp mãi |
| `VisibilityTimeout` 60 giây, lớn hơn thời gian chạy tối đa của Lambda xử lý (30 giây) | Tránh tin nhắn hiện lại khi hàm vẫn đang xử lý |
| Khóa idempotency tự hết hạn sau 60 giây, bản ghi tự xóa sau 24 giờ (TTL) | Worker bị chết giữa chừng không làm đơn kẹt mãi |
| Tách đường đồng bộ làm đối chứng (`/orders-sync`) | Có cơ sở so sánh công bằng trong cùng điều kiện |

#### 3.7. Công nghệ và công cụ sử dụng

| Nhóm | Công cụ |
|---|---|
| Hạ tầng dưới dạng mã | AWS SAM (AWS CloudFormation) |
| Môi trường chạy | AWS Lambda, Node.js 20, kiến trúc arm64 |
| Dịch vụ AWS | API Gateway, SQS, DynamoDB, Step Functions, SES, SNS, CloudWatch |
| Công cụ dòng lệnh | AWS CLI, SAM CLI, PowerShell |
| Công cụ đo | Script `loadtest.js` tạo tải và đo thời gian phản hồi |
| Báo cáo | Hugo, GitHub Pages |

#### 3.8. Dữ liệu và giao diện (API)

| Phương thức, đường dẫn | Mô tả |
|---|---|
| `POST /orders` | Đường bất đồng bộ: đưa đơn vào SQS và trả 202 ngay |
| `POST /orders-sync` | Đường đồng bộ làm đối chứng: ghi đơn và gửi email ngay trong yêu cầu |

Nội dung đơn hàng gồm: `email` (người nhận xác nhận), `items` (danh sách `sku`, `qty`), `total`. Các trường `simulateFailure` và `simulateEmailFailure` chỉ dùng để giả lập lỗi trong thực nghiệm và sẽ được loại bỏ khi dùng thật.

| Bảng DynamoDB | Khóa | Nội dung |
|---|---|---|
| Orders | `orderId` | Thông tin đơn và trạng thái xử lý |
| Idempotency | `orderId` | Khóa chống xử lý trùng, có TTL |

#### 3.9. Bảo mật

- Mỗi Lambda chỉ có quyền cần thiết trên đúng tài nguyên của nó (nguyên tắc đặc quyền tối thiểu); riêng `ses:SendEmail` dùng `Resource: '*'` cho đơn giản trong môi trường thử nghiệm.
- Không đưa Access Key hoặc khóa API lên GitHub; dùng profile AWS CLI và biến môi trường.
- SES ở chế độ sandbox chỉ gửi tới địa chỉ đã xác minh hoặc hộp thư giả lập.
- Sau khi hoàn thành, xóa stack và vô hiệu hóa Access Key.

### 4. Kết quả mong đợi

- Một hệ thống xử lý đơn hàng hoạt động trên AWS theo hướng event-driven, triển khai được bằng AWS SAM.
- Đơn hàng không bị mất khi một bước xử lý lỗi: đơn lỗi được giữ trong DLQ và có thể xử lý lại.
- Cùng một đơn không bị xử lý hoặc gửi email trùng nhờ cơ chế idempotency.
- Quản trị viên nhận được cảnh báo khi hệ thống có sự cố nhờ CloudWatch và SNS.
- Số liệu thực nghiệm so sánh kiến trúc event-driven với phương án xử lý đồng bộ qua các chỉ số ở mục 3.5.
- Tài liệu workshop hướng dẫn từng bước (chuẩn bị, SQS và DLQ, idempotency, Step Functions, giám sát, thực nghiệm, dọn dẹp tài nguyên).

### 5. Thời lượng và chi phí

Thời gian thực hiện workshop khoảng 2 đến 3 giờ. Phần lớn dịch vụ nằm trong free tier hoặc có chi phí rất thấp ở quy mô thử nghiệm. Cần thực hiện bước dọn dẹp tài nguyên ở cuối để tránh phát sinh phí, và đặt cảnh báo chi phí (AWS Budgets) từ đầu.

### 6. Kế hoạch triển khai

| Giai đoạn | Công việc | Kết quả |
|---|---|---|
| Tuần 9 | Chuẩn bị tài khoản, quyền IAM, công cụ; thiết kế kiến trúc | Môi trường sẵn sàng, bản thiết kế |
| Tuần 10 | Triển khai SQS, DLQ và cơ chế chống xử lý trùng bằng DynamoDB | Đơn hợp lệ được ghi một lần, đơn lỗi vào DLQ |
| Tuần 11 | Triển khai Step Functions, SES, SNS và giám sát bằng CloudWatch | Quy trình gửi email có thử lại, cảnh báo hoạt động |
| Tuần 12 | Thực nghiệm lỗi và tải, dọn dẹp tài nguyên, hoàn thiện báo cáo | Số liệu so sánh, báo cáo hoàn chỉnh |

### 7. Ước tính chi phí

| Dịch vụ | Nhận định ở quy mô thử nghiệm |
|---|---|
| Lambda, SQS, DynamoDB (theo yêu cầu), Step Functions, SNS, CloudWatch | Thường nằm trong free tier hoặc chi phí rất thấp với vài trăm đơn |
| Amazon SES | Ở chế độ sandbox, chỉ gửi tới địa chỉ đã xác minh hoặc hộp thư giả lập của SES |
| API Gateway | Chi phí theo số yêu cầu, rất thấp ở mức tải thử nghiệm |

Con số cụ thể cần kiểm tra bằng AWS Pricing Calculator trước khi chạy và đối chiếu lại bằng Cost Explorer sau 1 đến 2 ngày. Một cảnh báo chi phí (ví dụ ngưỡng 5 USD) được đặt từ đầu, và toàn bộ tài nguyên được xóa ở bước dọn dẹp.

### 8. Rủi ro và biện pháp giảm thiểu

| Rủi ro | Biện pháp |
|---|---|
| SES sandbox giới hạn 1 email mỗi giây, làm nghẽn khi thử tải lớn | Dùng hộp thư giả lập của SES, thử ở mức tải vừa phải, tăng số lần thử lại, hoặc xin chuyển ra khỏi sandbox |
| Quên xóa tài nguyên gây phát sinh chi phí | Có mục dọn dẹp riêng, đặt AWS Budgets, kiểm tra Cost Explorer |
| Lộ Access Key hoặc khóa API của hệ thống thử nghiệm | Không đưa lên GitHub, dùng `.gitignore`, xóa hoặc vô hiệu hóa key sau khi hoàn thành |
| Lỗi quyền IAM khi triển khai | Tạo sẵn policy cho workshop và kiểm tra bằng `aws sts get-caller-identity` |
| Số liệu đo bị nhiễu do khởi động lạnh của Lambda | Chạy khởi động nóng trước, mỗi mức tải chạy ít nhất 3 lần |

### 9. Phạm vi và giới hạn

- **Trong phạm vi:** luồng nhận đơn, xử lý bất đồng bộ, chống xử lý trùng, gửi email, cảnh báo, giám sát và thực nghiệm so sánh với phương án đồng bộ.
- **Ngoài phạm vi:** thanh toán, quản lý kho, giao diện người dùng, đăng nhập và phân quyền người dùng cuối.
- **Giới hạn:** thực nghiệm ở quy mô nhỏ nên kết quả mang tính tham khảo; đơn đã vào DLQ cần người đưa lại hàng đợi bằng tay (`start-message-move-task`), chưa có cơ chế tự xử lý lại, đây là hướng phát triển tiếp.

### 10. Tiêu chí thành công

- Đơn hợp lệ được xử lý xong, có email xác nhận và bản ghi đơn trong DynamoDB.
- Không có đơn nào bị mất khi giả lập lỗi: đơn lỗi nằm trong DLQ hoặc quy trình báo lỗi rõ ràng.
- Cùng một mã đơn gửi nhiều lần chỉ tạo một bản ghi và một email.
- Quản trị viên nhận cảnh báo khi có sự cố.
- Có số liệu so sánh đầy đủ giữa hai phương án theo các chỉ số ở mục 3.5.

### 11. Tài liệu tham khảo

1. [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
2. [Handling errors for an SQS event source in Lambda](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html)
3. [AWS Step Functions: error handling](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html)
4. [Amazon SQS: available CloudWatch metrics](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-available-cloudwatch-metrics.html)
5. [CloudWatch Logs Insights query syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)