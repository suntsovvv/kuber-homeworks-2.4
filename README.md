# Домашнее задание к занятию «Управление доступом»#
## Цель задания   
В тестовой среде Kubernetes нужно предоставить ограниченный доступ пользователю.
------

### Задание 1. Создайте конфигурацию для подключения пользователя
1. Создайте и подпишите SSL-сертификат для подключения к кластеру.
Создал ключ для пользователя _testuser_, запрос на сертификат и издал сертификат.
```bash
user@microk8s:~/kuber-homeworks-2.4$ openssl genrsa -out testuser.key 2048
user@microk8s:~/kuber-homeworks-2.4$ openssl req -new -key testuser.key -out testuser.csr -subj "/CN=testuser"
user@microk8s:~/kuber-homeworks-2.4$ openssl x509 -req -in testuser.csr -CA /var/snap/microk8s/current/certs/ca.crt -CAkey /var/snap/microk8s/current/certs/ca.key -CAcreateserial -out testuser.crt -days 500
Certificate request self-signature ok
subject=CN = testuser

``` 
Создал пользователя с указанием сертификатов:
```bash
user@microk8s:~/kuber-homeworks-2.4$ kubectl config set-credentials testuser --client-certificate=testuser.crt --client-key=testuser.key --embed-certs=true 
User "testuser" set.
user@microk8s:~/kuber-homeworks-2.4$ cat ~/.kube/config 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJ....
    server: https://192.168.0.105:16443
  name: microk8s-cluster
contexts:
- context:
    cluster: microk8s-cluster
    namespace: homework
    user: admin
  name: microk8s
current-context: microk8s
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data: LS0tLS1CRUdJ.......
    client-key-data: LS0tLS1CRUdJTi.....
- name: testuser
  user:
    client-certificate-data: LS0tLS1CRU.....
    client-key-data: LS0tLS1CRUdJTiBQUk.....

```
Создал контекст:
```bash
user@microk8s:~/kuber-homeworks-2.4$ kubectl config set-context testuser --cluster=microk8s-cluster --user=testuser
Context "testuser" created.
user@microk8s:~/kuber-homeworks-2.4$ kubectl config get-contexts 
CURRENT   NAME       CLUSTER            AUTHINFO   NAMESPACE
*         microk8s   microk8s-cluster   admin      homework
          testuser   microk8s-cluster   testuser 
user@microk8s:~/kuber-homeworks-2.4$ kubectl config use-context testuser 
Switched to context "testuser".
user@microk8s:~/kuber-homeworks-2.4$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
microk8s   Ready    <none>   4d    v1.31.2
```
Задействовал RBAC:
```bash
user@microk8s:~/kuber-homeworks-2.4$ microk8s enable rbac
Infer repository core for addon rbac
Enabling RBAC
Reconfiguring apiserver
[sudo] password for user: 
Restarting apiserver
RBAC is enabled
user@microk8s:~/kuber-homeworks-2.4$ kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "testuser" cannot list resource "nodes" in API group "" at the cluster scope
```
Создал манифест с ролью и привязкой (Пользователь может просматривать логи подов и их конфигурацию (`kubectl logs pod <pod_id>`, `kubectl describe pod <pod_id>`)):
```yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: homework
  name: test-role
rules:
- apiGroups: [""] 
  resources: ["pods/log","pods"]
  verbs: ["get", "watch", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: test-role-bind
  namespace: homework
subjects:
- kind: User
  name: testuser
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: test-role
  apiGroup: rbac.authorization.k8s.io
```
Применил и переключился на контекст _testuser_:
```bash
user@microk8s:~/kuber-homeworks-2.4$ kubectl apply -f role.yaml 
role.rbac.authorization.k8s.io/test-role created
rolebinding.rbac.authorization.k8s.io/test-role-bind created
user@microk8s:~/kuber-homeworks-2.4$ kubectl config use-context testuser 
Switched to context "testuser".
```
Заранее был создан pod _test-pod_ в namespace: homework
Проверяем:
```bash
user@microk8s:~/kuber-homeworks-2.4$ kubectl logs pods/test-pod -n homework
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2024/11/14 07:00:22 [notice] 1#1: using the "epoll" event method
2024/11/14 07:00:22 [notice] 1#1: nginx/1.27.2
2024/11/14 07:00:22 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14) 
2024/11/14 07:00:22 [notice] 1#1: OS: Linux 6.8.0-48-generic
2024/11/14 07:00:22 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 65536:65536
2024/11/14 07:00:22 [notice] 1#1: start worker processes
2024/11/14 07:00:22 [notice] 1#1: start worker process 29
2024/11/14 07:00:22 [notice] 1#1: start worker process 30
user@microk8s:~/kuber-homeworks-2.4$ kubectl describe  pods/test-pod -n homework
Name:             test-pod
Namespace:        homework
Priority:         0
Service Account:  default
Node:             microk8s/192.168.0.105
Start Time:       Thu, 14 Nov 2024 07:00:20 +0000
Labels:           <none>
Annotations:      cni.projectcalico.org/containerID: 9ebc48565e5d5808dbb4d73cb68562574f1a21ce32520d2dbe02a77044c187e9
                  cni.projectcalico.org/podIP: 10.1.128.237/32
                  cni.projectcalico.org/podIPs: 10.1.128.237/32
Status:           Running
IP:               10.1.128.237
IPs:
  IP:  10.1.128.237
Containers:
  nginx:
    Container ID:   containerd://0a6af38b430f9fa5b93fcba51ef58e42fb2d4d32b658d9bc1c5765050acad979
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:bc5eac5eafc581aeda3008b4b1f07ebba230de2f27d47767129a6a905c84f470
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 14 Nov 2024 07:00:22 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-5vdml (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-5vdml:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
user@microk8s:~/kuber-homeworks-2.4$ kubectl delete  pods/test-pod -n homework
Error from server (Forbidden): pods "test-pod" is forbidden: User "testuser" cannot delete resource "pods" in API group "" in the namespace "homework"
user@microk8s:~/kuber-homeworks-2.4$ kubectl delete  pods/test-pod -n homework
Error from server (Forbidden): pods "test-pod" is forbidden: User "testuser" cannot delete resource "pods" in API group "" in the namespace "homework"
user@microk8s:~/kuber-homeworks-2.4$ kubectl get svc -n homework
Error from server (Forbidden): services is forbidden: User "testuser" cannot list resource "services" in API group "" in the namespace "homework"
user@microk8s:~/kuber-homeworks-2.4$ kubectl get ingress -n homework
Error from server (Forbidden): ingresses.networking.k8s.io is forbidden: User "testuser" cannot list resource "ingresses" in API group "networking.k8s.io" in the namespace "homework"
```

2. Настройте конфигурационный файл kubectl для подключения.
3. Создайте роли и все необходимые настройки для пользователя.
4. Предусмотрите права пользователя. Пользователь может просматривать логи подов и их конфигурацию (`kubectl logs pod <pod_id>`, `kubectl describe pod <pod_id>`).
5. Предоставьте манифесты и скриншоты и/или вывод необходимых команд.
------
