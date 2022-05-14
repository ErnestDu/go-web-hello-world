## Task 0: Install a ubuntu 18.04 server 64-bit
Download the ISO and verify its checksum.

    wget https://releases.ubuntu.com/18.04/ubuntu-18.04.6-live-server-amd64.iso
    wget https://releases.ubuntu.com/18.04/SHA256SUMS
    sha256sum -c SHA256SUMS
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
Follow the steps in 

    sudo apt-get install -y curl openssh-server ca-certificates tzdata perl
    curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
    sudo EXTERNAL_URL="http://127.0.0.1" apt-get install gitlab-ce
    cat /etc/gitlab/initial_root_password

Open http://127.0.0.1:28080 in the host browser.
## Task 3: create a demo group/project in gitlab
Use the initial_root_password from Task 2 to login. Create an account named "demo", create a project named "go-web-hello-world"
Create a file named hello-world.go, copy the source code from https://gowebexamples.com/hello-world/ , change the port to 8080 because 80 was used by Gitlab.

    package main

    import (
        "fmt"
        "net/http"
    )

    func main() {
        http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
            fmt.Fprintf(w, "Hello, you've requested: %s\n", r.URL.Path)
        })

        http.ListenAndServe(":8080", nil)
    }

git add hello-world.go, then commit and push to Gitlab.

## Task 4: build the app and expose ($ go run) the service to 28081 port
install Go, https://go.dev/doc/install
    wget https://go.dev/dl/go1.18.2.linux-amd64.tar.gz
    rm -rf /usr/local/go && tar -C /usr/local -xzf go1.18.2.linux-amd64.tar.gz
    #export PATH=$PATH:/usr/local/go/bin
    go version

dockershim removed from Kubernetes
https://kubernetes.io/blog/2022/02/17/dockershim-faq/
install cri-dockerd
or a CRI implementation such as containerd CRI-O

