# Kubernetes for Users Prerequisites

### 1. Install "Docker for Desktop" 18.06 on the laptop that you'll be bringing to class

Install from: https://www.docker.com/products/docker-desktop

This must be on your host operating system (Windows or MacOS) not on a virtual machine. Docker for Desktop will leverage your OS virtualization technology. For Windows users this means it requires that your system has HyperV (Windows 10 Pro). 

### 2. Enable Kubernetes

Kubernetes ships with Docker for Desktop on both Windows and Mac. Open the Docker for Desktop application window, navigate to the Kubernetes tab, and click the appropriate check box:

<p align="center">
<img alt="Enable Kubernetes in Docker for Desktop on Mac" src="https://github.com/gotopple/k8s-for-users-prerequisites/raw/master/images/enable-kube-d4m.png">
<img alt="Enable Kubernetes in Docker for Desktop on Windows" src="https://github.com/gotopple/k8s-for-users-prerequisites/raw/master/images/enable-kube-d4w.png">
</p>

##### 3. Verify installation and version

    kubectl version -o yaml

You should get output that looks like:

```
clientVersion:
  buildDate: 2018-05-21T09:17:39Z
  compiler: gc
  gitCommit: 2bba0127d85d5a46ab4b778548be28623b32d0b0
  gitTreeState: clean
  gitVersion: v1.10.3
  goVersion: go1.9.3
  major: "1"
  minor: "10"
  platform: darwin/amd64
serverVersion:
  buildDate: 2018-05-21T09:05:37Z
  compiler: gc
  gitCommit: 2bba0127d85d5a46ab4b778548be28623b32d0b0
  gitTreeState: clean
  gitVersion: v1.10.3
  goVersion: go1.9.3
  major: "1"
  minor: "10"
  platform: linux/amd64
```

### 4. Pre-pull several Docker images

You're going to be using several official sample programs. Download those Docker images now in order to speed things up during class. Run the following commands on your laptop:

    docker pull redis:alpine
    docker pull alpine:3.8
    docker pull dockersamples/visualizer:stable
    docker pull dockersamples/examplevotingapp_worker:latest
    docker pull dockersamples/examplevotingapp_result:before
    docker pull dockersamples/examplevotingapp_vote:before
    docker pull dockersamples/examplevotingapp_result:after
    docker pull dockersamples/examplevotingapp_vote:after
    docker pull postgres:9.4
    docker pull k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0

### 5. Email me

When you've done these please respond to the email and let me know that you're ready to go.

## Class rules:

1. Please arrive on time
2. Please no work or work email during class
3. Lids down during lecture

## Copyright

&copy; Topple 2018

