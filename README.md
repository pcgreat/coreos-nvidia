
# Container Linux (aka CoreOS) NVIDIA Driver [![Build Status](https://travis-ci.org/src-d/coreos-nvidia.svg?branch=master)](https://travis-ci.org/src-d/coreos-nvidia) (forked from src-d)

Forked from https://github.com/src-d/coreos-nvidia. That repo is actually great, but I did take me a while to realize what it is doing. So to helper new learners in this field, I would like to add a few notes here.

1. The basic idea is to install the Nvidia driver in `srcd/coreos-nvidia` docker, so that it can access to the GPU. Then, run it as a service, and run your app docker with `--volume-from` the `srcd/coreos-nvidia` docker, so that your app docker can access GPU, too.
2. The `srcd/coreos-nvidia` is responsible for installing the nvidia driver, and your app docker is responsible for installing cuda.
3. **Run with srcd's built docker:**

    90% of the chance this will solve your problem. I am running a AWS p2 instance (Nvidia K80 GPU), and I don't have any problem with that. The built docker image `srcd/coreos-nvidia` which is publicly available from docker hub is awesome. 
    
    3.1.    Write below in a bash script (start.sh) on your Coreos server, and then execute `bash start.sh` 
    ```
    source /etc/os-release
    
    # Set up service for nvidia-driver (srcd/coreos-nvidia) in coreos-nvidia.service
    sudo tee -a /etc/systemd/system/coreos-nvidia.service <<EOF
    [Unit]
    Description=NVIDIA driver
    After=docker.service
    Requires=docker.service
    
    [Service]
    TimeoutStartSec=20m
    EnvironmentFile=/etc/os-release
    ExecStartPre=-/usr/bin/docker rm nvidia-driver
    ExecStartPre=/usr/bin/docker run --rm --privileged --volume /:/rootfs/ srcd/coreos-nvidia:${VERSION}
    ExecStart=/usr/bin/docker run --rm --name nvidia-driver srcd/coreos-nvidia:${VERSION} sleep infinity
    ExecStop=/usr/bin/docker stop nvidia-driver
    ExecStop=-/sbin/rmmod nvidia_uvm nvidia
    
    [Install]
    WantedBy=multi-user.target  
    EOF
    
    sudo systemctl enable /etc/systemd/system/coreos-nvidia.service
    sudo systemctl start coreos-nvidia.service
    
    lsmod | grep -i nvidia
    docker ps | grep -i nvidia-driver
    ``` 
    3.2.    And then you can run tensorflow or any other of your app dockers via
    ```
    docker run --rm -it \
        --volumes-from nvidia-driver \
        --env PATH=$PATH:/opt/nvidia/bin/ \
        --env LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/nvidia/lib \
        $(for d in /dev/nvidia*; do echo -n "--device $d "; done) \
        gcr.io/tensorflow/tensorflow:latest-gpu \
            python -c "import tensorflow as tf;tf.Session(config=tf.ConfigProto(log_device_placement=True))"
    ```
