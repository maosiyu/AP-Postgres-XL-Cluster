##### 环境

|HostName|IP|DES|
|----|----|----|
|test1| 192.168.180.46  | 主控机 |
|test2| 192.168.180.47  | 工作节点 GTM |
|test3| 192.168.180.48  | 工作节点 (GTM_Proxy, Coordinator, DataNode) |
|test4| 192.168.181.18  | 工作节点 (GTM_Proxy, Coordinator, DataNode) |

**注意： 主控机不参加作业**

##### 1.在中控机上创建 postgres 用户
``` ruby
[root@test1]~# useradd -m -d /home/postgres postgres
[root@test1]~# passwd postgres
pwd@123456
[root@test1]~#
# 追加 sudo 免密
[root@test1]~# cat >> /etc/sudoers << eric
postgres ALL=(ALL) NOPASSWD: ALL
eric
[root@test1]~#
```

##### 2.生成SSH Key
``` ruby
[root@test1]~# su - postgres
[postgres@test1 ~]$
[postgres@test1 ~]$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/postgres/.ssh/id_rsa):
Created directory '/home/postgres/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/postgres/.ssh/id_rsa.
Your public key has been saved in /home/postgres/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:g7tMJtyN6F4/QsUjQb5QQnf9QdbeNhDfflc39+BMB48 postgres@test1
The key's randomart image is:
+---[RSA 2048]----+
|   .o.+ .. .ooo  |
|     =..  ....o+.|
|    . .o   . oE+B|
|     ..o+   .+.=O|
|      ooS.    o.*|
|   . o.+ .      o|
|    +.B .        |
|   . *.o.        |
|   .o o...       |
+----[SHA256]-----+
[postgres@test1 ~]$
[postgres@test1 ~]$
```

##### 3.配置SSH免密

###### 3.1 添加集群 配置文件
``` ruby

[postgres@test1 ~]$ mkdir -p /home/postgres/deploy
[postgres@test1 ~]$
[postgres@test1 ~]$ cat > /home/postgres/deploy/hosts.ini << eric
[servers]
192.168.180.47
192.168.180.48
192.168.181.18

[gtm]
gtm ansible_ssh_host=192.168.181.18

[gtm_proxy]
192.168.180.47
192.168.180.48

[datanode]
datanode1 ansible_ssh_host=192.168.180.47
datanode2 ansible_ssh_host=192.168.180.48

[coordinator]
coordinator1 ansible_ssh_host=192.168.180.47
coordinator2 ansible_ssh_host=192.168.180.48


[all:vars]
username = postgres
deploy_dir = /home/postgres/deploy
eric

[postgres@test1 ~]$
```

###### 3.2 添加 ansible-playbook 配置文件
``` ruby
[postgres@test1 ~]$ cat > /home/postgres/deploy/create_users.yml << eric
# 使用方法 ansible-playbook -i hosts.ini create_users.yml -u root -k
---
- hosts: all

  tasks:
    - name: create user
      user: name={{ username }} shell=/bin/bash createhome=yes

    - name: set authorized key
      authorized_key:
        user: "{{ username }}"
        key: "{{ lookup('file', '/home/{{ username }}/.ssh/id_rsa.pub') }}"
        state: present

    - name: update sudoers file
      lineinfile:
        # 必须参数，指定要操作的文件。
        dest: /etc/sudoers
        # 使用此参数指定文本内容。
        line: '{{ username }} ALL=(ALL) NOPASSWD: ALL'
        # 借助insertafter参数可以将文本插入到“指定的行”之后，
        # insertafter参数的值可以设置为EOF或者正则表达式，EOF为End Of File之意，
        # 表示插入到文档的末尾，默认情况下insertafter的值为EOF，
        # 如果将insertafter的值设置为正则表达式，表示将文本插入到匹配到正则的行之后，
        # 如果正则没有匹配到任何行，则插入到文件末尾，当使用backrefs参数时，此参数会被忽略。
        insertafter: EOF
        # 使用正则表达式匹配对应的行，当替换文本时，如果有多行文本都能被匹配，则只有最后面被匹配到的那行文本才会被替换，
        # 当删除文本时，如果有多行文本都能被匹配，这么这些行都会被删除。
        regexp: '^{{ username }} .*'
        # 当想要删除对应的文本时，需要将state参数的值设置为absent，absent为缺席之意，表示删除，state的默认值为present。
        state: present
eric

[postgres@test1 ~]$
```

