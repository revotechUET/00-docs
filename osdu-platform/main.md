---
author: 
- AGS
- Viettel
title: PVN On-premise OSDU Platform
---
# Tổng quan

TO BE FILLED

# Prerequisites
## Cấu hình phần cứng

| **STT** | **Hạng mục**                            | **Khuyến nghị** |                        **Chú thích**                        |
|---------|-----------------------------------------|:---------------:|:-----------------------------------------------------------:|
|    1    | Tổng số vCPU (core)                     |                 | Online workers of the POPGIS system                         |
|    2    | Tổng bộ nhớ RAM (GB)                    |                 | All stages of pipelines for a day                           |
|    3    | Tổng dung lượng  phân vùng dữ liệu (TB) |                 | Phân các phân vùng dữ liệu này vào các worker nodes của K8s |
|    4    | Tổng số node                            |     $\ge 5$     |                                                             |

## Các dịch vụ hạ tầng phần mềm
- Hệ điều hành: Centos 7.x hoặc mới hơn, Ubuntu 22.04 hoặc mới hơn
- Hạ tầng triển khai: 
    - Kubernetes phiên bản 1.28.x hoặc mới hơn
    - CNI Providers (Networking): Calico / Flannel
    - Service Mesh: istio phiên bản 1.20 hoặc mới hơn

- Công cụ đóng gói và triển khai: Helm phiên bản v3.13.x hoặc mới hơn