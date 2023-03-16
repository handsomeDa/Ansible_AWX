# Ansible AWX_Linux] - Problems encountered during installation
## 1. YUM baseURL cannot connect properly
```shell
[root@awxTest ~]# yum -y install epel-release
/usr/local/lib/python3.6/site-packages/OpenSSL/crypto.py:8: CryptographyDeprecationWarning: Python 3.6 is no longer supported by the Python core team. Therefore, support for it is deprecated in cryptography and will be removed in a future release.
  from cryptography import utils, x509

CentOS-8 - AppStream                                                                                                                                          0.0  B/s |   0  B     00:32
Error: Failed to synchronize cache for repo 'AppStream'

[root@awxTest ~]# yum -y install git gcc gcc-c++ nodejs gettext device-mapper-persistent-data lvm2 bzip2 python3-pip wget nano
/usr/local/lib/python3.6/site-packages/OpenSSL/crypto.py:8: CryptographyDeprecationWarning: Python 3.6 is no longer supported by the Python core team. Therefore, support for it is deprecated in cryptography and will be removed in a future release.
  from cryptography import utils, x509

CentOS-8 - AppStream                                                                                                                                          0.0  B/s |   0  B     00:32
Error: Failed to synchronize cache for repo 'AppStream'
```
Execute the following command to change the mirror location, and then run the YUM command again.
```shell
cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
```
## 2. Unable to retrieve the AWX Project from Git
```shell
[asbadmin@awxTest ~]$ git clone -b "17.1.0" https://github.com/ansible/awx.git
Cloning into 'awx'...
```
=> Cannot access external network, please provide the URL and destination to ISD for whitelist opening, and then re-download.
## 3. The account executing the playbook does not have permission to use the docker command.
```shell
TASK [local_docker : Run migrations in task container] *****************************************************************************************************************************
fatal: [localhost]: FAILED! => {"changed": true, "cmd": "docker-compose run --rm --service-ports task awx-manage migrate --no-input", "delta": "0:00:00.733690", "end": "2023-03-14 10:10:08.542804", "msg": "non-zero return code", "rc": 1, "start": "2023-03-14 10:10:07.809114", "stderr": "Couldn't connect to Docker daemon at http+docker://localhost - is it running?\n\nIf it's at a non-standard location, specify the URL with the DOCKER_HOST environment variable.", "stderr_lines": ["Couldn't connect to Docker daemon at http+docker://localhost - is it running?", "", "If it's at a non-standard location, specify the URL with the DOCKER_HOST environment variable."], "stdout": "", "stdout_lines": []}
```
After executing "docker ps" command, the result is "Permission Denied".
```shell
[asbadmin@awxTest awx]$ docker ps
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.40/containers/json: dial unix /var/run/docker.sock: connect: permission denied
```
=> Switch to root and add "asbadmin" to the docker group
```shell
[root@awxTest ~]# groupadd docker
[root@awxTest ~]# usermod -aG docker asbadmin
```
=> Log out and close the Terminal, then log in again to use it, and verify with "docker ps" again.
## 4. The postgresql image pulled down by Docker is Version 12, but the playbook check is Version 10, so it will never pass the check no matter what.
```shell
TASK [local_docker : Check for existing Postgres data (run from inside the container for access to file)] ************************************************************************************
fatal: [localhost]: FAILED! => {"changed": true, "cmd": "docker run --rm -v '/home/asbadmin/awx_data/pgdocker:/var/lib/postgresql' centos:8 bash -c  \"[[ -f /var/lib/postgresql/10/data/PG_VERSION ]] && echo 'exists'\"\n", "delta": "0:00:11.864027", "end": "2023-03-14 10:19:32.995636", "msg": "non-zero return code", "rc": 1, "start": "2023-03-14 10:19:21.131609", "stderr": "Unable to find image 'centos:8' locally\n8: Pulling from library/centos\na1d0c7532777: Pulling fs layer\na1d0c7532777: Verifying Checksum\na1d0c7532777: Download complete\na1d0c7532777: Pull complete\nDigest: sha256:a27fd8080b517143cbbbab9dfb7c8571c40d67d534bbdee55bd6c473f432b177\nStatus: Downloaded newer image for centos:8", "stderr_lines": ["Unable to find image 'centos:8' locally", "8: Pulling from library/centos", "a1d0c7532777: Pulling fs layer", "a1d0c7532777: Verifying Checksum", "a1d0c7532777: Download complete", "a1d0c7532777: Pull complete", "Digest: sha256:a27fd8080b517143cbbbab9dfb7c8571c40d67d534bbdee55bd6c473f432b177", "Status: Downloaded newer image for centos:8"], "stdout": "", "stdout_lines": []}
...ignoring
```
=> Find the following directory "awx/installer/roles/local_docker/tasks/upgrade_postgres.yml", and modify the following contents
```yaml
- name: Check for existing Postgres data (run from inside the container for access to file)
  shell:
    cmd: |
      {{ container_command }} "[[ -f /var/lib/postgresql/10/data/PG_VERSION ]] && echo 'exists'"
  register: pg_version_file
  ignore_errors: true
```
=> Change "10" to "12", i.e. "{{ container_command }} "[[ -f /var/lib/postgresql/<mark>12</mark>/data/PG_VERSION ]] && echo 'exists'"