###### 3.3 修改ansible默认配置, 跳过 ssh 首次连接提示验证
``` ruby
[postgres@test1 ~]$ cat > /home/postgres/deploy/ansible.cfg << eric
[defaults]
# 跳过 ssh 首次连接提示验证
host_key_checking=False
# 关闭警告提示
command_warnings=False
deprecation_warnings=False
eric

[postgres@test1 ~]$
```

###### 3.4 批量执行 SSH免密
``` ruby
[postgres@test1 deploy]$ ansible-playbook -i hosts.ini create_users.yml -u root -k
SSH password:

PLAY [all] *****************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************
ok: [192.168.180.48]
ok: [192.168.180.47]
ok: [192.168.181.18]

TASK [create user] *********************************************************************************************************************
ok: [192.168.180.48]
ok: [192.168.180.47]
changed: [192.168.181.18]

TASK [set authorized key] **************************************************************************************************************
ok: [192.168.180.48]
changed: [192.168.181.18]
ok: [192.168.180.47]

TASK [update sudoers file] *************************************************************************************************************
ok: [192.168.180.48]
ok: [192.168.180.47]
changed: [192.168.181.18]

PLAY RECAP *****************************************************************************************************************************
192.168.180.47             : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.180.48             : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.181.18             : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

[postgres@test1 deploy]$
```

###### 3.5 测试互信是否成功
``` ruby
[postgres@test1 deploy]$
[postgres@test1 deploy]$ ansible -i hosts.ini all -m shell -a 'whoami'
192.168.180.47 | CHANGED | rc=0 >>
postgres

192.168.180.48 | CHANGED | rc=0 >>
postgres

192.168.181.18 | CHANGED | rc=0 >>
postgres

[postgres@test1 deploy]$
[postgres@test1 deploy]$
[postgres@test1 deploy]$ ansible -i hosts.ini all -m shell -a 'whoami' -b
192.168.181.18 | CHANGED | rc=0 >>
root

192.168.180.48 | CHANGED | rc=0 >>
root

192.168.180.47 | CHANGED | rc=0 >>
root

[postgres@test1 deploy]$
```

###### 3.6 添加hostname映射
``` ruby
[postgres@test1 deploy]$ cat > /home/postgres/deploy/modify_hosts.yml << eric
# 使用方法 ansible-playbook -i hosts.ini modify_hosts.yml
---
- hosts: servers
  tasks:
    - name: 配置 hosts 映射
      sudo: yes
      lineinfile:
        dest: '/etc/hosts'
        line: '{{ item.key }} {{ item.value }}'
        regexp: '{{ item.key }} {{ item.value }}'
        state: present
      # 定义集合，并循环执行所在的模块
      with_items:
        - { key: "192.168.181.18", value: "gtm" }
        - { key: "192.168.180.47", value: "node1" }
        - { key: "192.168.180.48", value: "node2" }
eric

[postgres@test1 deploy]$
```

##### 4.安装部署

###### 4.1 下载
官网：https://www.postgres-xl.org/
官网下载地址：https://www.postgres-xl.org/downloads/
``` ruby
# 下载到  /home/postgres/deploy/ 目录
[postgres@test1 deploy]$ wget https://www.postgres-xl.org/downloads/postgres-xl-10r1.1.tar.gz -P /home/postgres/deploy/
[postgres@test1 deploy]$
```
###### 或者使用bz2文件
下载*.tar.bz2：wget https://www.postgres-xl.org/downloads/postgres-xl-9.5r1.6.tar.bz2
``` ruby
# 安装转换工具
[postgres@test1 deploy]$ yum install -y bzip2
# bz2 转换 tar
[postgres@test1 deploy]$ bunzip2 postgres-xl-9.5r1.6.tar.bz2
[postgres@test1 deploy]$
[postgres@test1 deploy]$ tar -xf postgres-xl-9.5r1.6.tar
```
 **`注意`** `如果使用bz2文件，需要修改deploy.yml中压缩包的名称`

