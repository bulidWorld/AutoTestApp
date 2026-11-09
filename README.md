# AutoTest App：Kubernetes 双镜像部署

前端镜像中的 Nginx 提供 Vue 构建产物，并将 `/api/` 请求代理到集群内的 `autotest-backend:31010`。后端镜像用 Uvicorn 运行 FastAPI。前端 Service 使用 NodePort `30111` 对外开放；后端 Service 保持 ClusterIP，不需要 Ingress Controller。后端需要已初始化的外部 PostgreSQL。

请求路径（假设节点 IP 为 `192.168.100.143`）：

- 页面：`http://192.168.100.143:30111/` → 前端 Service:80 → Nginx:8090。
- API：`http://192.168.100.143:30111/api/v1/health` → 前端 Service:80 → Nginx:8090 → 后端 Service:31010 → FastAPI:31010。

NodePort 可从集群可达的节点 IP 访问，前提是节点防火墙和集群网络允许 `30111`。前后端 Deployment 和 Service 应部署在同一 namespace；前端 Nginx 使用后端 Service 的 DNS 名称。

## 构建镜像

在仓库根目录执行，并将仓库地址和版本换成实际值：

```bash
docker build -t YOUR_REGISTRY/autotest-backend:1.0.0 ./AutoTestService
docker build -t YOUR_REGISTRY/autotest-frontend:1.0.0 ./AutoTestWebsite/frontend
docker push YOUR_REGISTRY/autotest-backend:1.0.0
docker push YOUR_REGISTRY/autotest-frontend:1.0.0
```

两个 Deployment 的 `image` 应与推送地址一致。

## 部署

先将 [后端配置示例](AutoTestService/deploy/k8s/backend-config.example.yaml) 中的数据库主机等改为实际值，并在目标 namespace 创建 Secret，键名为 `database.secret`。下例使用 `autotest` namespace；请使用安全方式准备本地密码文件：

```bash
kubectl create namespace autotest --dry-run=client -o yaml | kubectl apply -f -
kubectl -n autotest create secret generic autotest-backend-db --from-file=database.secret=./database.secret
kubectl -n autotest apply -f AutoTestService/deploy/k8s/backend-config.example.yaml
kubectl -n autotest apply -f AutoTestService/deploy/k8s/backend.yaml
kubectl -n autotest apply -f AutoTestWebsite/frontend/deploy/k8s/frontend.yaml
```

先创建后端 Service，再启动前端 Nginx，以便解析 `autotest-backend`。确认本集群的 NodePort 范围包含 `30111`。前端开发服务器的 Vite 代理不会进入生产镜像。
