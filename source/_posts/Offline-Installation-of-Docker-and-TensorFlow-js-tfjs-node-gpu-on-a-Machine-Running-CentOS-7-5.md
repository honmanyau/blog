---
title: >-
  Offline Installation of Docker and TensorFlow.js (tfjs-node-gpu) on a Machine
  Running CentOS 7.5
tags:
  - tensorflow.js
  - tjfs
  - docker
  - centos
  - offline
date: 2019-03-22 04:36:52
---



## Table of Contents

* [Backstory](#Backstory)
* [Introduction](#Introduction)
* [Installing Docker](#Installing-Docker)
    * [Determining the Appropriate Version of Docker to Install](#Determining-the-Appropriate-Version-of-Docker-to-Install)
    * [Downloading Dependencies](#Downloading-Dependencies)
    * [Installing Docker on Both Machines](#Installing-Docker-on-Both-Machines)
    * [Testing the Docker Installations](#Testing-the-Docker-Installations)
* [Installing `nvidia-docker2`](#Installing-nvidia-docker2)
* [Creating a Node.js Docker Image](#Creating-a-Node-js-Docker-Image)
* [Downloading Node.js Dependencies](#Downloading-Node-js-Dependencies)
* [Modifying the `tfjs-node-gpu` Module for Offline Installation](#Modifying-the-tfjs-node-gpu-Module-for-Offline-Installation)
* [Setting up tfjs-node-gpu on the Offline Machine](#Setting-up-tfjs-node-gpu-on-the-Offline-Machine)


## Backstory

I procured a GTX 1070 eGPU off eBay at a very affordable price a few months ago because the intial work for some machine learning experiments were going well and I wanted to scale up. It was used in an unstable setup (Thunderbolt 2 Macbook Pro with dual-booted Ubuntu) but it was a risk that I was willing to take; in fact, I'm glad that I took the that risk because it got a large amount of work done and laid the foundation for something that is very likely publishable.

The card finally gave up after training neural networks for a couple of thousands of hours and, in order to continue the work, I then had the joy of trying to get the same setup running on a machine that a friend got me access to. As I found the process wasn't entirely straightforward, I thought I would document it in case it comes in handy to myself (again) or someone else!


## Introduction

The offline machine in question runs CentOS 7.5.1804, and has two Titan Xp running on driver version 390.77 but no CUDA and cuDNN drivers installed. I wanted to avoid rebooting the machine since:

* I'm not the only one using the machine.
* Physical access to the machine meant a plane ticket.
* Access to the machine was provided as a courtesy, I wanted to avoid hassling sysadmin as much as possible since it's not their job to baby-sit me.

Docker seemed like the perfect solution at the time, which worked out reasonably well in the end. The instructions below assume that the offline machine has a version of Nvidia driver that is comptaible with whatever TensorFlow distribution you are going to use already installed; otherwise, and if I'm not mistaken, rebooting is inevitable.


## Installing Docker

In the ideal case where updating the operating system (OS) as well as Nvidia drivers can be easily performed, you can likey simply install the latest version of Docker. However, **if updating those and/or rebooting is not an option**, then you need to determine a version of Docker whose dependencies won't require rebooting. The instructions below assume that updating the OS is NOT an option and that you do **not** wish to reboot the offline machine.

### Determining the Appropriate Version of Docker to Install

1. **\[Machine connected to the Internet\]** Set up the **stable** `docker-ce` repository with `yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`. [^1]
2. **\[Machine connected to the Internet\]** List all available `docker-ce` packages: [^2] [^3]
    ```shell
    [root@localhost ~]# yum list docker-ce --showduplicates | sort -r
    docker-ce.x86_64            3:18.09.3-3.el7                     docker-ce-stable
    docker-ce.x86_64            3:18.09.2-3.el7                     docker-ce-stable
    docker-ce.x86_64            3:18.09.1-3.el7                     docker-ce-stable
    docker-ce.x86_64            3:18.09.0-3.el7                     docker-ce-stable
    docker-ce.x86_64            18.06.3.ce-3.el7                    docker-ce-stable
    docker-ce.x86_64            18.06.2.ce-3.el7                    docker-ce-stable
    docker-ce.x86_64            18.06.1.ce-3.el7                    docker-ce-stable
    docker-ce.x86_64            18.06.0.ce-3.el7                    docker-ce-stable
    docker-ce.x86_64            18.03.1.ce-1.el7.centos             docker-ce-stable
    docker-ce.x86_64            18.03.0.ce-1.el7.centos             docker-ce-stable
    docker-ce.x86_64            17.12.1.ce-1.el7.centos             docker-ce-stable
    docker-ce.x86_64            17.12.0.ce-1.el7.centos             docker-ce-stable
    docker-ce.x86_64            17.09.1.ce-1.el7.centos             docker-ce-stable
    docker-ce.x86_64            17.09.0.ce-1.el7.centos             docker-ce-stable
    docker-ce.x86_64            17.06.2.ce-1.el7.centos             docker-ce-stable
    docker-ce.x86_64            17.06.1.ce-1.el7.centos             docker-ce-stable
    docker-ce.x86_64            17.06.0.ce-1.el7.centos             docker-ce-stable
    docker-ce.x86_64            17.03.3.ce-1.el7                    docker-ce-stable
    docker-ce.x86_64            17.03.2.ce-1.el7.centos             docker-ce-stable
    docker-ce.x86_64            17.03.1.ce-1.el7.centos             docker-ce-stable
    docker-ce.x86_64            17.03.0.ce-1.el7.centos             docker-ce-stable
    ```
3. **\[Machine connected to the Internet\]** Use the command `repoquery -R docker-ce-[PACKAGE NAME]` to list all dependencies for a given `docker-ce` package, [^2] [^3] for example:
    ```shell
    [root@localhost ~]# repoquery -R docker-ce-18.03.1.ce-1.el7.centos
    /bin/sh
    container-selinux >= 2.9
    device-mapper-libs >= 1.02.90-1
    iptables
    libc.so.6(GLIBC_2.17)(64bit)
    libcgroup
    libdevmapper.so.1.02()(64bit)
    libdevmapper.so.1.02(Base)(64bit)
    libdevmapper.so.1.02(DM_1_02_97)(64bit)
    libdl.so.2()(64bit)
    libdl.so.2(GLIBC_2.2.5)(64bit)
    libltdl.so.7()(64bit)
    libpthread.so.0()(64bit)
    libpthread.so.0(GLIBC_2.2.5)(64bit)
    libpthread.so.0(GLIBC_2.3.2)(64bit)
    libseccomp >= 2.3
    libseccomp.so.2()(64bit)
    libsystemd.so.0()(64bit)
    libsystemd.so.0(LIBSYSTEMD_209)(64bit)
    pigz
    rtld(GNU_HASH)
    systemd-units
    tar
    xz
    ```
4. **\[Offline machine\]** Check whether or not the machine has the required dependencies with `yum info [PACKAGE NAME]` (an incredibly manual process), for example:
    ```shell
    [root@localhost ~]# yum info container-selinux

    ...

    Available Packages
    Name        : container-selinux
    Arch        : noarch
    Epoch       : 2
    Version     : 2.74
    Release     : 1.el7
    Size        : 38 k
    Repo        : extras/7/x86_64
    Summary     : SELinux policies for container runtimes
    URL         : https://github.com/projectatomic/container-selinux
    License     : GPLv2
    Description : SELinux policy modules for use with container runtimes.
    ```
5. An alternative to **Step 4** is to download a `docker-ce` package with `yumdownloader [PACKAGE NAME]` on the machine connected to the Internet, [^4] transfer it to the offline machine, [^5] and attempt to install the package with `rpm -ivh` [^6] to determine what dependencies are required and whether or not a reboot will be required. **This method is preferred if the the two machines are running are running on different OS version, or that you expect the dependencies installed are not the same.**.

### Downloading Dependencies

1. **\[Machine connected to the Internet\]** Use `yum provides [DEPENDENCY NAME]` to find out which package provides a required dependency, if multiple packages are listed, pay attention to the versions numbers and processor architectures. For example: [^2]
    ```shell
    [root@localhost ~]# yum provides pigz             
   
    ...

    pigz-2.3.3-1.el7.centos.x86_64 : Parallel implementation of gzip
    Repo        : extras
    ```
2. **\[Machine connected to the Internet\]** Download `docker-ce` and all dependencies with `yumdownloader [PACKAGE NAME]` to a new directory as in the example shown below. In the case that you simply want to download a particularly version of docker with all its dependencies, use the `--resolve` flag when downloading the `docker-ce` package. [^2] [^3]
    ```shell
    [root@localhost ~]# mkdir docker
    [root@localhost ~]# cd docker
    [root@localhost:~/docker]# yumdownloader docker-ce-17.12.0.ce-1.el7.centos
    [root@localhost:~/docker]# yumdownloader pigz-2.3.3-1.el7.centos.x86_64

    ...
    ```
3. **\[Machine connected to the Internet\]** Archive the directory containing the dependencies, [^8] for example:
    ```shell
    [root@localhost:~/docker]# cd ..
    [root@localhost ~]# tar -czvf docker.tar.gz docker
    ```
4. Transfer the archive to the offline machine. [^5]

In my case, `docker-ce` version 17.03.2 doesn't require installation of any dependencies that would cause a reboot. [^7]

### Installing Docker on Both Machines

The instuctions below assume that the version of `docker-ce` downloaded is comptabile with both the offline machine and the one connected to the Internet. You may install different versions on each of the machine but keep that in mind when troubleshooting any issue that may come up.

1. **\[Machine connected to the Internet\]** Install Docker according the instructions in Docker Documentation [^1] or follow the steps below if you want a "practice run" of the offline installation.
2. **\[Offline machine\]** If you have created an archive of `docker-ce` and the required dependencies earlier, unarchive it and install the packages with `rpm -ivh *.rpm`; use the `--replacepkgs` and `--replacefiles` where appropriate: [^2] [^3]
    ```shell
    [root@localhost ~]# tar -zxvf docker.tar.gz
    [root@localhost ~]# cd docker
    [root@localhost:~/docker]# rpm -ivh *.rpm

    ...
    ```
3. Check that Docker is installed: [^9]
    ```shell
    [root@localhost:~/docker]# docker -v
    Docker version 17.03.2-ce, build f5ec1e2
    ```
4. Start Docker: [^1] [^2] [^3]
    ```shell
    [root@localhost:~/docker]# systemctl start docker
    ```
5. If you wish to have Docker start on boot: [^2] [^3]
    ```shell
    [root@localhost:~/docker]# systemctl enable docker
    ```

### Testing the Docker Installations

The Docker installation on the machine with an Internet connect can be tested simply with `docker run hello-world`. [^1] As the offline machine doesn't have a connection to pull any images from Docker Hub, images have to be first downloaded on the machine connected to the Internet, then transferred to the offline machine.

1. **\[Machine connected to the Internet\]** Download the `hello-world` image with `docker pull hello-world`.
2. **\[Machine connected to the Internet\]** Export the image:
    ```shell
    [root@localhost ~]# docker save -o hello-world.docker hello-world
    [root@localhost ~]# ls
    hello-world.docker
    ```
3. Transfer the Docker image to the offline machine. [^5]
4. **\[Offline machine\]** Import the image into Docker:
    ```shell
    [root@localhost ~]# ls
    hello-world.docker
    [root@localhost ~]# docker load < hello-world.docker
    [root@localhost ~]# docker run hello-world

    Hello from Docker!
    This message shows that your installation appears to be working correctly.

    To generate this message, Docker took the following steps:

    ...
    ```


## Installing `nvidia-docker2`

1. **\[Machine connected to the Internet\]** Add the repositories for `nvidia-docker2` as described in the `nvidia-docker2` documentation (remove the `>` sign if you are going to copy the commands below): [^10]
    ```shell
    [root@localhost ~]# distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
    > curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo \
    > | tee /etc/yum.repos.d/nvidia-docker.repo
    ```
2. **\[Machine connected to the Internet\]** Determine the name of the package for `nvidia-docker2` with `yum provides`, pay attention to the docker version appened to each of the `nvidia-docker2` versions:
    ```shell
    [root@localhost ~]# yum provides nvidia-docker2

    ...

    nvidia-docker2-2.0.0-1.docker1.12.6.noarch : nvidia-docker CLI wrapper
    Repo        : nvidia-docker



    nvidia-docker2-2.0.0-1.docker17.03.2.ce.noarch : nvidia-docker CLI wrapper
    Repo        : nvidia-docker

    ...
    ```
3. **\[Machine connected to the Internet\]** Download the `nvidia-docker2` package that matches the version of Docker **installed on the offline machine** with `yumdownloader`, for example:
    ```shell
    [root@localhost ~]# mkdir nvidia-docker
    [root@localhost ~]# cd nvidia-docker
    [root@localhost:~/nvidia-docker]# yumdownloader nvidia-docker2-2.0.0-1.docker17.03.2.ce.noarch

    ...

    [root@localhost:~/nvidia-docker]# ls
    nvidia-docker2-2.0.0-1.docker17.03.2.ce.noarch.rpm
    ```
4. **\[Machine connected to the Internet\]** Download the appropriate `nvidia/cuda` Docker image (as required by the version of TensorFlow that you are going to use) with the `devel` suffix. The example below shows the commands for CUDA 9.0 and cuDNN 7 but, if I'm not mistaken, CUDA 10.0 is required if you are going to use the latest version of `tfjs-node-gpu` (1.0.1 at the time of writing, which uses version 1.13 of the TensorFlow library):
    ```shell
    [root@localhost ~]# docker pull nvidia/cuda:9.0-cudnn7-devel
    [root@localhost ~]# docker save -o nvidia-cuda.docker nvidia/cuda:9.0-cudnn7-devel
    ```
4. Transfer both the `nvidia-docker2` package and the `nvidia/cuda` image to the offline machine. [^5]
5. **\[Offline machine\]** Install the package and "reload the Docker daemon configuration": [^10]
    ```shell
    [root@localhost ~]# rpm -ivh nvidia-docker2-2.0.0-1.docker17.03.2.ce.noarch.rpm
    [root@localhost ~]# pkill -SIGHUP dockerd
    ```
6. **\[Offline machine\]** Import the `nvidia/cuda` image:
    ```shell
    [root@localhost ~]# docker load < nvidia-cuda.docker
    ```
7. **\[Offline machine\]** Test the `nvidia-docker2` installation: [^11]
    ```shell
    [root@localhost ~]# docker run --runtime=nvidia --rm nvidia/cuda:9.0-cudnn7-devel nvidia-smi

    ...

    +-----------------------------------------------------------------------------+
    | NVIDIA-SMI 390.77                 Driver Version: 390.77                    |
    |-------------------------------+----------------------+----------------------+
    | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
    |===============================+======================+======================|
    |   0  TITAN Xp            Off  | 00000000:06:00.0 Off |                  N/A |
    | 24%   43C    P2    59W / 250W |  11635MiB / 12196MiB |      2%      Default |
    +-------------------------------+----------------------+----------------------+
    |   1  TITAN Xp            Off  | 00000000:82:00.0 Off |                  N/A |
    | 28%   48C    P2    63W / 250W |  11599MiB / 12196MiB |      0%      Default |
    +-------------------------------+----------------------+----------------------+
                                                                                  
    ...
    ```

It is worth noting that if you don't see any error messages at this point, you should have a working `nvidia-docker` setup that you can use for anything that depends on `nvidia-docker`. The instructions below are for those working specifically with TensorFlow.js under a Node.js environment (tfjs-node-gpu).


## Creating a Node.js Docker Image

There are various ways that one can go about setting up a TensorFlow Docker image. In my case I decided to build everything from an Ubuntu 18.04 image because that was the set up I used with my GTX 1070 eGPU.

1. **\[Machine connected to the Internet\]** Create a file called `Dockerfile`:
    ```shell
    [root@localhost ~]# touch Dockerfile
    ```
2. **\[Machine connected to the Internet\]** Add the content below to `Dockerfile`:
    ```shell
    # Use the official Ubuntu 18.04 image
    FROM ubuntu:18.04

    RUN apt-get update

    # Edit the following lines to include other packages that you work with
    RUN apt-get install -y nodejs npm curl vim less

    # Download n Node.js version manager and yarn
    RUN npm i -g n yarn

    # Install the latest version of Node.js; edit this line if a different
    # version is required, for example: RUN n 10.15.3.
    RUN n latest

    # Clean up
    RUN apt-get autoremove
    RUN apt-get clean

    # Use /app as default the work directory.
    WORKDIR /app
    ```
3. **\[Machine connected to the Internet\]** Build an image using the new `Dockerfile` (don't miss the dot at the end!): [^12]
    ```shell
    [root@localhost ~]# docker build -t tfjs-node-gpu .
    ```
4. **\[Machine connected to the Internet\]** Test the new image: 
    ```shell
    [root@localhost ~]# docker run -it tfjs-node-gpu
    root@ec5d8c89e41e:/app# node -v
    v11.11.0
    root@ec5d8c89e41e:/app# node -e "console.log([ 'eins', 'zwei', 'drei' ].join('\n'))"
    eins
    zwei
    drei
    root@ec5d8c89e41e:/app# exit
    ```
4. **\[Machine connected to the Internet\]** Export the image built in **Step 3**:
    ```shell
    [root@localhost ~]# docker save -o tfjs-node-gpu.docker tfjs-node-gpu
    ```
5. Transfer the image to the offline machine. [^5]
6. **\[Offline machine\]** Import the image into Docker:
    ```shell
    [root@localhost ~]# docker load < tfjs-node-gpu.docker
    ```
7. **\[Offline machine\]** Test the image as described in **Step 4** and check that `nvidia-smi` produces the expected output without errors.


## Downloading Node.js Dependencies

The steps below assume that you are starting a new project, but can be easily adapted for an existing project.

1. **\[Machine connected to the Internet\]** Start a container using the `tfjs-node-gpu` image created as described in the previous section, [^13] create a new directory and, inside the new directory, `package.json`:
    ```shell
    [root@localhost ~]# mkdir machine-learning
    [root@localhost ~/machine-learning]# cd machine-learning
    [root@localhost ~/machine-learning]# docker run -it -v $(pwd):/app --rm tfjs-node-gpu
    root@b0bc89eb30da:/app# npm init

    ...
    ```
2. **\[Machine connected to the Internet\]** Set up the offline mirror directory for `yarn`:
    ```shell
    root@b0bc89eb30da:/app# yarn config set yarn-offline-mirror ~/yarn-offline-mirror
    yarn config v1.13.0
    success Set "yarn-offline-mirror" to "/root/yarn-offline-mirror".
    Done in 0.04s.
    ```
3. **\[Machine connected to the Internet\]** Download the Node.js modules that you need for your project with `yarn`, for example:
    ```shell
    root@b0bc89eb30da:/app# yarn add @tensorflow/tfjs-node-gpu@^0.3.2 && \
    > yarn add -D typescript tslint ts-node @types/node
    ```
4. **\[Machine connected to the Internet\]** Move `~/.node-gyp` and `~/yarn-offline-mirror` to the working directory, and remove the `node_modules` directory. Note that the `/.node-gyp` directory is created when installing the `@tensorflow/tfjs-node-gpu` module and depends on the version of Node.js that is currently running; as such, **the version of Node.js used here must be the same as the version of Node.js shown in Step 4 of [Creating a Node.js Docker Image](#Creating-a-Node-js-Docker-Image)**.
    ```shell
    root@b0bc89eb30da:/app# mv ~/.node-gyp ~/yarn-offline-mirror . 
    root@b0bc89eb30da:/app# rm -rf node_modules
    root@b0bc89eb30da:/app# ls -a
    .  ..  .node-gyp  package.json  yarn-offline-mirror  yarn.lock
    ```

## Modifying the `tfjs-node-gpu` Module for Offline Installation

A TensorFlow libary is downloaded during installation of the `tfjs-node-gpu` module, which is obviously not possible on the offline machine; therefore, the library needs to be downloaded manually in advance and incoporated into `tfjs-node-gpu` archive.

1. **\[Machine connected to the Internet\]** Unpack the `tfjs-node-gpu` module and inspect `install.js`. Change the version number shown in the example below if you are working with a different version of `tfjs-node-gpu`.
    ```shell
    root@b0bc89eb30da:/app# ls -a
    .  ..  .node-gyp  package.json  yarn-offline-mirror  yarn.lock
    root@b0bc89eb30da:/app# cd yarn-offline-mirror
    root@b0bc89eb30da:/app/yarn-offline-mirror# tar -zxvf \@tensorflow-tfjs-node-gpu-0.3.2.tgz
    root@b0bc89eb30da:/app/yarn-offline-mirror# rm \@tensorflow-tfjs-node-gpu-0.3.2.tgz
    root@b0bc89eb30da:/app/yarn-offline-mirror# less package/scripts/install.js
    ```
2. **\[Machine connected to the Internet\]** Determine the URI of the TensorFlow libaray required. In this particular case, the URI is `BASE_URI + GPU_LINUX` (https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-gpu-linux-x86_64-1.12.0.tar.gz in the example below) as the offline machine runs on CentOS; adjust accordingly if you are setting things up on a different OS. In addition, please note that the **library version may be different if youare using a different version of `tfjs-node-gpu`**; for example, at the time of writing, `tfjs-node-gpu` 1.0.1 uses version 1.13 of the TensorFlow library.
    ```js
    const BASE_URI =
    'https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-';
    const CPU_DARWIN = 'cpu-darwin-x86_64-1.12.0.tar.gz';
    const CPU_LINUX = 'cpu-linux-x86_64-1.12.0.tar.gz';
    const GPU_LINUX = 'gpu-linux-x86_64-1.12.0.tar.gz';
    const CPU_WINDOWS = 'cpu-windows-x86_64-1.12.0.zip';
    const GPU_WINDOWS = 'gpu-windows-x86_64-1.12.0.zip';

    const TF_WIN_HEADERS_URI =
        'https://storage.googleapis.com/tf-builds/tensorflow-headers-1.12.zip';

    ...
    ```
3. **\[Machine connected to the Internet\]** Download the TensorFlow library required with the link determined in **Step 2** to the `package/deps` directory and unpack it:
    ```shell
    root@b0bc89eb30da:/app/yarn-offline-mirror# cd package/deps
    root@b0bc89eb30da:/app/yarn-offline-mirror/package/deps# curl --remote-name https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-gpu-linux-x86_64-1.12.0.tar.gz
    root@b0bc89eb30da:/app/yarn-offline-mirror/package/deps# tar -zxvf libtensorflow-gpu-linux-x86_64-1.12.0.tar.gz
    root@b0bc89eb30da:/app/yarn-offline-mirror/package/deps# rm libtensorflow-gpu-linux-x86_64-1.12.0.tar.gz
    ```
4. **\[Machine connected to the Internet\]** Modify the async function `run` in `install.js` so that the script doesn't attempt to download anything and simply calls the `build` async function as shown below. Again, this may be different for different version of `tfjs-node-gpu`, adapt accordingly.
    ```js
    async function run() {
      await build();
    // First check if deps library exists:
    // if (forceDownload !== 'download' && await exists(depsLibTensorFlowPath)) {
        // Library has already been downloaded, then compile and simlink:
        // await build();
    // } else {
        // Library has not been downloaded, download, then compile and symlink:
        // await cleanDeps();
        // await downloadLibtensorflow(build);
    // }
    }
    ```
5. **\[Machine connected to the Internet\]** Create a new `tfjs-node-gpu` archive, make sure that the correct version number is used:
    ```shell
    root@b0bc89eb30da:/app/yarn-offline-mirror/package/deps# cd ../..
    root@b0bc89eb30da:/app/yarn-offline-mirror# tar -czvf \@tensorflow-tfjs-node-gpu-0.3.2.tgz package
    root@b0bc89eb30da:/app/yarn-offline-mirror# rm -rf package
    ```
6. **\[Machine connected to the Internet\]** Exit the docker instance and create an archive of the project folder:
    ```shell
    root@b0bc89eb30da:/app/yarn-offline-mirror# exit
    [root@localhost ~/machine-learning] cd ..
    [root@localhost ~] tar -czvf machine-learning.tar.gz machine-learning
    ```
7. Transfer the archive to the offline machine. [^5]


## Setting up `tfjs-node-gpu` on the Offline Machine

1. **\[Offline machine\]** Unarchive `machine-learning.tar.gz` created in the previous section and start a `tfjs-node-gpu` container:
    ```shell
    [root@localhost ~]# ls
    machine-learning.tar.gz
    [root@localhost ~]# tar -zxvf machine-learning.tar.gz
    [root@localhost ~]# rm machine-learning.tar.gz
    [root@localhost ~]# cd machine-learning
    [root@localhost ~]# docker run -it -v $(pwd):/app --runtime=nvidia --rm tfjs-node-gpu
    root@18a41cf60791:/app# ls -a
    .  ..  .node-gyp  package.json  yarn-offline-mirror  yarn.lock
    ```
2. **\[Offline machine\]** Prepare directories and start offline installation with `yarn`: [^15]
    ```shell
    root@18a41cf60791:/app# mv .node-gyp yarn-offline-mirror ~
    root@18a41cf60791:/app# yarn config set yarn-offline-mirror ~/yarn-offline-mirror
    root@18a41cf60791:/app# yarn --offline --update-checksums

    ...

    success Saved lockfile.
    Done in 13.67s.
    ```
3. **\[Offline machine\]** Check that no errors are thrown when the `@tensorflow/tfjs-node-gpu` module is `require`d:
    ```shell
    root@18a41cf60791:/app# node -e "require('@tensorflow/tfjs-node-gpu');"
    ```
4. You're all set to start working with `tfjs-node-gpu` on the offline machine!



[^1]: [Docker Documentation | Get Docker CE for CentOS](https://docs.docker.com/install/linux/docker-ce/centos)
[^2]: [CODE FARM | Setup Docker Engine on Centos Offline](https://codefarm.me/2018/03/30/setup-docker-engine-on-centos-offline)
[^3]: [InsidePacket | Install Docker Offline on Centos7](https://davidwzhang.com/2018/11/17/install-docker-offline-on-centos7)
[^4]: For example: `yumdownloader docker-ce-18.03.1.ce-1.el7.centos`.
[^5]: With `scp` if the machine is in the same local network as the machine connceted to the Internet; otherwise, with your choice of portable storage medium.
[^6]: For example: `rpm -ivh docker-ce-18.03.1.ce-1.el7.centos.x86_64.rpm`.
[^7]: This is only an indication for what may work for you in case you are running CentOS 7.5.1804; as I didn't set up the offline from scratch, it is not clear whether or not some of the dependencies that require rebooting was already installed before I started.
[^8]: You may skip this if you are transferring using your favourite portable storage medium.
[^9]: The version number shown in the code snippet is only an example.
[^10]: [GitHub—NVIDIA/nvidia-docker > CentOS 7 (docker-ce), RHEL 7.4/7.5 (docker-ce), Amazon Linux 1/2](https://github.com/NVIDIA/nvidia-docker)
[^11]: The GPUs are in use in the example shown.
[^12]: It is worth noting that you may wish to specify the tag differently to include version information and/or author information, for example: `honmanyau/tfjs-node-gpu:1.0.0`.
[^13]: You may also use any Node.js, npm and yarn installation that you already have on your system.
[^14]: This step assumes that you started the container with the `-v $(pwd):/app` option, so that the working directory inside the docker instance is the `machine-learning` directory outside of the container.
[^15]: The `--update-checksums` flag is necessary because we have edited the `tfjs-node-gpu` archive ([yarn documentation](https://yarnpkg.com/lang/en/docs/cli/install/)).