###### 4.2 部署到节点机
``` ruby
[postgres@test1 deploy]$ cat > /home/postgres/deploy/deploy.yml << eric
# 使用方法 ansible-playbook -i hosts.ini deploy.yml
---
- hosts: servers
  tasks:

    - name: 创建deploy目录
      shell: 'mkdir -p {{ deploy_dir }}'

    - name: 安装postgres-xl
      sudo: yes
      shell: yum install -y flex bison readline-devel zlib-devel openjade docbook-style-dsssl

    - name: '上传文件'
      # 将本地文件复制到远程服务器
      copy:
        src: '{{ item.src }}'
        dest: '{{ item.dest }}'
      with_items:
        - { src: '{{ deploy_dir }}/postgres-xl-10r1.1.tar.gz', dest: '{{ deploy_dir }}/postgres-xl-10r1.1.tar.gz' }

    - name: 解压
      shell: cd {{ deploy_dir }} && tar -zxvf postgres-xl-10r1.1.tar.gz

    - name: 安装
      sudo: yes
      shell: cd {{ deploy_dir }}/postgres-xl-10r1.1 && ./configure --prefix=/home/postgres/pgxl && make && make install

    - name: 添加环境变量
      lineinfile:
        dest:  ~/.bashrc
        line: '{{ item.path }}'
        regexp: '^{{ item.path }} .*'
        state: absent
      # 定义集合，并循环执行所在的模块
      with_items:
        - { path: 'export PGHOME=/home/postgres/pgxl' }
        - { path: 'export PGUSER=postgres' }
        - { path: 'export LD_LIBRARY_PATH=\$PGHOME/lib' }
        - { path: 'export PATH=\$PGHOME/bin:\$PATH' }
eric

[postgres@test1 deploy]$
```

###### 4.3 初始化
``` ruby
[postgres@test1 deploy]$ cat > /home/postgres/deploy/init.yml << eric
# 使用方法 ansible-playbook -i hosts.ini init.yml
---
- hosts: gtm
  tags:
    - gtm
  tasks:
    - name: 初始化 gtm
      shell: 'mkdir -p {{ deploy_dir }}/data/gtm/ && mkdir -p {{ deploy_dir }}/logs/ && initgtm -D {{ deploy_dir }}/data/gtm/ -Z gtm'

    - name: 配置 gtm
      lineinfile:
        dest: '{{ deploy_dir }}/data/gtm/gtm.conf'
        line: '{{ item.key }} = {{ item.value }}'
        regexp: '^{{ item.key }}.*'
        state: present
      # 定义集合，并循环执行所在的模块
      with_items:
        - { key: "nodename", value: "'gtm'" }
        - { key: "listen_addresses", value: "'*'" }
        - { key: "port", value: "6666" }
        - { key: "startup", value: "ACT" }
        - { key: "keepalives_idle", value: "60" }
        - { key: "keepalives_interval", value: "10" }
        - { key: "keepalives_count", value: "10" }
        - { key: "log_file", value: "'{{ deploy_dir }}/logs/gtm.log'" }
        - { key: "log_min_messages", value: "WARNING" }


- hosts: gtm_proxy
  tags:
    - gtm_proxy
  tasks:
    - name: 初始化 gtm_proxy
      shell: 'mkdir -p {{ deploy_dir }}/data/gtm_proxy/ && mkdir -p {{ deploy_dir }}/logs && initgtm -D {{ deploy_dir }}/data/gtm_proxy/ -Z gtm_proxy'

    - name: 配置 gtm_proxy
      lineinfile:
        dest: '{{ deploy_dir }}/data/gtm_proxy/gtm_proxy.conf'
        line: '{{ item.key }} = {{ item.value }}'
        regexp: '^{{ item.key }}.*'
        state: present
      # 定义集合，并循环执行所在的模块
      with_items:
        - { key: "nodename", value: "'gtm_proxy'" }
        - { key: "listen_addresses", value: "'*'" }
        - { key: "port", value: "6661" }
        - { key: "worker_threads", value: "1" }
        - { key: "gtm_host", value: "'gtm'" }
        - { key: "gtm_port", value: "6666" }
        - { key: "gtm_connect_retry_interval", value: "5" }
        - { key: "keepalives_idle", value: "60" }
        - { key: "keepalives_interval", value: "10" }
        - { key: "keepalives_count", value: "10" }
        - { key: "log_file", value: "'{{ deploy_dir }}/logs/gtm_proxy.log'" }
        - { key: "log_min_messages", value: "WARNING" }


- hosts: datanode
  tags:
    - datanode
  tasks:
    - name: 初始化 datanode
      shell: 'mkdir -p {{ deploy_dir }}/data/datanode/ && initdb -D {{ deploy_dir }}/data/datanode/ --nodename {{ inventory_hostname }}'

    - name: 配置 datanode
      lineinfile:
        dest: '{{ deploy_dir }}/data/datanode/postgresql.conf'
        line: '{{ item.key }} = {{ item.value }}'
        regexp: '^{{ item.key }}.*'
        state: present
      # 定义集合，并循环执行所在的模块
      with_items:
        - { key: "listen_addresses", value: "'*'" }
        - { key: "port", value: "15432" }
        - { key: "pooler_port", value: "6668" }
        - { key: "max_pool_size", value: "100" }
        - { key: "pool_conn_keepalive", value: "600" }
        - { key: "pool_maintenance_timeout", value: "30" }
        - { key: "max_coordinators", value: "16" }
        - { key: "max_datanodes", value: "16" }
        - { key: "gtm_host", value: "'localhost'" }
        - { key: "gtm_port", value: "6661" }
        - { key: "pgxc_node_name", value: "'{{ inventory_hostname }}'" }

    - name: 配置 datanode
      lineinfile:
        dest: '{{ deploy_dir }}/data/datanode/pg_hba.conf'
        line: 'host all all 0.0.0.0/0 trust'
        regexp: 'host all all 0.0.0.0/0 trust'
        state: present


- hosts: coordinator
  tags:
    - coordinator
  tasks:
    - name: 初始化 coordinator
      shell: 'mkdir -p {{ deploy_dir }}/data/coordinator/ && initdb -D {{ deploy_dir }}/data/coordinator/ --nodename {{ inventory_hostname }}'

    - name: 配置 coordinator
      lineinfile:
        dest: '{{ deploy_dir }}/data/coordinator/postgresql.conf'
        line: '{{ item.key }} = {{ item.value }}'
        regexp: '^{{ item.key }}.*'
        state: present
      # 定义集合，并循环执行所在的模块
      with_items:
        - { key: "listen_addresses", value: "'*'" }
        - { key: "port", value: "5432" }
        - { key: "pooler_port", value: "6667" }
        - { key: "max_pool_size", value: "100" }
        - { key: "pool_conn_keepalive", value: "600" }
        - { key: "pool_maintenance_timeout", value: "30" }
        - { key: "max_coordinators", value: "16" }
        - { key: "max_datanodes", value: "16" }
        - { key: "gtm_host", value: "'localhost'" }
        - { key: "gtm_port", value: "6661" }
        - { key: "pgxc_node_name", value: "'{{ inventory_hostname }}'" }

    - name: 配置 coordinator
      lineinfile:
        dest: '{{ deploy_dir }}/data/coordinator/pg_hba.conf'
        line: 'host all all 0.0.0.0/0 trust'
        regexp: 'host all all 0.0.0.0/0 trust'
        state: present
eric

[postgres@test1 deploy]$
```

