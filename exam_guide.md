# 시험 문제 유형

## Youtube ttabae-learn 의 CKA 강의자료를 참고하여 작성한 내용입니다.
https://www.youtube.com/@ttabae-learn4274

## ETCD Backup & Restore
ETCD 정보는 기본적으로 /var/lib/etcd 에 저장된다.
작업시에는 sudo 명령어를 사용해야 한다.(root 권한 필요)
### 제공정보
- 작업할 current-context 정보 (ex: k8s-master)
- Backup / Restore 파일이 위치할 경로 (ex: /data/etcd-snapshot.db, /data/etcd-snapshot-previous.db)
- etcdctl 접속에 필요한 ca.crt / server.crt / server.key 파일 위치
### 풀이
1. `kubectl config current-context` 로 현재 node 확인 후, 문제에서 제공한 시스템(node)으로 접속 `ssh k8s-master`
2. /var/lib/etcd 의 etcd 정보를 문제에서 요청한 위치에 backup 한다.
    ```shell
    sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
      --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
      snapshot save <backup-file-location>
    ```
3. restore 할 .db 파일을 /var/lib/etcd_restore 위치로 restore 명령을 통해 가져온다.
    ```shell
    sudo ETCDCTL_API=3 etcdctl snapshot restore --data-dir <destination-data-dir-location> <target-snapshotdb-file-path>
    ```
4. etcd 가 바라보는 경로를 /var/lib/etcd 에서 /etcd_resotre 로 변경한다.  
   etcd yaml 파일에 대한 수정이 필요함. `/etc/kubernetes/manifests/etcd.yaml` 경로에서 찾을 수 있음  
   vi 로 yaml 파일에서 volumes -> hostPath 에 잡힌 etcd 저장소 경로를 restore 한 파일이 풀린 경로로 지정한다.
5. `sudo docker ps -a | grep etcd` 명령을 통해 etcd 의 상태가 모두 `up` 상태가 되는것을 확인한다.
   중간에 계속해서 `Exited` 상태가 표현될 수 있음. 기다려야함.
6. `kubectl get pods` 를 통해 정상적으로 작동하는지 확인.

## Pod 생성하기
### 제공정보
- 작업 클러스터: k8s
- new namespace: ecommerce
- pod name: eshop-main
- image: nginx:1.17
- env: DB=mysql

### 풀이
1. context 변경 `kubectl config use-context k8s`
2. namespace 생성 `kubectl create namespace ecommerce`
3. pod 생성 `kubectl run pod eshop-main --image=nginx:1.17 --env=DB=mysql --namespace=ecommerce --dry-run=client -o yaml`  
  dry run 으로 잘 동작되나 한번 해보고 수행하는것이 더 좋다.

## Statice Pod 생성하기
static pod 는 api 서버 없이 노드의 kubelet 에 의해 관리되는 파드이다.  
기본으로는 /etc/kubernetes/manifest 디렉토리에 static pod 에 대한 yaml 이 관리된다.  
etcd, api, controller 등이 대표적이 static pod 이다.  
/etc/kubernetes/manifest 경로에 yaml 파일이 생성되면, 자동으로 apply 된다.
### 제공정보
- node: hk8s-w1
- pod name: web
- image: nginx
### 풀이
1. 작업할 node 로 접속 `ssh hk8s-w1`
2. static pod 를 생성하려면 root 권한이 필요하다 `sudo -i`
3. static pod yaml 파일 저장경로 확인 `cat /var/lib/kubelet/config.yaml | grep staticPodPath`
2. pod 생성 명령을 통해 yaml 파일 생성 `kubectl run web --image=nginx --dry-run=client -o yaml`




## Multi Container POD 생성하기
### 제공정보
- 작업할 클러스터 : hk8s
- pod name : lab004
- container image list : nginx, redis, memcached
### 풀이
1. context 변경 `kubectl config use-context hk8s`
1. 이미지 1개를 참조하는 pod yaml 샘플 생성   
`kubectl run lab004 --image=nginx --dry-run=client -o yaml > multi_pod.yaml`
2. vi 를 통해 yaml 의 containers 에 redis, memcached 이미지 추가

## Side Car Container POD 생성하기
### 제공정보
- sidecar name: sidecar
- image: busybox
- targetPod: eshop-cart-app
- command: /bin/sh -c 'tail -n+1 -F /var/log/eshop-app.log'
- volume: /var/log
- log file: cart-app.log
### 풀이
1. running 중인 pod 에서 yaml 추출 `kubectl get pod eshop-cart-app -o yaml > sidecar.yaml`
2. yaml 에 sidecar container 추가
    ```shell
   # 아래 내용을 containers 하위에 추가
   - name: sidecar
     image: busybox
     args: [/bin/sh, -c, 'tail -n+1 -F /var/log/eshop-app.log']
     volumeMounts:
     - name: varlog
       mountPath: /var/log
   ```