```yaml
- name: Record Postgres version
  shell: |
    {{ container_command }} "cat /var/lib/postgresql/10/data/PG_VERSION"
  register: old_pg_version
  when: pg_version_file is defined and pg_version_file.stdout == 'exists'
```
=> Change "10" to "12", i.e. "{{ container_command }} "cat /var/lib/postgresql/<mark>12</mark>/data/PG_VERSION"
## 5. pip/pip3 does not have the docker module.
```shell
TASK [local_docker : Remove AWX containers before migrating postgres so that the old postgres container does not get used] *******************************************************************
fatal: [localhost]: FAILED! => {"changed": false, "msg": "Failed to import the required Python library (Docker SDK for Python: docker (Python >= 2.7) or docker-py (Python 2.6)) on awxTest's Python /usr/bin/python3. Please read module documentation and install in the appropriate location. If the required library is installed, but Ansible is using the wrong Python interpreter, please consult the documentation on ansible_python_interpreter, for example via `pip install docker` or `pip install docker-py` (Python 2.6). The error was: No module named 'docker'"}
...ignoring
```
```shell
TASK [local_docker : Start the containers] ***************************************************************************************************************************************************
fatal: [localhost]: FAILED! => {"changed": false, "msg": "Failed to import the required Python library (Docker SDK for Python: docker (Python >= 2.7) or docker-py (Python 2.6)) on awxTest's Python /usr/bin/python3. Please read module documentation and install in the appropriate location. If the required library is installed, but Ansible is using the wrong Python interpreter, please consult the documentation on ansible_python_interpreter, for example via `pip install docker` or `pip install docker-py` (Python 2.6). The error was: No module named 'docker'"}
```
=> Switch to root account and reinstall.
```shell
pip3 install docker-compose
```



# Supplement
## 1. Issue with pulling postgresql using docker module in Ansible playbook:
Executing the following command returns nothing.
```shell
docker run --rm -v '/myawx/pgdocker:/var/lib/postgresql' centos:8 bash -c  "[[ -f /var/lib/postgresql/10/data/PG_VERSION ]] && echo 'exists'"
```
Executing the following command returns "exists".
```shell
docker run --rm -v '/myawx/pgdocker:/var/lib/postgresql' centos:8 bash -c  "[[ -f /var/lib/postgresql/12/data/PG_VERSION ]] && echo 'exists'"
```
=> Indicates that the installed version inside docker is 12, not 10, so the check will never pass.


## 2. Executing the following command can enter docker:
```shell
docker run --rm -it -v '/myawx/pgdocker:/var/lib/postgresql' centos:8 bash
```