###### 4.4 启动
``` ruby
[postgres@test1 deploy]$ cat > /home/postgres/deploy/start.yml << eric
# 使用方法 ansible-playbook -i hosts.ini start.yml
---
- hosts: gtm
  tags:
    - gtm
  tasks:
    - name: 启动 gtm
      shell: setsid gtm_ctl start -Z gtm -D {{ deploy_dir }}/data/gtm/ &


- hosts: gtm_proxy
  tags:
    - gtm_proxy
  tasks:
    - name: 启动 gtm_proxy
      shell: setsid gtm_ctl start -Z gtm_proxy -D {{ deploy_dir }}/data/gtm_proxy/ &


- hosts: datanode
  tags:
    - datanode
  tasks:
    - name: 启动 datanode
      shell: pg_ctl start -Z datanode -D {{ deploy_dir }}/data/datanode/


- hosts: coordinator
  tags:
    - coordinator
  tasks:
    - name: 启动 coordinator 
      shell: pg_ctl start -Z coordinator -D {{ deploy_dir }}/data/coordinator/

eric

[postgres@test1 deploy]$
```

###### 4.5停止
``` ruby
[postgres@test1 deploy]$ cat > /home/postgres/deploy/stop.yml << eric
# 使用方法 ansible-playbook -i hosts.ini stop.yml
---
- hosts: coordinator
  tags:
    - coordinator
  tasks:
    - name: 停止 coordinator
      shell: pg_ctl stop -Z coordinator -D {{ deploy_dir }}/data/coordinator/
      ignore_errors: yes


- hosts: datanode
  tags:
    - datanode
  tasks:
    - name: 停止 datanode
      shell: pg_ctl stop -Z datanode -D {{ deploy_dir }}/data/datanode/
      ignore_errors: yes


- hosts: gtm_proxy
  tags:
    - gtm_proxy
  tasks:
    - name: 停止 gtm_proxy
      shell: gtm_ctl stop -Z gtm_proxy -D {{ deploy_dir }}/data/gtm_proxy/
      ignore_errors: yes


- hosts: gtm
  tags:
    - gtm
  tasks:
    - name: 停止 gtm
      shell: gtm_ctl stop -Z gtm -D {{ deploy_dir }}/data/gtm/
      ignore_errors: yes
eric

[postgres@test1 deploy]$
```

