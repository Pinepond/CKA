## kubectl 명령어 document
https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands

## kubectl cheatsheet
https://kubernetes.io/docs/reference/kubectl/cheatsheet/

## dry-run 을 통한 yaml 생성
kubernetes yaml 파일을 손으로 하나씩 작성하는 방법 대신 간단한 dry-run 명령 과 -o 옵션을 통해
yaml 파일을 손쉽게 생성할 수 있다.

```shell
# POD
kubectl run nginx --image=nginx --dry-run=client -o yaml > {yaml file name}
# Deployment
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > {yaml file name}
kubectl create deployment --image=nginx nginx --replicas=4 --dry-run=client -o yaml > {yaml file name}
# Service
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml > {yaml file name}
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml > {yaml file name}

```

## kubectl 기본 namespace 변경
kubectl 을 사용하다보면 개별 namespace 의 정보를 얻기 위해서는 --namespace 나 -n
옵션을 통해 namespace 를 입력해야 정보를 가져올 수 있다.
계속해서 kubectl 에 기본으로 잡힌 namespace 와 다른 namespace 에서 작업해야 한다면
current-context 를 변경해 작업하는것이 수월하다.
기본적으로 config set-context 사용시에 현재 namespace 와 앞으로 사용할 namespace 를
입력하게 된다. 현재 namespace 를 입력하는 곳을 `$(kubectl config current-context)` (현재 namespace 를 출력하는 명령어)
로 대체하면 바로 다른 namespace 로 전활 할 수있다.
```shell
kubectl config set-context {현재 namespace} --namespace={앞으로 사용할 namespace}
kubectl config set-context $(kubectl config current-context) --namespace=dev
```
