# Raspberry Pi Standalone Kubelet Tutorial

Forked from original [standalone kubelet tutorial](https://github.com/kelseyhightower/standalone-kubelet-tutorial)

This tutorial will guide you through running the Kubernetes [Kubelet](https://kubernetes.io/docs/reference/generated/kubelet/) in standalone mode on Raspberry Pi 3 [ArchLinux ARM](https://archlinuxarm.org/). You will also deploy an application using a static pod, test it, then upgrade the application.
If you have different version of Raspberry Pi, this tutorial should work, just follow different installation steps ([Raspberry Pi](https://archlinuxarm.org/platforms/armv6/raspberry-pi), [Raspberry Pi 2](https://archlinuxarm.org/platforms/armv7/broadcom/raspberry-pi-2).

## Rationale

In some cases you have only one Raspberry Pi laying around and it's not enough to form a Kubernetes Cluster.

## Setup ArchLinux Arm

Follow the steps described in <https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-3>

### Setup Wifi (optional)

Connect your Raspberry Pi to your router via ethernet cable and ssh into it:

    ssh alarm@IP (default password is `alarm`)

Connect to your wifi access point:

    wifi-menu -o

Enable wifi on startup:

    netctl enable wlan0-WLAN-NAME

### Update

Update you system:

    su (default password is `root`)
    pacman -Sy pacman
    pacman-key --init
    pacman -S archlinux-keyring
    pacman-key --populate archlinux
    pacman -Syu --ignore filesystem
    pacman -S filesystem --force
    systemctl reboot

Install `sudo`:

    pacman -S sudo 

### Install Docker

Setup Docker and optional dependencies:

    sudo pacman -Syu docker btrfs-progs lxc

Enable Docker on startup:

    sudo systemctl enable docker.service

If you want to access docker from user without sudo, add user to the docker group:

    sudo usermod -aG docker $USER (and relogin)

Enable memory cgroups, edit the file below:

    sudo vi /boot/cmdline.txt

and add to the end of the line:

    cgroup_enable=memory

Reboot the systems, so that memory cgroup changes take effect:

    sudo systemctl reboot

### Verify

Check cgroups and namespaces status, verify that cgroups are enabled:

    lxc-checkconfig

Check docker logs, verify that docker is started:

    journalctl -u docker

## Install the Standalone Kubelet

Download the Kubelet configuration file:

    wget -q --show-progress --https-only --timestamping \
      https://raw.githubusercontent.com/povilasv/raspberrypi-standalone-kubelet-tutorial/master/config.yaml

Create configuration and manifests directory:

    sudo mkdir -p /etc/kubernetes/manifests

Move Kubelet config.yaml to `/etc/kubernetes` directory:

    sudo mv config.yaml /etc/kubernetes/

Download the Kubelet systemd file:

    wget -q --show-progress --https-only --timestamping \
      https://raw.githubusercontent.com/povilasv/raspberrypi-standalone-kubelet-tutorial/master/kubelet.service

Move the kubelet.service unit file to the systemd configuration directory:

    sudo mv kubelet.service /etc/systemd/system/

Start the kubelet service:

    sudo systemctl daemon-reload

    sudo systemctl enable kubelet

    sudo systemctl start kubelet

### Verification

It will take a few minutes for the kubelet container to download and initialize. Verify the kubelet is running:

    sudo systemctl status kubelet

Check that Kubelet container is running:

    docker ps

> You should see `kubelet` container running. 

View logs for Kubelet service:

    journalctl -u kubelet

## Static Pods

In this section you will deploy an application that responds to HTTP requests with its running config and version. The application configuration will be initialized before the HTTP service starts by using an init container. Once the pod has started the application configuration will be updated every 30 seconds by a configuration sidecar.

Download the `app-v0.1.0.yaml` pod manifest:

    wget -q --show-progress --https-only --timestamping \
      https://raw.githubusercontent.com/povilasv/raspberrypi-standalone-kubelet-tutorial/master/pods/app-v0.1.0.yaml

Move the `app-v0.1.0.yaml` pod manifest to the Kubelet manifest directory:

    sudo mv app-v0.1.0.yaml /etc/kubernetes/manifests/app.yaml

> Notice the `app-v0.1.0.yaml` pod manifest is being renamed to `app.yaml`. This prevents our application from being deployed twice. Each pod must have a unique `metadata.name`.

List the installed container images:

    sudo docker images

    REPOSITORY                               TAG                 IMAGE ID            CREATED             SIZE
    povilasv/configurator                    arm-0.1.0           895ce4f6c72a        13 minutes ago      2.37MB
    povilasv/app                             arm-0.1.0           87c7ade0a0f5        17 minutes ago      5.99MB
    gcr.io/google_containers/hyperkube-arm   v1.10.1             1184887730bd        11 days ago         557MB
    k8s.gcr.io/pause-arm                     3.1                 e11a8cbeda86        4 months ago        374kB

List the running containers:

    docker ps

> You should see three containers running which represent the `app` pod and a `kubelet` container. Docker does not understand pods so the containers are listed as individual containers following the Kubernetes naming convention. 

### Verification

At this point the `app` pod is up and running on port 80 in the host namespace. Make a HTTP request to the `app` pod using the localhost address:

    curl http://127.0.0.1


    version: 0.1.0
    hostname: alarmpi
    key: 1524542873

Wait about 30 seconds and make another HTTP request to the `app` pod:

    curl http://127.0.0.1


    version: 0.1.0
    hostname: alarmpi
    key: 1524542903

> Notice the `key` field has changed. This configuration setting is being updated by the `configurator` sidecar container running in the `app` pod.

### Testing Remote Access

The `app` pod is listening on `0.0.0.0:80` in the host network and is accessible via local network.

Make a HTTP request to the `app` pod using the `standalone-kubelet` local network's IP address:

    curl http://IP


    version: 0.1.0
    hostname: alarmpi
    key: 1524543083

## Updating Static Pods

    ssh alarm@IP

Download the `app-v0.2.0.yaml` pod manifest:

    wget -q --show-progress --https-only --timestamping \
      https://raw.githubusercontent.com/povilasv/raspberrypi-standalone-kubelet-tutorial/master/pods/app-v0.2.0.yaml

Move the `app-v0.2.0.yaml` pod manifest to the kubelet manifest directory:

    sudo mv app-v0.2.0.yaml /etc/kubernetes/manifests/app.yaml

> Notice the `app-v0.2.0.yaml` is being renamed to `app.yaml`. This overwrites the current pod manifest and will force the kubelet upgrade the `app` pod.

List the installed container images:

    docker images


    REPOSITORY                               TAG                 IMAGE ID            CREATED             SIZE
    povilasv/app                             arm-0.2.0           6cfd36f23609        10 minutes ago      5.99MB
    povilasv/configurator                    arm-0.1.0           895ce4f6c72a        36 minutes ago      2.37MB
    povilasv/app                             arm-0.1.0           87c7ade0a0f5        41 minutes ago      5.99MB
    gcr.io/google_containers/hyperkube-arm   v1.10.1             1184887730bd        11 days ago         557MB
    k8s.gcr.io/pause-arm                     3.1                 e11a8cbeda86        4 months ago        374kB

> Notice the `povilasv/app:arm-0.2.0` image has been added to the local repository.

### Verification

At this point `app` version `0.2.0` is up and running. Make a HTTP request to the `app` pod using the localhost address:

    curl http://127.0.0.1

    version: 0.2.0
    hostname: alarmpi
    key: 1524543455

## Cleanup

    sudo systemctl disable kubelet

    sudo rm /etc/kubernetes/manifests/app.yaml

## Advanced tutorial

If you liked this tutorial, I have another advanced [tutorial where I cross compile the Kubernetes Kubelet for ARM](https://povilasv.me/raspberrypi-kubelet) and run it as a simple binary on Raspberry PI.