###### 4.6清除
``` ruby
[postgres@test1 deploy]$ cat > /home/postgres/deploy/cleanup_data.yml << eric
# 使用方法 ansible-playbook -i hosts.ini cleanup_data.yml
---
- hosts: coordinator
  tags:
    - coordinator
  tasks:
    - name: 清除 coordinator
      shell: rm -rf {{ deploy_dir }}/data/coordinator/
      ignore_errors: yes


- hosts: datanode
  tags:
    - datanode
  tasks:
    - name: 清除 datanode
      shell: rm -rf {{ deploy_dir }}/data/datanode/
      ignore_errors: yes


- hosts: gtm_proxy
  tags:
    - gtm_proxy
  tasks:
    - name: 清除 gtm_proxy
      shell: rm -rf {{ deploy_dir }}/data/gtm_proxy/
      ignore_errors: yes


- hosts: gtm
  tags:
    - gtm
  tasks:
    - name: 清除 gtm
      shell: rm -rf {{ deploy_dir }}/data/gtm/
      ignore_errors: yes
eric

[postgres@test1 deploy]$
```

###### 4.7 执行注册
``` ruby
# node1 上执行
psql -p 15432 -c "CREATE NODE datanode2 WITH (TYPE='datanode',HOST='node2',PORT=15432)"
psql -p 15432 -c "CREATE NODE coordinator1 WITH (TYPE='coordinator',HOST='node1',PORT=5432)"
psql -p 15432 -c "CREATE NODE coordinator2 WITH (TYPE='coordinator',HOST='node2',PORT=5432)"
psql -p 15432 -c "select pgxc_pool_reload()" # 重新加载
psql -p 15432 -c "select * from pgxc_node" # 查看结果

psql -p 5432 -c "CREATE NODE coordinator2 WITH (TYPE='coordinator',HOST='node2',PORT=5432)"
psql -p 5432 -c "CREATE NODE datanode1 WITH (TYPE='datanode',HOST='node1',PORT=15432)"
psql -p 5432 -c "CREATE NODE datanode2 WITH (TYPE='datanode',HOST='node2',PORT=15432)"
psql -p 5432 -c "select pgxc_pool_reload()" # 重新加载
psql -p 5432 -c "select * from pgxc_node" # 查看结果
```
``` ruby
# node2 上执行
psql -p 15432 -c "CREATE NODE datanode1 WITH (TYPE='datanode',HOST='node1',PORT=15432)"
psql -p 15432 -c "CREATE NODE coordinator1 WITH (TYPE='coordinator',HOST='node1',PORT=5432)"
psql -p 15432 -c "CREATE NODE coordinator2 WITH (TYPE='coordinator',HOST='node2',PORT=5432)"
psql -p 15432 -c "select pgxc_pool_reload()" # 重新加载
psql -p 15432 -c "select * from pgxc_node" # 查看结果

psql -p 5432 -c "CREATE NODE coordinator1 WITH (TYPE='coordinator',HOST='node1',PORT=5432)"
psql -p 5432 -c "CREATE NODE datanode1 WITH (TYPE='datanode',HOST='node1',PORT=15432)"
psql -p 5432 -c "CREATE NODE datanode2 WITH (TYPE='datanode',HOST='node2',PORT=15432)"
psql -p 5432 -c "select pgxc_pool_reload()" # 重新加载
psql -p 5432 -c "select * from pgxc_node" # 查看结果
```

###### 4.8 连接数据库
**随意选择一个节点机的IP地址进行连接**
``` ruby
[postgres@test1 ~]$  psql -h 192.168.180.47 -p 5432 -U postgres
psql (PGXL 10r1.1, based on PG 10.6 (Postgres-XL 10r1.1))
Type "help" for help.

postgres=#

```

---


**启动集群的顺序:**

1. gtm
2. gtm_proxy
3. datanode
4. coordinator


**关闭集群的顺序:**

1. coordinator
2. datanode
3. gtm_proxy
4. gtm
