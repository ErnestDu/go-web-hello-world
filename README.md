## Task 0: Install a ubuntu 18.04 server 64-bit
Download the ISO and verify its checksum.
```    
wget https://releases.ubuntu.com/18.04/ubuntu-18.04.6-live-server-amd64.iso
wget https://releases.ubuntu.com/18.04/SHA256SUMS
sha256sum -c SHA256SUMS
```
Use VirtualBox to install a virtual machine. Assign 4 CPUs, 8GB memory and 1TB disk to the virtual machine. Configure Network Adapter 1 port forwarding as described in the task.
## Task 1: Update system
SSH login to the virtual machine.
```
# run on host
ssh 127.0.0.1 -p 22222
```
Use apt-get to update repository and upgrade system.
```
# run on vm
sudo apt-get update
sudo apt-get upgrade
# check kernel version
uname -r
```
## Task 2: Install Gitlab
Follow the steps in https://about.gitlab.com/install/#ubuntu?version=ce
```    
# Install and configure the necessary dependencies
sudo apt-get install -y curl openssh-server ca-certificates tzdata perl
# Add the GitLab package repository and install the package
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo EXTERNAL_URL="http://127.0.0.1" apt-get install gitlab-ce
# a password will be randomly generated
cat /etc/gitlab/initial_root_password
```
Open http://127.0.0.1:28080 in the host browser, a Gitlab login page will be shown.
## Task 3: create a demo group/project in gitlab
Use the initial_root_password from Task 2 to login. Create an account named "demo", create a project named "go-web-hello-world"
Create a file named hello-world.go, copy the source code from https://gowebexamples.com/hello-world/ , change the port to 8081 because 80 was used by Gitlab.
```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, you've requested: %s\n", r.URL.Path)
    })

    http.ListenAndServe(":8081", nil)
}
```
Push the code to Gitlab.
```
git add hello-world.go
git commit
git push
```
## Task 4: build the app and expose ($ go run) the service to 28081 port
Install Go with the instructions from https://go.dev/doc/install
```
# download package
wget https://go.dev/dl/go1.18.2.linux-amd64.tar.gz
# remove old go and install
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.18.2.linux-amd64.tar.gz
# export path
export PATH=$PATH:/usr/local/go/bin
# check install
go version
```
Run the go app after go is installed.
```
# run as a normal user
git clone http://127.0.0.1:28080/demo/go-web-hello-world.git
cd go-web-hello-world
go run hello-world.go &
# test, run on host
curl http://127.0.0.1:28081
```
We will get response text "Hello, you've requested: /".
## Task 5: install docker
Install docker on ubuntu, https://docs.docker.com/engine/install/ubuntu/
```    
# remove old versions
sudo apt-get remove docker docker-engine docker.io containerd runc
# Set up the repository
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# Set up the repository
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
# add user to docker group
sudo gpasswd -a ernest docker
# test
docker run hello-world
```
## Task 6: run the app in container
Pull latest golang docker image
```
docker pull golang
```
build a local docker image, create a Dockerfile with the following contents
```    
FROM golang
COPY hello-world.go /
EXPOSE 8082
RUN go build /hello-world.go
CMD ["./hello-world"]    
```
build docker image 
```
docker build -f Dockerfile .
```
run the docker image in detached mode, forward port 8081 to 8082
```    
docker run -d -p 8082:8081 52af1cce71cb
```
Test if the go app in the docker container is running
```
# run on host machine
curl http://127.0.0.1:28082
```
## Task 7 push image to dockerhub
```
docker login
# build the image with tag
docker build -t ernest/go-web-hello-world:v0.1 -f Dockerfile .
# push to docker hub
docker push ernest/go-web-hello-world:v0.1
```
Image can be found in https://hub.docker.com/r/ernest/go-web-hello-world

## Task 8: document the procedure in a MarkDown file
MarkDown syntax https://daringfireball.net/projects/markdown/syntax
    
## Task 9: install a single node Kubernetes cluster using kubeadm
### Install cri-dockerd
dockershim removed from Kubernetes, use cri-dockerd instead.
```shell
git clone https://github.com/Mirantis/cri-dockerd.git
mkdir bin
cd src && go get && go build -o ../bin/cri-dockerd
mkdir -p /usr/local/bin
install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
cp -a packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-dockerd.service
systemctl daemon-reload
systemctl enable cri-dockerd.service
systemctl enable --now cri-dockerd.socket
```
### Install kubeadm
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
### Setup Kubernetes Cluster
```
kubeadm init --cri-socket unix:///var/run/cri-dockerd.sock --pod-network-cidr=10.244.0.0/16
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
### add worker role
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane- node-role.kubernetes.io/master-
kubectl label node ubuntu-demo node-role.kubernetes.io/worker=worker
```
### get node
```
kubectl get node
```

## Task 10: deploy the hello world container
https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/
create a deployment with 2 replicas running
kubectl apply -f hello-application.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  selector:
    matchLabels:
      app: demo-app
  replicas: 2
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: hello-world
          image: ernest/go-web-hello-world:v0.1
          ports:
            - containerPort: 8081
              protocol: TCP
```
create a service to expose 8081 to virual machine 31080 port
kubectl apply -f hello-application-service.yaml
```
kind: Service
apiVersion: v1
metadata:
  name: hello-world-service
spec:
  selector:
    app: demo-app
  ports:
  - protocol: TCP
    port: 8081
    targetPort: 8081
    nodePort: 31080
  type: NodePort
```

## Task 11: install kubernetes dashboard
https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
```
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 31081
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort
```

kubectl apply recommended.yaml
open https://127.0.0.1:31081/ in a host browser. Kubernetes Dashboard sign in page is shown.

## Task 12: generate token for dashboard login in task 11
https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md
### Creating a Service Account
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```
Save as admin-user.yaml then run ``kubectl apply -f admin-user.yaml``
### Creating a ClusterRoleBinding
```
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
```
Save as dashboard-adminuser.yaml then run ``kubectl apply -f dashboard-adminuser.yaml``
### Generate token for admin-user
```
kubectl create token admin-user -n kubernetes-dashboard
```
Token can be used to sign into kubernetes dashboard.
## Task 13: publish your work
