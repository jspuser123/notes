# Docker install
```bash
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo docker login -u <username> -p <password>
```
or

# 1. docker install apt list
```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null 
```
# 2. apt install docker
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin 
```
# 3. add user to docker group
```bash
sudo usermod -aG docker ${USER}   
```
# 4. test docker
```bash
docker run hello-world 
```


# docker fix
```bash
docker version 
sudo systemctl restart docker 
sudo systemctl stop docker 
sudo dockerd --debug 
sudo rm -rf /var/run/docker/runtime-runc/moby/docker_id 
sudo apt-get remove docker docker-engine docker.io containerd runc 
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
# majar error cgroup
```bash
sudo apt-get install cgroupfs-mount
sudo cgroupfs-mount

# or

quiet selinux=0
quiet splash cgroup_enable=memory swapaccount=1 systemd.unified_cgroup_hierarchy=0
sudo update-grub
sudo reboot
```