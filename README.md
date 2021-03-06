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
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; kubelet : 클러스터의 모든 머신에서 실행되는 파드와 컨테이너 시작과 같은 작업을 수행하는 컴포넌트  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; kubectl : 클러스터와 통신하기 위한 커맨드 라인 유틸리티  
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
- 인그레스 : 쿠버네티스 내부 파드 노출시키기 = L7 프록시, 로드밸런서 역할     
- nginx를 젤 많이 활용  
   
[k-control]  
*kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.48.1/deploy/static/provider/aws/deploy.yaml  
kubectl get ns  
kubectl get pods -n ingress-nginx  
--> 설치 후 ingress-nignx-admission-patch, ingress-nignx-admission-create는 job으로 실행된거라서 status가 completed로 뜸  
  
<인그레스 컨트롤 수정>   
kubectl edit svc -n ingress-nginx ingress-nginx-controller*  
--> spec.externalIPs: 에 각 node IP 주소 적어주기(워커노드의 목록을 나열)
--> clusterIP, externalTrafficPolicy: Cluster도 확인  
*kubectl get svc -n ingress-nginx*  
--> 설치완료  

<인그레스는 두 가지 그룹이 존재>  
&nbsp;&nbsp;&nbsp;&nbsp;- 일반적으로 extensions는 잘 사용하지 않음  
&nbsp;&nbsp;&nbsp;&nbsp;- 인그레스라는 아이가 v1, beta1을 사용한지 3년정도 --> beta버전이 너무 오래있었음 --> 공식 버전은 v1으로 변경 --> 그래서 extensions는 곧 사라질거라서 잘 사용하지 않음  
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
--> rules: 라우팅 정책  
--> host: L7  
&nbsp;&nbsp;&nbsp;  - HTTPD기반이라서 IP로 접속하지 않음  
--> path: / -> 호스트 하위 경로 지정해준 것  
--> backend: host로 접속 -> 서비스이름:포트로 연결 -> 해당 파드로 연결  
 
 
+ 순서  
&nbsp;&nbsp;&nbsp;&nbsp;: 클라이언트 -> 인그레스(컨트롤러) -> 라우팅 규칙 -> 서비스(노드포트) -> 파드   
*kubectl create -f myapp-ing.yaml*  
  
### rook  
- K8s rook (CEPH 스토리지 구성)  
-  온프레미스환경의 K8s 에서 스토리지클래스를 사용하기위한 좋은 방법중 하나로 현재 rook라는 프로젝트가 있음. 지금부터 rook기반의 Ceph 스토리지 설치방법  

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
- MetalLB는 베어 메탈을 위한 로드 밸런서 구현, 쿠버네티스 표준 라우팅 프로토콜을 사용하는 클러스터  
- 표준 네트워크 장비와 통합되는 네트워크 로드 밸런서 구현을 제공하여 이러한 불균형을 시정하는 것을 목표로 하여 베어메탈 클러스터의 외부 서비스도 가능한 한 "정상 작동"하도록 함  
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
- 쿠버네티스에서는 기본적으로 사용할 수 있는 모니터링 도구로 힙스터(heapster)(주로 사용량확인)  
- 힙스터를 간소화한 버전
- 쿠버네티스에서 필요한 핵심 데이터들은 대부분 etcd에 저장되지만 메트릭 데이터들을 etcd에 저장하면 etcd의 부하가 너무 커지기 때문에 그렇게 하지 않고 메모리에 저장하도록 되어있음  
  
  
*git clone https://github.com/kubernetes-incubator/metrics-server.git  
cd metrics-server  
kubectl apply -f deploy/1.8+/*  
--> kubectl을 이용해서 적용하면, v1beta1.metrics.k8s.io 라는 apiservce가 생성되고, metrics-server 라는 디플로이먼트와 서비스가 생성  
