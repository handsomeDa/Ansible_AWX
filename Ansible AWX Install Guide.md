# Ansible AWX_Linux Installation Steps
## 1. Install basic sources:
```shell
yum -y install epel-release
yum -y update
yum -y install git gcc gcc-c++ ansible nodejs gettext device-mapper-persistent-data lvm2 bzip2 python3-pip wget nano
```
## 2. Install Docker service and enable it:
```shell
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf install docker-ce --nobest -y
systemctl enable --now docker.service
```
## 3. Set Python to use version 3 and update related modules:
```shell
alternatives --set python /usr/bin/python3
pip3 install -U pip setuptools
pip3 install setuptools_rust
pip3 install wheel
```
## 4. Install Docker Compose:
```shell
pip3 install docker-compose
```
## 5. Clone the Ansible AWX source:
```shell
git clone -b "17.1.0" https://github.com/ansible/awx.git
```
## 6. Generate an SSL key:
```shell
openssl rand -base64 30 > sslkey
```
## 7. Create physical directories that map to Docker virtual directories:
```shell
mkdir -p /home/asbadmin/awx_data/awxcompose
mkdir -p /home/asbadmin/awx_data/pgdocker
mkdir -p /home/asbadmin/awx_data/projects
```
Or:
```shell
mkdir -p /home/asbadmin/awx_data/awxcompose /home/asbadmin/awx_data/pgdocker /home/asbadmin/awx_data/projects
```
## 8. Navigate to the Ansible AWX playbook installation directory and edit the inventory file:
```shell
cd awx/installer/
vi inventory
```
## 9. Edit the parameters in the inventory file:
```yaml
[all:vars]
dockerhub_base=ansible
awx_task_hostname=awx
awx_web_hostname=awxweb
postgres_data_dir="/home/asbadmin/awx_data/pgdocker"     ## Physical directory generated earlier - PostgreSQL
host_port=80
host_port_ssl=443
docker_compose_dir="/home/asbadmin/awx_data/awxcompose"  ## Physical directory generated earlier - Docker Compose
pg_username=awx
pg_password=awxpassword
pg_database=awx
pg_port=5432
admin_user=admin
admin_password=password
create_preload_data=False
secret_key=PblrUyeSvBMVWqHaaDMFcABcjzgG5dAhfeOgge4S      ## SSL key generated earlier
awx_official=true
project_data_dir= /home/asbadmin/awx_data/projects       ## Physical directory generated earlier - Ansible AWX Projects
```
## 10. Run the Ansible playbook installation:
```shell
ansible-playbook -i inventory install.yml
```



# Supplement
## Startup Ansible AWX
```shell
cd /home/asbadmin/awx_data/awxcompose
docker-compose up –d
```
## Shutdown Ansible AWX
```shell
cd /home/asbadmin/awx_data/awxcompose
docker-compose stop
```