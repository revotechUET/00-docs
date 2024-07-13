---
author: 
- AGS
- Viettel
title: OSDU Explorer - Hướng dẫn cài đặt và vận hành
---
# Tổng quan
Tài liệu này hướng dẫn cài đặt ứng dụng OSDU Explorer vào hạ tầng K8S

# Điều kiện tiên quyết
Để OSDU Explorer hoạt động cần có PVN On-premise OSDU Platform (Xem tài liệu osdu-platform-main)

# Cài đặt
## Chuẩn bị
### Bộ code đã được build
- Tải xuống build code 
```bash
wget https://file.i2g.cloud/osdu/osdu-client.tar.gz
```
- Giải nén file
```bash
tar -xvzf osdu-client.tar.gz
```
### Các file k8s manifest
- File **pvc.yaml**: File này tạo pvc storage cho pod của osdu-client,dùng để mount thư mục build code vào trong pod. Có thể tuỳ chỉnh tuỳ theo cách lưu trữ của từng cụm k8s. Dung lượng khuyến nghị 5Gi. Sau đó copy build code đã chuẩn bị vào thư mục mount này.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ags-osdu-client
  namespace: osdu
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client
  resources:
    requests:
      storage: 5Gi
status:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 5Gi
  phase: Bound
```
- File **web-server.yaml**: File này deploy một webserver nginx, dùng để serve trang osdu-client
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ags-osdu-client
  namespace: osdu
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        volumeMounts:
        - name: nfs
          mountPath: /usr/share/nginx/html
        ports:
        - containerPort: 80
      volumes:
      - name: nfs
        persistentVolumeClaim:
          claimName: ags-osdu-client
```
- File **oclient-istio.yaml**: File này deploy một domain istio để truy cập vào webserver qua domain __http://ags-oclient.pvn.local__ từ bên ngoài, hoặc qua service nodePort 9080 của k8s
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ags-osdu-client-svc
  namespace: osdu
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 9080
---
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: ags-osdu-client-gateway
  namespace: osdu
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - ags-oclient.pvn.local
    port:
      name: http
      number: 80
      protocol: HTTP
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ags-osdu-client
  namespace: osdu
spec:
  gateways:
  - ags-osdu-client-gateway
  hosts:
  - ags-oclient.pvn.local
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: ags-osdu-client-svc
        port:
          number: 80
```
## Quy trình thực hiện
```bash
kubectl apply -f pvc.yaml
kubectl apply -f web-server.yaml
kubectl apply -f oclient-istio.yaml
```
# Kết quả cài đặt
### Từ trình duyệt truy cập được vào trang web http://ags-oclient.pvn.local và đăng nhập tài khoản osdu thành công
