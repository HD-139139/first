# Kubernetes 
현재 여기는 쿠버네티스 설치의 과정을 보여주는 글

## Kuberadm

*sudo apt-get update  
sudo apt-get install -y apt-transport-https ca-certificates curlsudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg  
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list  
sudo apt-get update  
sudo apt-get install -y kubelet kubeadm kubectl  
sudo apt-mark hold kubelet kubeadm kubectl*  
--> 쿠버네티스의 필수요소 kubelet, kubeadm, kubectl을 설치해주었음   
--> k-control, k-node* 모두 설치해주어야함    
--> 쿠버네티스를 설치하기 위한 도구를 설치하는 것  
--> kubeadm : 클러스터를 부트스랩하는 명령  
&nbsp;&nbsp;&nbsp;  kubelet : 클러스터의 모든 머신에서 실행되는 파드와 컨테이너 시작과 같은 작업을 수행하는 컴포넌트  
    kubectl : 클러스터와 통신하기 위한 커맨드 라인 유틸리티  
--> apt-mark hold => 자동 패키지 업그레이드를 막아주는 명령어(= 업데이트 블럭)  



## Cluster

[k-control]  
*kubeadm init —control-plane-endpoint <IP주소> --pod-network-cidr <IP 대역> --apiserver-advertise-address <IP 사용 대역>*  
--> -control-plane-endpoint : API서버  
--> --pod-network-cidr : 컨테이너를 파드  
--> --apiservevr-advertise-address : 다른노드에게 API서버 주소를 알려주는 것  

*mkdir -p $HOME/.kube  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
sudo chown $(id -u):$(id –g) $HOME/.kube/config*  
--> 자격증명파일  

*kubectl apply –f <add-on.yaml>* --> calico network(https://docs.projectcalico.org/manifests/calico.yaml)  
--> 컨테이너 네트워크 인터페이스(CNI) 설치 완료  


[k-node*]  
*kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>*   
 --> node를 클러스터에 추가   

## Cluster Upgrade  
--> 업그레이할 땐 순서가 가장 중요함  
--> kubeadm -> kubectl/kubelet  
  
*apt update*  
*apt-cache madison kubeadm*  
  
[k-control]  
*apt-mark unhold kubeadm && \  
apt-get update && apt-get install -y kubeadm=1.19.12-00 && \  
apt-mark hold kubeadm*  
<1.21.x-00에서 x를 최신 패치 버전으로 바꾼다.>  
--> 여기서 앞서 hold해둔 kubeadm을 잠시 풀어 작업이 진행될 수 있게 하고 다시 mark hold를 해서 자동 업그레이드를 막아줌(호환성을 위해)  
  
or  
    
*apt-get update && \  
apt-get install -y --allow-change-held-packages kubeadm=1.19.12-00*  
<# apt-get 버전 1.1부터 다음 방법을 사용할 수도 있다.>  
  
*sudo kubeadm upgrade node  
apt-get update && \  
apt-get install -y --allow-change-held-packages kubeadm=1.19.12-00 kubectl=1.19.12-00  
sudo systemctl daemon-reload   
sudo systemctl restart kubelet*  
--> 1.19.12-00으로 업그레이드!  
  
[k-node*]  
*apt-get update && \  
apt-get install -y --allow-change-held-packages kubeadm=1.19.12-00  
sudo kubeadm upgrade node  
apt-get update && \  
apt-get install -y --allow-change-held-packages kubeadm=1.19.12-00 kubectl=1.19.12-00  
sudo systemctl daemon-reload   
sudo systemctl restart kubelet*  
--> 1.19.12-00으로 업그레이드!  

마이너버전 업그레이드  
*sudo apt install kubeadm=1.18.19-00 kubectl=1.18.19-00 kubelet=1.18.19-00*  
--> 버전을 일부러 낮춰줄거임 왜냐면 버전 최신이면 안전성 위험에 조금은 취약하기 때문  
  
## Addon  
  
### ingress  
     
[k-control]  
*kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.48.1/deploy/static/provider/aws/deploy.yaml  
kubectl get ns  
kubectl get pods -n ingress-nginx  
kubectl edit svc -n ingress-nginx ingress-nginx-controller*  
--> spec.externalIPs: 에 각 node IP 주소 적어주기(워커노드의 목록을 나열)  
*kubectl get svc -n ingress-nginx*  
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
*kubectl create -f myapp-ing.yaml*  
  
### rook  

*Vagrantfile*
```
  config.vm.define "k-node1" do |config|
    config.vm.box = "ubuntu/focal64"
    config.vm.provider = "virtualbox" do |vb|
      vb.name = "k-node1"
      vb.cpus = 2
      vb.memory = 3000
      unless File.exis?('./.disk/ceph1.vid')
        vb.customize ['createmedium', 'disk', '--filename', './.disk/ceph1.vdi', '--size', 10240]
      end
      vb.customaize ['storageattach', :id, '--storagectl', 'SCSI', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', './.disk/ceph1.vdi']
    end
    config.vm.hostname = "k-node1"
    config.vm.network "private_network", ip: "192.168.200.51"
  end
```  
--> 각 노드에 VM에 크기가 10G인 비어있는 디스크 할당  
vagrant halt 후 reload  

[rook - ceph cluster]  
*git clone --single-branch --branch v1.6.7 https://github.com/rook/rook.git  
cd rook/cluster/examples/kubernetes/ceph*  
--> Ceph 소스 다운로드  
  
*kubectl create -f crds.yaml -f common.yaml -f operator.yaml*  
--> crds, common, operator 설치    
  
*kubectl create -f cluster.yaml (3 worker)*  
또는  
*kubectl create -f cluster-test.yaml (1 worker)*  
--> cluster 생성    

*kubectl create -f csi/rbd/storageclass.yaml*    
--> 블럭 장치용 스토리지 클래스 생성  
  
*kubectl create -f filesystem.yaml (3 worker)*  
또는  
*kubectl create -f filesystem-test.yaml (1 worker)*   
--> 파일시스템 생성
  
*kubectl create -f csi/cephfs/storageclass.yaml*  
--> 파일 스토리지 생성  
--> 파일 시스템 생성 후 확인을 해야함!(a,b가 있어야함)  

  
[rook - ceph status(잘 됐는지 확인)]  
*kubectl create -f toolbox.yaml*  
--> toolbox 생성  
  
*kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph -s*
--> exec로 ceph를 실행  
--> health: HEALTH_WARN / HEALTH_OK 둘중 하나의 상태여야 함  
  

### MetalLB
--> MetalLB는 베어 메탈을 위한 로드 밸런서 구현, 쿠버네티스 표준 라우팅 프로토콜을 사용하는 클러스터  
--> 표준 네트워크 장비와 통합되는 네트워크 로드 밸런서 구현을 제공하여 이러한 불균형을 시정하는 것을 목표로 하여 베어메탈 클러스터의 외부 서비스도 가능한 한 "정상 작동"하도록 함.  
*kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml  
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml  
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"*  

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.200.240-192.168.200.250
``` 

*kubectl create -f metallb-config.yaml*  

  
### metrics-server

*git clone https://github.com/kubernetes-incubator/metrics-server.git  
cd metrics-server  
kubectl apply -f deploy/1.8+/*  
