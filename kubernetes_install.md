apt-get install     apt-transport-https     ca-certificates     curl     gnupg-agent     software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
apt-key fingerprint 0EBFCD88
add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
apt update
apt upgrade
apt-get install docker-ce docker-ce-cli containerd.io
apt-cache madison docker-ce
apt-cache madison docker-ce

---------------------------------------------Kurulum master and Node-----------------------

swapoff -a

apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt update
apt-get install -y docker.io kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

-------------------------Master------------------------

kubeadm init --> swapoff

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.50:6443 --token hbjvv3.elozzhugbxtg0hfe \
    --discovery-token-ca-cert-hash sha256:34d7d4225bf69df2ec9a384c28dfeaa6047cc9dd17f9d0f00a76373bc049e8e3


root@huso:~# kubectl get nodes
NAME   STATUS     ROLES                  AGE   VERSION
huso   NotReady   control-plane,master   10m   v1.20.1

------------------------loglama gibi node hakkında bilgilieri veriyor ------------------------------------

root@huso:~# kubectl describe node huso

-------------------------network tercih kullanımı-----------------------------

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

root@huso:~# kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
huso   Ready    control-plane,master   14m   v1.20.1

kubectl get pods -A

--------------------------tab ile otomatik tamamlamak için------------------------

echo '' >> ~/.bashrc
echo 'source <(kubeadm completion bash)' >> ~/.bashrc
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc

-----------------------------------------------------------------------------------

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.41.2/deploy/static/provider/cloud/deploy.yaml

kubectl describe pods -n ingress-nginx ingress-nginx-controller-c4f944d4d-tj9ns

----------------------------masterı node olarak kullanabilmek için------------------------------

kubectl taint nodes --all node-role.kubernetes.io/master-

-----------------------------------------------------------------------------------

kubectl get services -A -w

-----------------------------metallb kurulum network----------------------------------------------

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"

----------------------------------------------elle çalıştırmak icin------------------------

root@huso:~# kubectl proxy --disable-filter=true --address='192.168.1.201'

-------------------------------kubernetes dashboard kullanıcı-----------------------------

root@huso:~# 

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF

root@huso:~# 

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

---------------------------------SSL------------------------------------

openssl req -x509 -new -newkey rsa:4096 -nodes -sha256 -days 3650  -subj "/CN=*.husoyargic.com" -addext subjectAltName=DNS:*.husoyargic.com -keyout tls.key -out tls.crt
kubectl create secret generic husoyargic --namespace ingress-nginx --from-file=tls.crt --from-file=tls.key
kubectl delete secrets -n ingress-nginx husoyargic
kubectl create secret tls husoyargic --namespace ingress-nginx --cert=tls.crt --key=tls.key

kubectl edit -n ingress-nginx deployments.apps ingress-nginx-controller
- --default-ssl-certificate=ingress-nginx/husoyargic  --->added

kubectl rollout restart deployment -n ingress-nginx ingress-nginx-controller

kubectl logs -f -n ingress-nginx ingress-nginx-controller-5cc6769bc4-sp44c

vim ingress_dns.yml
 annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
	nginx.ingress.kubernetes.io/secure-backends: "true"
	nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

root@huso:~/KUBERNETES# kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-7ngbn
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: f304f4ac-5cc5-4b6a-90a0-49cd7440d01c

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IndPMzVXdWFnUWUxM25id05nN2xDNWNJYlJ3Wl82enNBVm43Zks0NGdONFEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTduZ2JuIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJmMzA0ZjRhYy01Y2M1LTRiNmEtOTBhMC00OWNkNzQ0MGQwMWMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.moN8C-K-lujj0VemaDZ44_52MXwJ2V_DE9hawtBAydHDK58yfa46onehHpjdlSI-f7jpQ-sm9l_PGjQ-KTtoN7cv2B3cfAJ5E_LhJfGq8ASTNCsPKSg9EATcughZ3aNAVU-KBYFMIHu-HcGUdyQM0eZ99NZO6aZleZrk_zFVGJWXqHMZg5o7iqhPngJHHuTIf1jTsEhs3NSu4GknX5uElLztLgZDLuLtkj-50RvADMhEIExrmLGGCWnpujDI-qhy1Io3oZnWuj79iiXxjYJR61EeVLNi2tNK-6MM1MUsPDzvgpLdvbq-o6LnC6GhWPPamHiSzUEQOjP5Ols3CxdDsA
root@huso:~/KUBERNETES#