3. pod 재배포 `kubectl replace --force -f sidecar.yaml`

## POD scale out
### 제공정보
- cluster: hk8s
- pod 명 : eshop-order
- namespace: devops
- deployment: eshop-order
### 풀이
1. 작업 클러스터 변경 `kubectl config use-context k8s`
2. Scale out 작업 수행 `kubectl scale deployment eshop-order --replicas=5 -n devops`

## Deployment 생성 / scale out
### 제공정보
- cluster: k8s
- name : webserver
- replicas: 2
- label: app_env_stage=dev
- container name: webserver
- container image: nginx:1.14
- pod 2개 짜리 생성 이후 3개로 scale out
### 풀이
1. `kubectl config use-context k8s`
2. `kubectl create deployment webserver --image nginx:1.14 --replicas=2 --dry-run=client -o yaml > dep.yaml`
3. yaml 에서 label / matches label 수정 , 다른 값 정상여부 확인
3. `kubectl scale deployment webserver --replicas=3`

## Rolling update & roll back
### 제공정보
- cluster: k8s
- name : nginx-app
- image: nginx:1.11.10-alpine
- replicas: 3
- deployment 생성 후 nginx 버전 변경 1.11.13-alpine
### 풀이
```shell
kubectl config use-context k8s
kubectl create deployment nginx-app --image nginx:1.11.10-alpine --replicas=3 --dry-run=client -o yaml > dep.yaml
kubectl apply -f dep.yaml
kubectl set image deployment nginx-app nginx=nginx:1.11.13-alpine --record
kubectl rollout history deployment nginx-app
kubectl rollout undo deployment nginx-app
kubectl rollout history deployment nginx-app
```

## Node Selector
POD 를 특정 Node 에 띄우는 케이스
Node 에 달린 label 을 selector 에 입력해준다.
### 제공정보
- cluster: k8s
- pod name: eshop-store
- image: nginx
- node selector: disktype=ssd
### 풀이
```shell
kubectl config use-contect k8s
# node 의 label 중 disktype 을 컬럼으로 보여준다.
kubectl get nodes -L disktype
kubectl run eshop-store --image nginx --dry-run=client -o yaml > pod.yaml
# containers 와 동일한 level 에 node selector 추가
nodeSelector:
  disktype=ssd 
kubectl apply -f pod.yaml
```

## Node 관리
### 제공정보
- cluster: k8s-worker1
- node: k8s-worker1
- k8s-worker1 as unavailable and reschedule all the pods running on it.
### 풀이
kubectl config use-context k8s-worker1
kubectl drain k8s-worker1 --ignore-daemonset --force

## Node 정보 수집
### 제공정보
ready 상태 노드 숫자를 특정 파일에 저장하기 (no schedule 상태인것은 제외)
### 풀이
kubectl get nodes -o wide
kubectl describe node k8s-node1 | grep -i noSchedule
kubectl describe node k8s-node2 | grep -i noSchedule
kubectl describe node k8s-node2 | grep -i taints
echo "1" > /filePath

## Deploment & service expose
### 제공정보
- cluster: k8s
- 기동중인 front-end deployment 에 http 라는이름의 80/tcp port expose
- front-end-svc 생성 위에서 추가한 http port 를 사용하며, node port 유형임
### 풀이
> docs 에서 service 로 검색하면 deployment 와 svc 통합된 yaml 샘플 확인가능

kubectl config use-context k8s
kubectl get deployment front-end -o yaml > dep.yaml
```yaml
containers:
- name: http
  image: nginx
  ports:
  - containerPort: 80
    name: http
```
kubectl create service front-end-svc --dry-run=client -o yaml > svc.yaml

## cpu 사용량이 높은 pod 검색
### 제공정보
- pod label 이 name=overloaded-cpu 중 가장 많은  CPU 를 사용하느 POD 찾아 파일에 쓰기
### 풀이
kubectl top pod -l name=overloaded-cpu --sort-by cpu

## init 컨테이너를 포함한 pod 운영
init 컨테이너들은 순차적으로 실행되며, 이후에 main container 가 실행됨
### 제공정보
- web-pod 라는 init 컨테이너 추가해줘
- init container 는 /workdir/data.txt 라는 빈 파일을 생성해야함
- 기존 yaml 은 제공됨
### 풀이
1. edit yaml
```yaml
initContainers:
- name: web-pod
  image: busybox
  command: ['sh','-c','touch /workdir/data.txt']
  volumeMounts:
  - name: workdir
    mountpath: "/workdir"
```
2. kubectl apply -f test.yaml

