---
title: "AWS re:Invent 2025 Recap - Hạ tầng & Bảo mật"
date: 2026-01-27T15:59:18+07:00
weight: 3
chapter: false
pre: " <b> 4.1. </b> "
---

**Ngày:** 27 tháng 1, 2026
**Địa điểm:** Văn phòng AWS Việt Nam (Tầng 26 & 36), TP. Hồ Chí Minh
**Vai trò:** Người tham dự (FCJ Cloud Intern - Team NeuraX)

## Mô tả sự kiện

Sự kiện này là buổi tổng kết toàn diện về các cập nhật hạ tầng và bảo mật quan trọng nhất từ AWS re:Invent 2024. Buổi chia sẻ quy tụ các kiến trúc sư đám mây và chuyên gia bảo mật để tìm hiểu về các đổi mới trong thiết kế chip silicon, điện toán serverless và các hoạt động bảo mật được hỗ trợ bởi AI trực tiếp từ các chuyên gia AWS.

## Các hoạt động chính

Sự kiện tập trung vào các chủ đề kỹ thuật then chốt với các chi tiết kỹ thuật được trích xuất từ các phiên thảo luận chuyên sâu của AWS:

**Tính toán thế hệ mới & Silicon**  
Giới thiệu **AWS Graviton5**, sở hữu **192 lõi trên mỗi chip** và **bộ nhớ đệm (cache) lớn gấp 5 lần** so với Graviton4. Các phiên bản **Amazon EC2 M9g** mới chạy trên nền chip này cung cấp hiệu năng giá tốt hơn tới **25%** cho các tải công việc thông thường. Đối với các tác vụ AI chuyên biệt, **Trainium3 UltraServers** đã được ra mắt (hiện có sẵn tại US East/West), được thiết kế để tăng tốc huấn luyện các mô hình với hàng nghìn tỷ tham số trong khi giảm đáng kể chi phí trên mỗi token.

**Sự tiến hóa của Serverless**  
Khám phá **AWS Lambda Managed Instances**, giúp xóa nhòa ranh giới giữa sự đơn giản của serverless và tính linh hoạt của EC2, cho phép các hàm chạy trên phần cứng chuyên dụng với toàn quyền kiểm soát VPC. Một điểm nhấn quan trọng là **AWS Lambda Durable Functions**, giới thiệu cơ chế **durable execution** (thực thi bền bỉ) sử dụng checkpoint và replay. Lập trình viên có thể sử dụng các hàm gốc `context.step()` và `context.wait()` để xây dựng các luồng công việc có thể **tạm dừng thực thi lên đến một năm** mà không phải trả phí cho thời gian chờ, lý tưởng cho các quy trình chờ phê duyệt từ con người hoặc phản hồi AI dài ngày.

**Hạ tầng dữ liệu hiệu năng cao**  
Đi sâu vào **Amazon S3 Vectors** (hiện đã GA), hỗ trợ tới **1 tỷ vector mỗi index** và sử dụng thuật toán **HNSW (Hierarchical Navigable Small World)** cho độ trễ truy vấn dưới 100ms. Khả năng gốc này của S3 giúp giảm tổng chi phí sở hữu lên tới **90%** so với việc vận hành các cơ sở dữ liệu vector chuyên biệt. Ngoài ra, **Amazon S3 Tables** hiện đã hỗ trợ **replication liên vùng** và **Intelligent-Tiering**, tự động di chuyển dữ liệu giữa các tầng nóng (Express One Zone) và lạnh (Standard) dựa trên mức độ truy cập. S3 Tables cũng tích hợp sẵn các tính năng bảo trì như **nén file (compaction)** và **snapshot expiration** để đảm bảo hiệu suất ổn định.

**Hoạt động bảo mật dựa trên AI**  
Bản xem trước về **AWS Security Agent**, cho phép các đội ngũ mở rộng chuyên môn AppSec bằng cách sử dụng LLM. Công cụ này thực hiện các bản **đánh giá thiết kế dựa trên AI** bằng cách phân tích sơ đồ kiến trúc và **phân tích mã nguồn chuyên sâu** để tìm ra các lỗi logic phức tạp vượt ra ngoài danh mục OWASP Top 10. Một tính năng nổi bật là **kiểm thử xâm nhập theo ngữ cảnh (contextual penetration testing)**, trong đó agent tạo ra và thực thi các kịch bản khai thác an toàn trong một môi trường sandbox để xác tin các lỗ hổng trước khi triển khai. **AWS Security Hub** cũng đã chính thức khả dụng rộng rãi (GA) với khả năng phân tích gần như thời gian thực và ưu tiên rủi ro dựa trên các phát hiện trên toàn bộ tài khoản AWS.

**Hạ tầng toàn cầu & Mạng an toàn**  
Giới thiệu **AWS AI Factories**, cho phép các tổ chức triển khai hạ tầng AI được quản lý hoàn chỉnh (bao gồm phần cứng chuyên dụng và các mô hình tích hợp sẵn) ngay tại trung tâm dữ liệu của họ để đáp ứng các yêu cầu về lưu giữ dữ liệu. Về kết nối hybrid, **Amazon Route 53 Global Resolver** (preview) đã được giới thiệu, sử dụng **phân giải dựa trên anycast an toàn** để quản lý DNS thống nhất cho cả tên miền công cộng và nội bộ, giảm bớt nhu cầu cho các kiến trúc DNS hybrid phức tạp và các quy tắc chuyển tiếp thủ công.

## Kết quả

- **Graviton5 & Trainium3**: Thiết yếu để mở rộng quy mô tải công việc AI một cách hiệu quả với 192 lõi và hiệu năng tốt hơn 25%.
- **Lambda Durable Functions**: Đơn giản hóa các luồng công việc dài ngày với quản lý trạng thái nội tại và thời gian tạm dừng lên tới 1 năm.
- **S3 Vectors & Tables**: Cung cấp nền tảng vững chắc và tiết kiệm chi phí cho các ứng dụng RAG khổng lồ với các tính năng bảo trì tự động và tiết kiệm 90% chi phí.
- **AWS Security Agent**: Chủ động bảo vệ ứng dụng thông qua kiểm thử xâm nhập và đánh giá thiết kế tự động trong suốt vòng đời phát triển.
- **Route 53 Global Resolver & AI Factories**: Giải quyết các nhu cầu phức tạp về đám mây lai và chủ quyền dữ liệu với phân giải anycast và hạ tầng AI quản lý on-premises.
- Xác định tiềm năng sử dụng **S3 Tables và GuardDuty** để tăng cường bảo mật dữ liệu và khả năng quan sát vận hành cho dự án NutriTrack.