4. **Build your custom coreos-nvidia image:**

    Again, most of the time you don't need this. Step 3 alone can solve all your problems. But if you do decide to build your own `coreos-nvidia` image, see some tips here:
    
    4.1.    You might need to upgrade your docker version on your coreos first. Some [new feature like "as BUILD"](https://docs.docker.com/engine/userguide/eng-image/multistage-build/) won't work for docker version below 1.17.05. The easiest way in coreos to update docker version is to update coreos version. To do that, just simply `sudo update_engine_client -update`. Reboot after that.
    
    4.2.    Build docker image. You need to pass your own argument. Here is an example: `docker build -t coreos-nvidia --build-arg COREOS_RELEASE_CHANNEL=stable --build-arg COREOS_VERSION=1576.5.0 --build-arg NVIDIA_DRIVER_VERSION=387.34 --build-arg KERNEL_VERSION=4.14.11 --build-arg KERNEL_TAG=4.14.11 .`
    
    - **COREOS_VERSION** can be found via `cat /etc/os-release`
    
    - **COREOS_RELEASE_CHANNEL** can be select as "stable", "alpha", "beta", but you need to make sure the file `https://${COREOS_RELEASE_CHANNEL}.release.core-os.net/amd64-usr/${COREOS_VERSION}/coreos_developer_container.bin.bz2` exists. For example, given your coreos version, sometimes only "stable" or only "beta" version exists.
    
    - **NVIDIA_DRIVER_VERSION** In theory you can find the most appropriate driver through http://www.nvidia.com/Download/index.aspx?lang=en-uk, but I find 387.34 works perfect. That's also the version `srcd` used to build `srcd/coreos-nvidia`
    
    - **KERNEL_VERSION** and **KERNEL_TAG** You can find it via `uname -r`
    
    4.3.    After successfully build the docker, you need to test the command `docker run --rm --privileged --volume /:/rootfs/ srcd/coreos-nvidia:${VERSION}`. 
    
    - Note that this command can only be executed once, because `insmod` can only insert the `nvidia.ko` module and `nvidia-uvm.ko` to linux kernel once. 
    
    - And that command cannot work without `--privileged`, because inserting module requires root privilege.
    
    - After that, you will find three files /dev/nvidia0, /dev/nvidia-uvm, /dev/nvidiactl on your CoreOS server

    4.4.    You can test nvidia-smi via 
    
    ```sh
    docker run --rm $(for d in /dev/nvidia*; do echo -n "--device $d "; done) \
        srcd/coreos-nvidia:${VERSION} nvidia-smi -L
    
    // Outputs:
    // GPU 0: Tesla K80 (UUID: GPU-d57ec7e8-ab97-8612-54ac-9d53a183f818)
    ```
    
    This command doesn't require `--privileged`, as previous command has already inserted `nvidia.ko` and `nvidia-uvm.ko` module
    
    4.5.    Try to understand the commands in `/etc/systemd/system/coreos-nvidia.service`, basically it does the same thing as step 4.4, which is to run the docker but keep it alive via "sleep infinity" and give it the name `nvidia-driver`. Change your `/etc/systemd/system/coreos-nvidia.service` accordingly to use your custom docker image.

5. Build your app docker:

    You can select cuda for your docker ```FROM nvidia/cuda:7.5-cudnn5-devel```. A sample app Dockerfile is here 
    ```
    FROM nvidia/cuda:7.5-cudnn5-devel
    
    WORKDIR /opt/app
    
    ADD requirements.txt /opt/app/
    
    RUN apt-get update && \
        apt-get install -y software-properties-common && \
        add-apt-repository ppa:jonathonf/python-3.6 && \
        apt-get update && \
        apt-get install --no-install-recommends -y python3.6 wget && \
        wget https://bootstrap.pypa.io/get-pip.py && \
        python3.6 get-pip.py && \
        ln -sf /usr/bin/python3.6 /usr/local/bin/python3 && \
        ln -sf /usr/local/bin/pip /usr/local/bin/pip3 && \
        pip3 install --upgrade pip setuptools && \
        apt-get install --no-install-recommends -y \
        python3.6-dev \
        build-essential \
        libopenblas-dev \
        liblapack-dev && \
        apt-get clean && \
        apt-get autoclean && \
        rm -rf /var/lib/apt/lists && \
        pip3 install theano==0.9.0
        
    ENV THEANO_FLAGS 'floatX=float32,device=gpu0'
    ```

6.  You should be able to run your docker via
    ``` sh
    docker run --rm -it     
        --volumes-from nvidia-driver     
        --env PATH=$PATH:/opt/nvidia/bin/:/usr/local/cuda/bin     
        --env LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/nvidia/lib:/usr/local/cuda/lib     
        $(for d in /dev/nvidia*; do echo -n "--device $d "; done) 
        YOUR_APP_DOCKER_NAME```
   And do `python3 -c 'import theano'` inside your docker image

7. If you are not sure about step 5 or 6, you can just run the `tensorflow` GPU enabled container and verifying the identified devices (like srcd suggested):

    ```sh
    docker run --rm -it \
        --volumes-from nvidia-driver \
        --env PATH=$PATH:/opt/nvidia/bin/ \
        --env LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/nvidia/lib \
        $(for d in /dev/nvidia*; do echo -n "--device $d "; done) \
        gcr.io/tensorflow/tensorflow:latest-gpu \
            python -c "import tensorflow as tf;tf.Session(config=tf.ConfigProto(log_device_placement=True))"
    
    ```



Below are the README from original author:
-----------------------------------


Yet another NVIDIA driver container for Container Linux (aka CoreOS).

Many different solutions to load the NVIDIA modules in a CoreOS kernel has been created during the last years, this is just another one trying to fit the *source{d}* requirements:

- Load the NVIDIA modules in the kernel of the host.
- Make available the NVIDIA libraries and binaries to other containers.
- Works with unmodified third-party containers.
- Avoid permanent changes on the host system. 

## Contents

* [Hot it works](#how-it-works)
* [Installation](#installation)
* [Usage](#usage)
* [Available Images]($available-images)
* [Custom images](#custom-images)

## How it works

Executing the `srcd/coreos-nvidia` for your CoreOS version the nvidia modules are loaded in the kernel and the devices are created in the rootfs.

```sh
source /etc/os-release
docker run --rm --privileged --volume /:/rootfs/ srcd/coreos-nvidia:${VERSION}
```

You can test the execution running the next command:

```sh
docker run --rm $(for d in /dev/nvidia*; do echo -n "--device $d "; done) \
    srcd/coreos-nvidia:${VERSION} nvidia-smi -L

// Outputs:
// GPU 0: Tesla K80 (UUID: GPU-d57ec7e8-ab97-8612-54ac-9d53a183f818)
```

## Installation

The installation is done using a systemd unit, this unit has two goals:

- Load the modules in the kernel in every startup, unload it if the service is stopped.
- Keep running a docker container called `nvidia-driver` to allow other images access to the libraries and binaries from the NVIDIA driver, using the `--volumes-from`.

Create the following systemd unit at `/etc/systemd/system/coreos-nvidia.service`:

```sh
[Unit]
Description=NVIDIA driver
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=20m
EnvironmentFile=/etc/os-release
ExecStartPre=-/usr/bin/docker rm nvidia-driver
ExecStartPre=/usr/bin/docker run --rm --privileged --volume /:/rootfs/ srcd/coreos-nvidia:${VERSION}
ExecStart=/usr/bin/docker run --rm --name nvidia-driver srcd/coreos-nvidia:${VERSION} sleep infinity
ExecStop=/usr/bin/docker stop nvidia-driver
ExecStop=-/sbin/rmmod nvidia_uvm nvidia

[Install]
WantedBy=multi-user.target
```

And now just enable and start the unit:

```sh
sudo systemctl enable /etc/systemd/system/coreos-nvidia.service
sudo systemctl start coreos-nvidia.service
```


After start the service we should see the modules loaded in the kernel:

```
lsmod | grep -i nvidia
```
```
nvidia_uvm            679936  0
nvidia              12980224  1 nvidia_uvm
```

And the `nvidia-driver` container running:

```
docker ps | grep -i nvidia-driver
```
```
8cea48f9d556   srcd/coreos-nvidia:1465.7.0   "sleep infinity"   11 hours ago   nvidia-driver
```

## Usage

To easily use the NVIDIA driver in other standard containers, we use the `--volumes-from`, this requires to run a container based on our image, the `/dev/nvidia*` devices and a setting the  `$PATH` and `$LD_LIBRARY_PATH` variables to make it work properly.

A simple example running `nvidia-smi` in a bare `fedora` container:

```sh
docker run --rm -it \
    --volumes-from nvidia-driver \
    --env PATH=$PATH:/opt/nvidia/bin/ \
    --env LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/nvidia/lib \
    $(for d in /dev/nvidia*; do echo -n "--device $d "; done) \
    fedora:26 nvidia-smi
```

Running the `tensorflow` GPU enabled container and verifying the identified devices:

```sh
docker run --rm -it \
    --volumes-from nvidia-driver \
    --env PATH=$PATH:/opt/nvidia/bin/ \
    --env LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/nvidia/lib \
    $(for d in /dev/nvidia*; do echo -n "--device $d "; done) \
    gcr.io/tensorflow/tensorflow:latest-gpu \
        python -c "import tensorflow as tf;tf.Session(config=tf.ConfigProto(log_device_placement=True))"

```

## Available Images

Eventually an image for all the Container Linux version for all the release channels should be available, to ensure this, a Travis cron is executed everyday that checks if a new Container Linux versions exists, if exists a new image will be created.

The list of images is available at: [https://hub.docker.com/r/srcd/coreos-nvidia/tags/](https://hub.docker.com/r/srcd/coreos-nvidia/tags/).

### What I can do if I can't find an image for my version?

If your version was released today, you must to wait until the nightly cron. If wasn't released today and was after *11/Oct/2016*, open an issue, something has failed. If you image is older than this you must to build the image from the Dockerfile.

## Custom images

The builds of the Docker image are managed by a Makefile.

To build a image fot the latest stable version of Linux Container and the latest version of the NVIDIA driver just execute:

```
make build
```

The configuration is done through environment variables, for example if you want to build the image for the latest alpha version you can execute:

```
COREOS_RELEASE_CHANNEL=alpha make build
```

### Variables:

- `COREOS_RELEASE_CHANNEL`: Linux Container release channel: `stable`, `beta` or `alpha`. By default `stable`
- `COREOS_VERSION`: Linux Container version, if empty the last available version for the given release channel will be used. The version is retrieved making a request to the release feed.
- `NVIDIA_DRIVER_VERSION`: NVIDIA Driver version, if empty the last available version will be used. The version is retrieve from https://github.com/aaronp24/nvidia-versions/.
- `KERNEL_VERSION`: Kernel version used in the given `COREOS_VERSION`, if empty is retrieve from the CoreOS release feed.


## License

GPLv3, see [LICENSE](LICENSE)

