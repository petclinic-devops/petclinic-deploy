# 🔗 Kết nối Jenkins EC2 với Kubernetes Cluster

## 1️⃣ Thiết lập kubeconfig trên Jenkins EC2

| Bước | Hành động | Lệnh / Thao tác |
|------|-----------|----------------|
| 1 | Tạo thư mục `.kube` | ```bash sudo mkdir -p /home/jenkins/.kube ``` |
| 2 | Tạo file `config` | ```bash sudo vim /home/jenkins/.kube/config ``` |
| 3 | Dán nội dung kubeconfig từ master node | 1. Trên **master node**, chạy: <br> ```bash cat ~/.kube/config ``` <br> 2. Copy toàn bộ nội dung (bao gồm `apiVersion`, `clusters`, `contexts`, `users`). <br> 3. Quay lại **vim trên Jenkins EC2**, nhấn `i` → dán nội dung → nhấn `ESC` → gõ `:wq` để lưu. |
| 4 | Chỉnh quyền file | ```bash sudo chown jenkins:jenkins /home/jenkins/.kube/config ``` <br> ```bash sudo chmod 600 /home/jenkins/.kube/config ``` |

> **Lưu ý:** Quyền `600` đảm bảo chỉ user `jenkins` có thể đọc/ghi file config, tránh lỗi truy cập.


## 2️⃣ Kiểm tra kết nối với Kubernetes Cluster

# Chuyển sang user Jenkins
- sudo su - jenkins 
- chown jenkins:jenkins /home/jenkins/.kube/config 
- chmod 600 /home/jenkins/.kube/config

# Export kubeconfig
- export KUBECONFIG=/home/jenkins/.kube/config

# Kiểm tra nodes
- kubectl get nodes
# ✅ Nếu hiển thị danh sách nodes, nghĩa là Jenkins EC2 đã kết nối thành công với Kubernetes Cluster.

