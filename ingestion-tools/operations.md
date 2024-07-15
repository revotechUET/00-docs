---
author: 
- AGS
- Viettel
title: PVN OSDU Ingestion tools - Tài liệu vận hành 
---
# Tổng quan
Tài liệu này hướng dẫn cách vận hành/sử dụng PVN OSDU Ingest Tools

# Điều kiện tiên quyết (Prerequisites)
PVN OSDU Ingest Tools cần được triển khai thành công (Xem tài liệu ingestion-tools-main)

# Sử dụng PVN OSDU Ingestion Tools
## Giao diện làm việc
PVN OSDU Ingestion Tools có 2 mode làm việc: (i) Command line mode, và (ii) REST service mode.

Với command-line mode người vận hành cần truy cập vào k8s pod tương ứng với PVN OSDU Ingestion Tools và chạy các lệnh utilities tương ứng

Với REST service mode người vận hành cần gọi các REST Api của PVN OSDU Ingestion Tools bằng các công cụ thích hợp (ví dụ: công cụ lập trình, tiện ích curl hoặc Postman)

## Các nhiệm vụ vận hành

### Nạp dữ liệu reference data
- Working mode: Chạy trục tiếp command trong pod py-osdu
```bash
kubectl exec -it ags-pyosdu-858d7989d5-b4mzl -- bash
```
- các bước thực hiện
```bash
cd reference_ingest
python run_storage.py
```
- Sau khi lệnh python kết thúc. Cần chờ 15-20p để quá trình index được thực hiện. Sau đó thực hiện lệnh tìm kiếm **reference-data** trên OSDU. Kết quả thực hiện thành công
```bash
python -m wi_pyosdu.search --kind "osdu:wks:reference-data--*:*"
```
### Nạp dữ liệu 
### Chuẩn bị dữ liệu: 
- Dữ liệu được đưa vào nfs, đc mount vào thư mục **/data-ingest**. Bao gồm bốn thư mục tương ứng với bốn domain **Wellbore**, **Seismic**, **Reservoirs**, **WellDelivery**
### Nạp dữ liệu Wellbore
#### Command-line mode
- Mở command-line py-osdu
```bash
kubectl exec -it ags-pyosdu-858d7989d5-b4mzl -- bash
```
- Chạy lệnh ingest data wellbore theo thứ tự
```bash
python -m server.cmd --ingest --wellbore --wb-file-type document
python -m server.cmd --ingest --wellbore --wb-file-type las
python -m server.cmd --ingest --wellbore --wb-file-type csv
```
- Hoặc chạy một lệnh để ingest toàn bộ
```bash
python -m server.cmd --ingest --wellbore --wb-file-type all
```
#### REST Service mode
- Kiểm tra endpoint của service py-osdu. Service này đc expose qua service nodePort của k8s, ví dụ: **port 9010**
```bash
# check status service
curl http://master-ip:9010 
{
    "version": "0.0.1",
    "message": "AGS API is running!"
}
```
- Gọi API để trigger quá trình ingest
```bash
# start ingest
curl http://master-ip:9010/ingest/wellbore?api_key=Pvn2024
{
    "running": false,
    "last_run": 0
}
```

### Nạp dữ liệu Seismic
#### Command-line mode
- Mở command-line py-osdu
```bash
kubectl exec -it ags-pyosdu-858d7989d5-b4mzl -- bash
```
- Chạy lệnh ingest data seismic theo thứ tự
```bash
python -m server.cmd --ingest --seismic --ws-file-type document
python -m server.cmd --ingest --seismic --ws-file-type segy
```
- Hoặc chạy một lệnh để ingest toàn bộ
```bash
python -m server.cmd --ingest --seismic --ws-file-type all
```
#### REST Service mode
- Kiểm tra endpoint của service py-osdu. Service này đc expose qua service nodePort của k8s, ví dụ: **port 9010**
```bash
# check status service
curl http://master-ip:9010 
{
    "version": "0.0.1",
    "message": "AGS API is running!"
}
```
- Gọi API để trigger quá trình ingest
```bash
# start ingest
curl http://master-ip:9010/ingest/seismic?api_key=Pvn2024
{
    "running": false,
    "last_run": 0
}
```
### Nạp dữ liệu Reservoir
#### Command-line mode
Under progress
### Nạp dữ liệu Khoan
Under progress
### Theo giõi, đánh giá kết quả của quá trình ingest
- API kiêm tra trạng thái xem job nào đang được chạy, khi một job đang ở trạng thái **running** sẽ không cho phép trigger một process ingest mới đối với domain tương ứng.
```
curl http://ddns.i2g.cloud:30000/status
{
    "wellbore": {
        "running": false,
        "last_run": "2024-07-15T15:27:08.210540+07:00"
    },
    "reservoir": {
        "running": false,
        "last_run": 0
    },
    "seismic": {
        "running": false,
        "last_run": 0
    },
    "welldelivery": {
        "running": false,
        "last_run": 0
    }
}
```
- API theo dõi quá trình ingest
```bash
curl http://ddns.i2g.cloud:30000/status/error
curl http://ddns.i2g.cloud:30000/status/success
```
- API docs chi tiết hơn xem tại **http://master-ip:9010/docs**