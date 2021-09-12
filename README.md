# containerized_drivers

# 1. Setup Ubuntu
```
sudo visudo
sudo vi /etc/apt/apt.conf.d/20auto-upgrades 
sudo apt-get install linux-image-5.4.0-42-generic linux-headers-5.4.0-42-generic linux-modules-extra-5.4.0-42-generic
sudo vi /etc/default/grub
sudo update-grub
reboot
dpkg --get-selections |grep linux-
sudo apt-get autoremove --purge linux-{headers,image,modules}-5.11.0-34
dpkg --get-selections |grep linux-
sudo apt-get autoremove --purge linux-{headers,image,modules}-5.8.0-43
sudo vi /etc/default/grub
sudo update-grub
shutdown -h now
```

# 2. Install Docker
```
sudo apt-get install curl
curl https://get.docker.com | sh && sudo systemctl --now enable docker
sudo cp /lib/systemd/system/docker.service /etc/systemd/system/
sudo vi /etc/systemd/system/docker.service 
~~~~~~~~~~
# Edit as like below:
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --data-root /mnt/docker
#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
~~~~~~~~~~
sudo systemctl daemon-reload
sudo systemctl restart docker
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list |   sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update
sudo apt-get install -y nvidia-docker2
sudo systemctl restart docker
```

# 3. Install contaierized OFED
```
sudo apt-get install -y git
git clone https://github.com/Mellanox/ofed-docker.git
cd ofed-docker/
vi ubuntu/Docerfile
~~~~~~~~~~
#   apt-get -yq install curl perl && \
    apt-get -yq install curl perl libcap2 && \
#   /root/${D_OFED_PATH}/mlnxofedinstall --without-fw-update --kernel-only --force ${D_WITHOUT_FLAGS} && \
    /root/${D_OFED_PATH}/mlnxofedinstall --with-nfsrdma --with-nvmf --enable-gds --force && \
~~~~~~~~~~
sudo docker build -t ofed-driver --build-arg D_BASE_IMAGE=ubuntu:20.04 --build-arg D_OFED_VERSION=5.3-1.0.0.1 --build-arg D_OS=ubuntu20.04 --build-arg D_ARCH=x86_64 ubuntu/
sudo docker images
sudo docker run --rm -itd -v /run/mellanox/drivers:/run/mellanox/drivers:shared -v /etc/network:/etc/network -v /host/etc:/host/etc -v /host/lib/udev:/lib/udev --net=host --privileged ofed-driver
lsmod |grep -i ib
```

# 4. Install containerized Nvidia-driver
```
#vi /etc/nvidia-container-runtime/config.toml
sudo sed -i 's/^#root/root/' /etc/nvidia-container-runtime/config.toml
sudo tee /etc/modules-load.d/ipmi.conf <<< "ipmi_msghandler"   && sudo tee /etc/modprobe.d/blacklist-nouveau.conf <<< "blacklist nouveau"   && sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf <<< "options nouveau modeset=0"
sudo update-initramfs -u
sudo reboot
sudo docker pull nvcr.io/nvidia/driver:470.57.02-ubuntu20.04
sudo docker run --name nvidia-driver -d --privileged --pid=host   -v /run/nvidia:/run/nvidia:shared   -v /var/log:/var/log   --restart=unless-stopped   nvcr.io/nvidia/driver:470.57.02-ubuntu20.04
sudo docker logs -f nvidia-driver
lsmod |grep -i nvidia
ls /run/nvidia/driver/
```

# 5. Test nvidia-smi in some containers
```
sudo docker run -it --rm --gpus all --name="ubuntu" ubuntu:20.04
```


# 6. Install containerized peer-mem driver
```
sudo docker build -t nv-peer-mem --build-arg D_BASE_IMAGE=ubuntu:20.04 --build-arg D_NV_PEER_MEM_BRANCH=1.0-9 gpu-direct/ubuntu/
sudo docker run --rm -it -v /run/mellanox/drivers:/run/mellanox/drivers -v /run/nvidia/driver:/run/nvidia/drivers --privileged nv-peer-mem
```