---
![Kết nối Jenkins với Kubernetes](https://github.com/user-attachments/assets/26f11e59-ec9e-4d15-8ef5-ad5667d5de62)
---

# CD Pipeline từ Jenkins cho Petclinic

Tiếp theo sẽ thực hiện quy trình CD từ Jenkins:

- Tạo pipeline project `CD-petclinic-` như hình:  
  ![Pipeline Project](https://github.com/user-attachments/assets/c0291971-b92d-4491-b9de-f509716305b8)

- Pipeline sẽ **checkout repo `petclinic-devops`** và deploy các service lên cluster K8s bằng các manifest đã cấu hình sẵn.

---

## Jenkinsfile / Pipeline

```groovy
pipeline {
    agent any

    environment {
        NAMESPACE = 'petclinic'
        REPO_DIR = '/var/lib/jenkins/workspace/CD-petclinic'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/petclinic-devops/petclinic-deploy.git', credentialsId: 'github-https-cred'
            }
        }

        stage('Deploy K8s') {
            steps {
                echo "Deploying workloads to namespace ${NAMESPACE}"
                sh """
                # Tạo namespace trước
                kubectl apply -f ${REPO_DIR}/k8s/namespaces.yaml

                # Deploy secrets
                kubectl apply -f ${REPO_DIR}/k8s/secret/openai-secret.yaml -n ${NAMESPACE}

                # Deploy workloads
                kubectl apply -f ${REPO_DIR}/k8s/workload/config-server-deployment.yaml -n ${NAMESPACE}
                kubectl apply -f ${REPO_DIR}/k8s/workload/config-server-service.yaml -n ${NAMESPACE}

                kubectl apply -f ${REPO_DIR}/k8s/workload/discovery-server-deployment.yaml -n ${NAMESPACE}
                kubectl apply -f ${REPO_DIR}/k8s/workload/discovery-server-service.yaml -n ${NAMESPACE}

                kubectl apply -f ${REPO_DIR}/k8s/workload/customers-service-deployment.yaml -n ${NAMESPACE}
                kubectl apply -f ${REPO_DIR}/k8s/workload/customers-service-service.yaml -n ${NAMESPACE}

                kubectl apply -f ${REPO_DIR}/k8s/workload/visits-service-deployment.yaml -n ${NAMESPACE}
                kubectl apply -f ${REPO_DIR}/k8s/workload/visits-service-service.yaml -n ${NAMESPACE}

                kubectl apply -f ${REPO_DIR}/k8s/workload/vets-service-deployment.yaml -n ${NAMESPACE}
                kubectl apply -f ${REPO_DIR}/k8s/workload/vets-service-service.yaml -n ${NAMESPACE}

                kubectl apply -f ${REPO_DIR}/k8s/workload/api-gateway-deployment.yaml -n ${NAMESPACE}
                kubectl apply -f ${REPO_DIR}/k8s/workload/api-gateway-service.yaml -n ${NAMESPACE}

                kubectl apply -f ${REPO_DIR}/k8s/workload/admin-server-deployment.yaml -n ${NAMESPACE}
                kubectl apply -f ${REPO_DIR}/k8s/workload/admin-server-service.yaml -n ${NAMESPACE}

                kubectl apply -f ${REPO_DIR}/k8s/workload/genai-server-deployment.yaml -n ${NAMESPACE}
                kubectl apply -f ${REPO_DIR}/k8s/workload/genai-server-service.yaml -n ${NAMESPACE}

                kubectl apply -f ${REPO_DIR}/k8s/workload/tracing-server-deployment.yaml -n ${NAMESPACE}
                kubectl apply -f ${REPO_DIR}/k8s/workload/tracing-server-service.yaml -n ${NAMESPACE}
                """
            }
        }

        stage('Verify') {
            steps {
                sh "kubectl get pods -n ${NAMESPACE}"
                sh "kubectl get svc -n ${NAMESPACE}"
            }
        }
    }

    post {
        success { 
            echo '✅ Deployment successful!' 
            script {
                def pods = sh(script: "kubectl get pods -n ${NAMESPACE} --no-headers | wc -l", returnStdout: true).trim()
                def runningPods = sh(script: "kubectl get pods -n ${NAMESPACE} --field-selector=status.phase=Running --no-headers | wc -l", returnStdout: true).trim()
                def buildUrl = env.BUILD_URL
                def time = new Date().format("yyyy-MM-dd HH:mm:ss")

                slackSend(
                    channel: '#social',
                    color: 'good',
                    message: """
✅ Deployment SUCCESSFUL!
*Namespace:* ${NAMESPACE}
*Time:* ${time}
*Pods Running / Total:* ${runningPods} / ${pods}
*Jenkins Build:* ${buildUrl}
                    """
                )
            }
        }
        failure { 
            echo '❌ Deployment failed!' 
            script {
                def pods = sh(script: "kubectl get pods -n ${NAMESPACE} --no-headers | wc -l", returnStdout: true).trim()
                def runningPods = sh(script: "kubectl get pods -n ${NAMESPACE} --field-selector=status.phase=Running --no-headers | wc -l", returnStdout: true).trim()
                def buildUrl = env.BUILD_URL
                def time = new Date().format("yyyy-MM-dd HH:mm:ss")

                slackSend(
                    channel: '#social',
                    color: 'danger',
                    message: """
❌ Deployment FAILED!
*Namespace:* ${NAMESPACE}
*Time:* ${time}
*Pods Running / Total:* ${runningPods} / ${pods}
*Jenkins Build:* ${buildUrl}
                    """
                )
            }
        }
    }
}
```

***Lưu ý quan trọng bạn phải có credentials để đăng nhập GitHub, cluster K8s, Slack***
<img width="6532" height="828" alt="image" src="https://github.com/user-attachments/assets/c22da7cd-f9c5-40e7-8511-59c3b29f8d6a" />
<img width="5656" height="2144" alt="image" src="https://github.com/user-attachments/assets/c06e0539-1328-44e1-bd7f-41fce6f00820" />
<img width="2136" height="1748" alt="image" src="https://github.com/user-attachments/assets/44def3fc-b398-4759-bfc3-faac8af35084" />

# 🔗 Kết nối Grafana + Prometheus với Cluster Kubernetes bằng Helm

## 1️⃣ Cài đặt `kube-prometheus-stack`

Trên máy Master Node, chạy các lệnh sau:

```bash
# Thêm Helm repo của Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Cập nhật repo
helm repo update

# Cài đặt kube-prometheus-stack
helm install kube-prometheus-stack \
  --create-namespace \
  --namespace kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack

# Kiểm tra các Pod trong namespace kube-prometheus-stack
kubectl -n kube-prometheus-stack get pods
```
## 2️⃣ Chạy Prometheus
```bash
kubectl port-forward -n kube-prometheus-stack svc/kube-prometheus-stack-prometheus 9090:9090
```
- Sau khi chạy lệnh trên, Prometheus sẽ có thể truy cập tại http://localhost:9090.
<img width="7652" height="2708" alt="image" src="https://github.com/user-attachments/assets/220c44ce-675d-42e4-b1ae-2b944f5db742" />

## 3️⃣ Hiển thị Grafana
```bash
kubectl port-forward -n kube-prometheus-stack svc/kube-prometheus-stack-grafana 8080:80
```
- Sau khi chạy lệnh trên, Grafana sẽ có thể truy cập tại http://localhost:8080.
- Tài khoản mặc định:
- Username: admin
- Password: prom-operator
<img width="7668" height="3996" alt="image" src="https://github.com/user-attachments/assets/779239f7-11a8-4e4a-a6d0-6d51856d7c49" />

# TÀI LIỆU THAM KHẢO
- https://spacelift.io/blog/prometheus-kubernetes
- https://grafana.com/docs/grafana/latest/setup-grafana/installation/helm/
- https://www.youtube.com/watch?v=9ZUy3oHNgh8&t=614s

