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
- Bước 1: Tải source code ingestion-tool sau đó giải nén 
```bash
wget https://file.i2g.cloud/osdu/osdu-ingest.tar.gz 
tar -xvzf osdu-ingest.tar.gz
```
- Bước 2: Buid docker image, ví dụ IP habor: **10.10.10.1**
```bash
docker build -t 10.10.10.1/osdu/osdu-ingest:latest .
```
- Bước 3: Publish image lên habor registry
```bash
docker push 10.10.10.1/osdu/osdu-ingest:latest
```
# Triển khai PVN OSDU Ingestion Tools vào hạ tầng K8S
Sau khi PVN OSDU Ingestion Tools image được build thành công. Image này cần được triển khai vào hạ tầng K8S. Các bước thực hiện được trình bày trong các tiểu mục sau

## Chuẩn bị thông tin triển khai 
- Địa chỉ NFS server (chứa data phục vụ cho ingestion). Ví dụ: **192.168.1.15/volume1/osdu-ingest**
- Storage: Tạo pvc mount nfs storage **ags-pyosdu-pvc.yaml**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: osdu-data-ingest
spec:
  storageClassName: osdu-data-ingest
  persistentVolumeReclaimPolicy: Retain
  capacity:
    storage: 1000Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: "/volume1/osdu-ingest"
    server: "192.168.1.15"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: osdu-data-ingest-pvc
spec:
  storageClassName: osdu-data-ingest
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1000Gi
```
- File manifest của ingestion tool **ags-pyosdu.yaml**. Lưu ý thay thông tin tương ứng với cụm k8s. **hostAliases**, các biến môi trường tuỳ chỉnh. Data ingestion được mount vào thư mục **/data-ingest** của pod
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ags-pyosdu
  namespace: osdu
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ags-pyosdu
  template:
    metadata:
      labels:
        app: ags-pyosdu
    spec:
      volumes:
      - name: nfs
        persistentVolumeClaim:
          claimName: ags-pyosdu
      - name: data-ingest
        persistentVolumeClaim:
          claimName: osdu-data-ingest-pvc
      hostAliases:
      - ip: 192.168.1.80 # địa chỉ IP của osdu master node hoặc VIP của cụm masters
        hostnames:
        - osdu.pvn.local
        - s3.pvn.local
        - keycloak.pvn.local
        - airflow.pvn.local
        - minio.pvn.local
      containers:
      - name: ags-pyosdu
        # habor pyosdu image
        image: 10.10.10.1/osdu/ags/py-osdu:v1.4
        imagePullPolicy: IfNotPresent
        command: ["python"]
        args: ["-m", "uvicorn", "server.server:app","--host", "0.0.0.0", "--port", "8000"]
        ports:
        - containerPort: 8000
        volumeMounts:
        - name: nfs
          mountPath: /app
        - name: data-ingest
          mountPath: /data-ingest
        workingDir: /app
        env:
        # Thông tin minio của cụm osdu
        - name: MC_HOST_minio
          value: "http://minio:12345678@s3.osdu.i2g.cloud"
        # Địa chỉ keycloak
        - name: AUTH_URL
          value: https://keycloak.i2g.cloud
        # Keycloak token fetch url
        - name: TOKEN_FETCH_URL
          value: https://keycloak.i2g.cloud/realms/osdu/protocol/openid-connect/token
        # Địa chỉ osdu api
        - name: OSDU_BASE
          value: http://osdu.osdu.i2g.cloud
        # Client id của ứng dụng ingestion
        - name: CLIENT_ID
          value: wi
        # Client secret
        - name: CLIENT_SECRET
          value: langnhang
        # User của ứng dụng ingestion
        - name: USERNAME
          value: wi@osdu.i2g.cloud
        # Mật khẩu
        - name: PASSWORD
          value: "123456"
        # OSDU partition
        - name: PARTITION_ID
          value: osdu
        # Keycloak oauth base url
        - name: OAUTH_BASE_URL
          value: https://keycloak.i2g.cloud/realms/osdu/protocol/openid-connect/auth
        - name: AUTH_METHOD
          value: OIDC_LOGIN
        - name: ADMIN_CLIENT_ID
          value: datafier
        - name: ADMIN_CLIENT_SECRET
          value: brXszXvfhVJNWJqa
        - name: MINIO_ENDPOINT
          value: http://$(MINIO_SERVICE_HOST):$(MINIO_SERVICE_PORT)
        - name: MINIO_ACCESS_KEY
          value: minio
        - name: MINIO_SECRET_KEY
          value: "12345678"
        - name: OETP_URL
          value: http://oetp-server:9002
        - name: SERVER_AUTHENTICATE
          value: "True"
        # API Key của app ingestion
        - name: SERVER_API_KEY
          value: "Pvn2024"
        - name: SERVER_INGEST_DATA_PATH
          #value: "/app/demo-data/"
          value: "/data-ingest/"
        - name: PATH
          value: "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/root/.local/lib/python3.10/site-packages/bin"
        # User admin keycloak
        - name: MASTER_USERNAME
          value: user
        # Password admin keycloak
        - name: MASTER_PASSWORD
          value: "@Revotech123"
---
apiVersion: v1
kind: Service
metadata:
  name: ags-pyosdu-svc
spec:
  type: NodePort
  selector:
    app: ags-pyosdu
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 8000
    nodePort: 30000
```

## Thực hiện triển khai
```bash
kubectl apply -f ags-pyosdu-pvc.yaml
kubectl apply -f ags-pyosdu.yaml
```
## Kiểm tra hệ thống sau khi cài đặt
```bash
kubectl get pods | grep ags-pyosdu
ags-pyosdu-858d7989d5-b4mzl                      1/1     Running                       0              4d19h
```