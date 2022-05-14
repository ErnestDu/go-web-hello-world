## Task 0: Install a ubuntu 18.04 server 64-bit
Download the ISO and verify its checksum.
``wget https://releases.ubuntu.com/18.04/ubuntu-18.04.6-live-server-amd64.iso
wget https://releases.ubuntu.com/18.04/SHA256SUMS
sha256sum -c SHA256SUMS``
Use VirtualBox to install a virtual machine. Assign 4 CPUs, 8GB memory and 1TB disk to the virtual machine. Configure Network Adapter 1 port forwarding as described in the task.
## Task 1: Update system
    # host
    ssh 127.0.0.1 -p 22222
    # vm
    sudo apt-get update
    sudo apt-get upgrade
    # check kernel version
    uname -r
## Task 2: Install Gitlab
Follow the steps in https://about.gitlab.com/install/#ubuntu?version=ce
    
    # Install and configure the necessary dependencies
    sudo apt-get install -y curl openssh-server ca-certificates tzdata perl
    # Add the GitLab package repository and install the package
    curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
    sudo EXTERNAL_URL="http://127.0.0.1" apt-get install gitlab-ce
    # a password will be randomly generated
    cat /etc/gitlab/initial_root_password

Open http://127.0.0.1:28080 in the host browser, a Gitlab login page will be shown.
## Task 3: create a demo group/project in gitlab
Use the initial_root_password from Task 2 to login. Create an account named "demo", create a project named "go-web-hello-world"
Create a file named hello-world.go, copy the source code from https://gowebexamples.com/hello-world/ , change the port to 8081 because 80 was used by Gitlab.

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

git add hello-world.go, then commit and push to Gitlab.

## Task 4: build the app and expose ($ go run) the service to 28081 port
install Go, https://go.dev/doc/install
    wget https://go.dev/dl/go1.18.2.linux-amd64.tar.gz
    rm -rf /usr/local/go && tar -C /usr/local -xzf go1.18.2.linux-amd64.tar.gz
    export PATH=$PATH:/usr/local/go/bin
    go version
    # run as a normal user
    git clone http://127.0.0.1:28080/demo/go-web-hello-world.git
    cd go-web-hello-world
    go run hello-world.go &
    # run on host
    curl http://127.0.0.1:28081
## Task 5: install docker
https://docs.docker.com/engine/install/ubuntu/
    
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

## Task 6: run the app in container
    
pull latest golang docker image

    docker pull golang

build a local docker image, create a Dockerfile with the following contents
    
    FROM golang
    COPY hello-world.go /
    EXPOSE 8082
    RUN go build /hello-world.go
    CMD ["./hello-world"]    

build docker image 
    docker build -f Dockerfile .

run the docker image in detached mode, forward port 8081 to 8082
    
    docker run -d -p 8082:8081 52af1cce71cb
    # run on host machine
    curl http://127.0.0.1:28082

## Task 7 push image to dockerhub
    
    docker login
    # build the image with tag
    docker build -t ernest/go-web-hello-world:v0.1 -f Dockerfile .
    # push to docker hub
    docker push ernest/go-web-hello-world:v0.1

Image can be found in https://hub.docker.com/r/ernest/go-web-hello-world

## Task 8: document the procedure in a MarkDown file
    markdown syntax https://daringfireball.net/projects/markdown/syntax
    
## Task 9: install a single node Kubernetes cluster using kubeadm

    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
    sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

install cri-dockerd
To build this code (in a POSIX environment):
```shell
mkdir bin
cd src && go get && go build -o ../bin/cri-dockerd
```

To install, on a Linux system that uses systemd, and already has Docker Engine installed
```shell
# Run these commands as root
mkdir -p /usr/local/bin
install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
cp -a packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-dockerd.service
systemctl daemon-reload
systemctl enable cri-dockerd.service
systemctl enable --now cri-dockerd.socket
```
    
    kubeadm init --pod-network-cidr=10.244.0.0/16 
    kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
dockershim removed from Kubernetes
https://kubernetes.io/blog/2022/02/17/dockershim-faq/
install cri-dockerd
or a CRI implementation such as containerd CRI-O

    
