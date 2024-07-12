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

# Cài đặt
## Chuẩn bị bộ cài đặt

### Tải xuống helm chart của OSDU baremetal

```bash
helm pull oci://community.opengroup.org:5555/osdu/platform/deployment-and-operations/infra-gcp-provisioning/gc-helm/osdu-gc-baremetal --version 0.25.6
```

- Output: **osdu-gc-baremetal-0.25.6.tgz**

### Giải nén file: 

```bash
tar -xvzf osdu-gc-baremetal-0.25.6.tgz
```

- Khi này được thư mục chart của osdu-gc-baremetal

### Cập nhật image cho các chart

- Mặc định **osdu-gc-baremetal** sửa dụng các container image được lưu trữ trên cloud, nên trong trường hợp cụm server cài đặt osdu không có internet thì cần phải lưu các image này về private registry hub ở local, có thể dùng **docker private registry** hoặc **habor private registry**

- Việc cần làm là tìm tất cả các image này trong thư mục **osdu-gc-baremetal** rồi thay đổi chúng. Ví dụ trong file **osdu-gc-baremetal/charts/minio/values.yaml**:

```yaml
image:
    registry: community.opengroup.org:5555/osdu/platform/deployment-and-operations/base-containers-gcp/bitnami-common-library-helm-chart
    repository: gc-bitnami-shell
    tag: 11-debian-11-r11
```
- Thực hiện sửa "__community.opengroup.org:5555/osdu/platform/deployment-and-operations/base-containers-gcp/bitnami-common-library-helm-chart__" thành "__10.10.10.10/osdu/platform/deployment-and-operations/base-containers-gcp/bitnami-common-library-helm-chart__"

## Trình tự thực hiện
- Chuẩn bị ổ lưu trữ cho cụm k8s. Tạo các pv/pvc cho các ứng dụng `minio, postgresql, elasticsearch, elasticsearch-data, rabbitmq`
- Chuẩn bị một domain name: (dns server/hostfile)
- Chuẩn bị file `customs-values.yaml`. Thay đổi nội dung cho phù hợp với môi trường sẽ deploy: `global.domain`, `keycloak.auth.adminPassword`, ...Lưu ý thay đổi persistence size tương ứng với dung lương của các pv/pvc đã được tạo

```yaml
global:
  domain: "pvn.local"
  # Configuration parameter to switch between HTTP and HTTPS mode for external endpoint.
  # Default - HTTP. HTTPS requires additional configuration
  useHttps: false

keycloak:
  auth:
    # Fill in variable value, the value should contain only alphanumerical characters and should be at least 8 symbols
    adminPassword: "Pvn123456"
  # This value should be set to 'none' if https is not used (global.useHttps = false), otherwise the value needs to be changed to 'passthrough' if https is used (global.useHttps = true)
  proxy: none

minio:
  auth:
    # Fill in variable value
    rootPassword: "Pvn123456"
  persistence:
    size: 200Gi
  # This value should be set to 'true' when using self-signed certificates or installing on minikube and docker desktop
  #useInternalServerUrl: true

postgresql:
  global:
    postgresql:
      auth:
        # Fill in variable value
        postgresPassword: "Pvn123456"
  persistence:
    size: 20Gi

airflow:
  externalDatabase:
    # Fill in variable value
    password: "Pvn123456"
  auth:
    # Fill in variable value
    password: "Pvn123456"

elasticsearch:
  security:
    # Fill in variable value
    elasticPassword: "Pvn123456"
  master:
    persistence:
      size: 20Gi
  data:
    replicaCount: 2
    persistence:
      size: 20Gi

rabbitmq:
  auth:
    # Fill in variable value
    password: "Pvn123456"

gc_oetp_client_deploy:
  enabled: true
gc_oetp_server_deploy:
  enabled: true
```
- Apply helm chart
```bash
helm install -f custom-values.yaml osdu-baremetal osdu-gc-baremetal -nosdu
```
- Sau khi apply, các k8s pod sẽ được khởi tạo và có trạng thái `Pennding` hoặc `CrashLoopBackoff`, điều này là bình thường,cần đợi khoảng 30p-1h cho các app ready

## Các lưu ý khi thực hiện apply 

- Chỉ được coi là thành công khi toàn bộ các pod đều trong trạng thái Running. Nếu pod nào lỗi cần phải inspect pod đó và tiến hành sửa lỗi
- Nếu 1 số pod bị stuck ở trạng thái `CrashLoopBackoff` quá lâu, có thể tiến hành `rollout restart` toàn bộ `deployments` và `statefullset` của cụm rồi chờ đợi

