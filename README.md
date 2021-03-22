# traefik-header-route-demo

traefik

## treafik部署

参考文档：https://cloud.tencent.com/document/product/457/32730
![image](https://user-images.githubusercontent.com/42019725/111931943-4ee33380-8af7-11eb-9ccd-211b44677f9b.png)


## 服务部署
案例demo中，将部署3个服务（V1/v2/v3）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      version: v1
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: "openresty/openresty:centos"
        ports:
        - name: http
          protocol: TCP
          containerPort: 80
        volumeMounts:
        - mountPath: /usr/local/openresty/nginx/conf/nginx.conf
          name: config
          subPath: nginx.conf
      volumes:
      - name: config
        configMap:
          name: nginx-v1
---
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: nginx
    version: v1
  name: nginx-v1
data:
  nginx.conf: |-
    worker_processes  1;
    events {
        accept_mutex on;
        multi_accept on;
        use epoll;
        worker_connections  1024;
    }
    http {
        ignore_invalid_headers off;
        server {
            listen 80;
            location / {
                access_by_lua '
                    local header_str = ngx.say("nginx-v1")
                ';
            }
        }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-v1
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
    name: http
  selector:
    app: nginx
    version: v1
```

参考上面yaml示例。

## 基于Header划分请求

```yaml
kind: IngressRoute
metadata:
  name: ingressroute1
  namespace: default
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`your.example.com`) && Headers(`Version`,`v1`)
    kind: Rule
    services:
    - name: nginx-v1
      port: 80
```

## 测试

```shell

$ curl -H "Host: your.example.com" -H "Version: v1" http://<EXTERNAL-IP>
nginx-v1

$ curl -H "Host: your.example.com" -H "Version: v2" http://<EXTERNAL-IP>
nginx-v2

$ curl -H "Host: your.example.com" -H "Version: v3" http://<EXTERNAL-IP>
nginx-v3

```