## NodePort 서비스 생성
### 제공정보
- port 32767 리슨하는 nodeport 서비스 생성
- label: app:webui
### 풀이
1. app:webui 레이블을 사용하는 pod 의 port 확인  
   `kubectl get pod --selector app:webui --show-labels`  
   `kubectl describe pod {POD_NAME}`

2. Service yaml 생성
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
     app:webui 
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32767
```

3. 서비스 정상확인  
curl {workername}:32767

## config map 운영
### 제공정보
- namespace: ckad 에서 작업한다.  
- web-config 라는 config map 을 만든다.    
connection_string=localhost:80  
external_url=cncf.io
- web-pod 생성 image 는 nginx:1.19.8-alpine 이며 위에서 생성한 env 사용
### 풀이
- config map  생성
  ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
      name: web-config
      namespace: ckad
   data:
      connection_string: "localhost:80"
      external_url: "cncf.io"
   ```
- pod 생성
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: web-pod
     namespace: ckad
   spec:
     containers:
       - name: web-pod
         image: nginx:1.19.8-alpine
         envFrom:
         - configMapRef:
             name: web-config
   ```
- pod 에 env 명령 수행하여 정상 적용여부 확인  
`kubectl exec -n ckad web-pod -- env`

## Secret 운영
### 제공정보
- Secret 생성 
  - name: super-secret
  - data: password=secretpass
- pod 생성 1
  - name: pod-secrets-via-file
  - iamge: redis
  - mount path: /secrets
- pod 생성 1
  - name: pod-secrets-via-env
  - iamge: redis
  - export password as PASSWORD env
### 풀이
- Secret 생성  
`kubectl create secret generic super-secret --from-literal=password=secretpass`  
`kubectl get secret super-secret -o yaml`
- pod 1 - secret 파일 마운트
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-secrets-via-file
    spec:
      containers:
      - name: pod-secrets-via-file
        image: redis
        volumeMounts:
        - name: super-secret
          mountPath: "/secrets"
      volumes:
      - name: super-secret
        secret:
          secretName: super-secret
    ```
  `kubectl exec -it pod-secrets-via-file -- cat /secrets/password`
- pod 2 - 환경변수 사용
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: pod-secrets-via-env
    spec:
      containers:
      - name: pod-secrets-via-env
        image: redis
        env:
          - name: PASSWORD
            valueFrom:
              secretKeyRef:
                name: super-secret
                key: password
    ```
  `kubectl exec -it pod-secrets-via-env -- env`

## ingress
### 제공정보
- nginx pod 생성
  - namespace: ingress-nginx
  - image: nginx
  - label: app=nginx
- nginx service 생성
- ingress 구성
  - name : app-ingress
  - 30080/ 접속시 nginx 로 연결
  - 30080/app 접속시 appjs-service 로 연결
  - annotation 포함
     ```yaml
    annotations:
      kubernetes.io/ingress.class: nginx
     ```
### 풀이
- nginx pod 생성  
`kubectl run nginx --image=nginx -l=app=nginx --port=80 -n ingress-nginx`
- nginx service 생성  
`kubectl expose pod -n  ingress-nginx pod nginx --port 80 --target-port 80`
- ingress 구성
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: app-ingress
      namespace: ingress-nginx
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
        kubernetes.io/ingress.class: nginx
    spec:
      rules:
      - http:
          paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: appjs-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
    ```
  `curl k8s-worker1:30080`  
  `curl k8s-worker1:30080/app`

## Persistent Volume
### 제공정보
- Persistent Volume 생성
  - name: app-config
  - capacity: 1G
  - access mode: ReadWriteMany
  - storage class: az-c
  - volume type: hostPath , /srv/app-config
### 풀이
- pv 생성
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: app-config
      labels:
        type: local
    spec:
      storageClassName: az-c
      capacity:
        storage: 1Gi
      accessModes:
        - ReadWriteMany
      hostPath:
        path: "/srv/app-config"
    ```
- pv 확인  
`kubectl get pv`

## PVC 사용하는 POD 생성
### 제공정보
- PVC 생성
  - name: app-volume
  - storage class: app-hostpath-sc
  - capacity: 10Mi
- Pod 생성
  - name: web-server-pod
  - image: nginx
  - mountpath: /usr/share/nginx/html
- 생성되는 pod 는 ReadWriteMany access 가능해야함
### 풀이
- PVC 생성
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: app-volume
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 10Mi
      storageClassName: app-hostpath-sc
    ```
    `kubectl get pvc`
- POD 생성
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: web-server-pod
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
          - mountPath: "/usr/share/nginx/html"
            name: mypd
      volumes:
        - name: mypd
          persistentVolumeClaim:
            claimName: app-volume
    ```
    `kubectl describe pod web-server-pod`