---
author: 
- AGS
- Viettel
title: PVN OSDU Ingestion tools - Hướng dẫn cài đặt 
---
# Tổng quan
TO BE FILLED

# Điều kiện tiên quyết (Prerequisites)

Hạ tầng PVN On-premise OSDU Platform đã được cài đặt và triển khai thành công (Xem tài liệu _osdu-platform_) để có thể thực hiện triển khai PVN OSDU Ingestion Tools. 

PVN OSDU Ingestion Tools cũng sử dụng cùng các hạ tầng/dịch vụ/phần mềm của PVN On-premise OSDU Platform sau:

- Hạ tầng K8s nơi PVN On-premise OSDU Platform được triển khai lên (goi tắt là "hạ tầng K8S")
- Dịch vụ lưu trữ cung cấp nền tảng lưu trữ cho các PV, PVC của hạ tầng K8S
- Dịch vụ service mesh (istio) của hạ tầng K8S

# Build PVN OSDU Ingestion Tools from source code
PVN OSDU Ingestion Tools cần được build thành Docker image để chạy trong môi trường K8S. Các bước thực hiện build như sau:
**BEGIN: Hoàng sửa**
- Bước 1:
- Bước 2:
- Bước 3:
**END: Hoàng sửa** 

# Triển khai PVN OSDU Ingestion Tools vào hạ tầng K8S
Sau khi PVN OSDU Ingestion Tools image được build thành công. Image này cần được triển khai vào hạ tầng K8S. Các bước thực hiện được trình bày trong các tiểu mục sau

## Chuẩn bị thông tin triển khai 
**BEGIN: Hoàng sửa**
- Tuỳ biến namespace
- Biến môi trường
- Config map
- volume mount
- ...
**END: Hoàng sửa** 

## Thực hiện triển khai
**BEGIN: Hoàng sửa**
Lệnh chạy triển khai
Kết quả của việc triển khai thành công
**END: Hoàng sửa** 