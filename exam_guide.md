# 시험 문제 유형
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
- cluster: hk8s
- name : webserver
- pod count: 2
- label: app_env_stage=dev
- container name: webserver
- container image: nginx:1.14
- pod 2개 짜리 생성 이후 3개로 scale out
### 풀이
1. `kubectl config use-context hk8s`
2. `kubectl create deployment webserver --image nginx:1.14 --replicas=2 --dry-run=client -o yaml > dep.yaml`
3. yaml 에서 label / matches label 수정 , 다른 값 정상여부 확인
3. `kubectl scale deployment webserver --replicas=3`