```
NAME                                             READY   STATUS                        RESTARTS       AGE
airflow-bootstrap-deployment-5c779c47d9-8zrm6    1/1     Running                       0              2d3h
airflow-scheduler-75468bf948-gdbz7               1/1     Running                       0              2d3h
airflow-web-5884687c64-cr7m6                     1/1     Running                       2 (2d3h ago)   2d3h
config-f4c55ff79-df98c                           1/1     Running                       0              2d3h
crs-catalog-596f4d95c5-k8hvz                     1/1     Running                       0              2d3h
crs-conversion-5d95794975-6s8g8                  1/1     Running                       0              2d3h
dataset-7c9767f86b-r5mmd                         2/2     Running                       0              2d3h
eds-dms-6875754fdc-2ndr7                         1/1     Running                       1 (2d3h ago)   2d3h
elastic-bootstrap-deployment-6c59ff4bd8-ccp44    1/1     Running                       0              2d3h
elasticsearch-0                                  1/1     Running                       14 (9d ago)    9d
elasticsearch-data-0                             1/1     Running                       14 (9d ago)    9d
entitlements-757f95c4db-v2gml                    2/2     Running                       0              2d3h
entitlements-bootstrap-b4557986-mx2zz            1/1     Running                       0              2d3h
file-856bc74bf-qrdq9                             2/2     Running                       1 (2d3h ago)   2d3h
indexer-6896d78dcc-fsxnl                         2/2     Running                       0              2d3h
keycloak-bootstrap-deployment-86bbd4db8c-kdplr   1/1     Running                       0              2d3h
legal-6444fd66d6-zvrwn                           1/1     Running                       1 (2d3h ago)   2d3h
minio-bootstrap-deployment-8d4d8f86c-wm6p4       1/1     Running                       0              2d3h
notification-869555ccff-llq8d                    1/1     Running                       4 (2d3h ago)   2d3h
oetp-client-544945fbc6-57tg7                     1/1     Running                       0              2d3h
oetp-server-5c44786446-sd6mf                     1/1     Running                       0              2d3h
opa-5bbc957bd5-dljqw                             1/1     Running                       0              2d
partition-55d7bd578c-rcjbr                       2/2     Running                       0              2d3h
partition-bootstrap-d5c85f56d-brdwm              2/2     Running                       1 (2d3h ago)   2d3h
policy-7b5b756f68-b5jdw                          1/1     Running                       0              2d
policy-7b5b756f68-x2sbt                          1/1     Running                       0              2d
policy-bootstrap-5b8dd575bc-7ptxg                1/1     Running                       0              2d
postgres-bootstrap-deployment-f9cbf56c4-kjtqb    1/1     Running                       0              2d3h
postgresql-db-0                                  1/1     Running                       0              9d
rabbitmq-0                                       1/1     Running                       0              9d
rabbitmq-bootstrap-deployment-8899f6674-d66vf    1/1     Running                       0              2d3h
redis-dataset-dc7967b8b-74xht                    1/1     Running                       0              2d3h
redis-entitlements-fc4f6f787-4gnpm               1/1     Running                       0              2d
redis-indexer-849d496f9-tbjpq                    1/1     Running                       0              2d3h
redis-notification-76d49cdb9-h4xlq               1/1     Running                       0              2d3h
redis-search-65db47f54b-vs6fr                    1/1     Running                       0              2d
redis-seismic-store-7dfffd9779-pqrhw             1/1     Running                       0              2d3h
redis-storage-56f4b87b4f-t8xsk                   1/1     Running                       0              2d3h
register-6f6f85b9c5-pnjkm                        1/1     Running                       0              2d3h
schema-5558986c79-w78g5                          1/1     Running                       0              2d3h
schema-bootstrap-757578f8cd-88pwk                1/1     Running                       1 (2d3h ago)   2d3h
search-69b8c66f4f-fwxgg                          1/1     Running                       0              2d
secret-6b8dc4f9d4-nq2lv                          1/1     Running                       0              2d3h
seismic-store-86dffdc4c-nzjrw                    1/1     Running                       0              2d3h
storage-6f5b5cf56c-8zdcs                         1/1     Running                       0              2d3h
storage-6f5b5cf56c-d92k5                         1/1     Running                       0              2d3h
storage-6f5b5cf56c-lff65                         1/1     Running                       1 (2d3h ago)   2d3h
unit-b4859cd8b-th4nd                             1/1     Running                       0              2d3h
well-delivery-56ff57d454-9cc5g                   1/1     Running                       2 (2d3h ago)   2d3h
wellbore-665cd4b56f-jsnpn                        1/1     Running                       0              2d3h
workflow-69d66cb7f6-cdj49                        1/1     Running                       0              2d3h
workflow-bootstrap-98c565747-c7gc7               1/1     Running                       7 (2d3h ago)   2d3h
```

## Các bước thực hiện sau cài đặt (Post installation)
- Tạo keycloak client, user, password

- Add user vào groups

## Kiểm tra hệ thông sau khi cài đặt