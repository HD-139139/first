# Kubernetes 
현재 여기는 쿠버네티스 설치의 과정을 보여주는 글

## Kuberadm

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curlsudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl


## Cluster

[k-control]
kubeadm init —control-plane-endpoint <IP주소> --pod-network-cidr <IP 대역> --apiserver-advertise-address <IP 사용 대역>
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id –g) $HOME/.kube/config
kubectl apply –f <add-on.yaml> --> calico network(https://docs.projectcalico.org/manifests/calico.yaml)

[k-node*]
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash> 
  

## Cluster Upgrade
apt update
apt-cache madison kubeadm

[k-control]
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.19.12-00 && \
apt-mark hold kubeadm
<1.21.x-00에서 x를 최신 패치 버전으로 바꾼다.>
--> 여기서 mark hold를 해서 자동 업그레이드를 막아줌(호환성을 위해)

or
  
apt-get update && \
apt-get install -y --allow-change-held-packages kubeadm=1.19.12-00
<# apt-get 버전 1.1부터 다음 방법을 사용할 수도 있다.>

sudo kubeadm upgrade node
apt-get update && \
apt-get install -y --allow-change-held-packages kubeadm=1.19.12-00 kubectl=1.19.12-00
sudo systemctl daemon-reload 
sudo systemctl restart kubelet
--> 1.19.12-00으로 업그레이드!

[k-node*]
apt-get update && \
apt-get install -y --allow-change-held-packages kubeadm=1.19.12-00
sudo kubeadm upgrade node
apt-get update && \
apt-get install -y --allow-change-held-packages kubeadm=1.19.12-00 kubectl=1.19.12-00
sudo systemctl daemon-reload 
sudo systemctl restart kubelet
--> 1.19.12-00으로 업그레이드!
  
## Addon

[ingress]
  
[k-control]
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.48.1/deploy/static/provider/aws/deploy.yaml
kubectl get ns
kubectl get pods -n ingress-nginx
kubectl edit svc -n ingress-nginx ingress-nginx-controller
--> spec.externalIPs: 에 각 node IP 주소 적어주기(워커노드의 목록을 나열)
kubectl get svc -n ingress-nginx
--> myapp-ing.yaml이란 이름으로 yaml파일 만들어주기
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: myapp-ing
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: myapp-svc-np
          servicePort: 80
```
kubectl create -f myapp-ing.yaml
