# 🔗 Kết nối Jenkins EC2 với Kubernetes Cluster

## 1️⃣ Thiết lập kubeconfig trên Jenkins EC2

| Bước | Hành động | Lệnh / Thao tác |
|------|-----------|----------------|
| 1 | Tạo thư mục `.kube` | ```bash sudo mkdir -p /home/jenkins/.kube ``` |
| 2 | Tạo file `config` | ```bash sudo vim /home/jenkins/.kube/config ``` |
| 3 | Dán nội dung kubeconfig từ master node | 1. Trên **master node**, chạy: <br> ```bash cat ~/.kube/config ``` <br> 2. Copy toàn bộ nội dung (bao gồm `apiVersion`, `clusters`, `contexts`, `users`). <br> 3. Quay lại **vim trên Jenkins EC2**, nhấn `i` → dán nội dung → nhấn `ESC` → gõ `:wq` để lưu. |
| 4 | Chỉnh quyền file | ```bash sudo chown jenkins:jenkins /home/jenkins/.kube/config ; sudo chmod 600 /home/jenkins/.kube/config ``` |

> **Lưu ý:** Quyền `600` đảm bảo chỉ user `jenkins` có thể đọc/ghi file config, tránh lỗi truy cập.

---

## 2️⃣ Kiểm tra kết nối với Kubernetes Cluster

```bash
sudo su - jenkins
chown jenkins:jenkins /home/jenkins/.kube/config
chmod 600 /home/jenkins/.kube/config
export KUBECONFIG=/home/jenkins/.kube/config
kubectl get nodes
✅ Nếu hiển thị danh sách nodes, nghĩa là Jenkins EC2 đã kết nối thành công với Kubernetes Cluster.
