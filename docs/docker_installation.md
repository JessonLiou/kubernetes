1. sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

2. sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

3. yum install docker-ce-17.06.2.ce-1.el7.centos

4. vim /etc/docker/daemon.json

{
  "exec-opts": ["native.cgroupdriver=systemd"]
}

4. systemctl start docker

5. docker --version

  Docker version 17.06.2-ce, build cec0b72
