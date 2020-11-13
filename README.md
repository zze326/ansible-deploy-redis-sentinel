# Ansible 部署 Redis 哨兵集群
> 下面所有操作在 Ubuntu 18.04 中进行。

此 Ansible 的功能为构建 Redis 哨兵集群，会在每个目标主机中启动一个 Redis 实例和哨兵实例，下面演示构建一个 3 节点 Redis 哨兵集群。

---

1、准备一台管理机执行如下操作安装 Ansible：

```bash
$ sudo apt update
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository --yes  ppa:ansible/ansible:2.7.6
$ sudo apt update
$ sudo apt-get install ansible
```

2、 取消 Ansible 检查 Key ：

```bash
$ vim /etc/ansible/ansible.cfg
# 取消此行注释
host_key_checking = False
```

3、克隆 Ansible Project：

```bash
$ git clone https://github.com/zze326/ansible-deploy-redis-sentinel.git
```

4、配置主机清单：

```bash
$ cd ansible-deploy-redis-sentinel/
$ vim hosts.yml
all:
  vars:
    # SSH 用户
    ansible_user: zze
    # SSH 密码
    ansible_ssh_pass: root1234
    # sudo 提权密码
    ansible_sudo_pass: root1234
    # Redis 编译好的二进制文件目录
    # 编译好的 Redis 二进制文件目录，这里我提供 5.0.10 的下载链接：
    # 	https://pan.baidu.com/s/1R5nDmpUqozVyu2hOD8kIeQ 提取码：kl7f
    # 如果你需要使用其它版本的 Redis，可通过 http://download.redis.io/releases/ 下载源码包进行编译，默认 make install 后的二进制包放在了 /usr/local/bin 下，所以将 /usr/local/bin/redis-* 拷贝到下面 bin_dir 指定的目录即可 
    bin_dir: /opt/redis_bin
    # 安装目录
    install_dir: /opt/apps
    # Redis 实例端口
    instance_port: 6379
    # Redis 哨兵端口
    sentinel_port: 26379
    # Redis 密码
    password: 123
  hosts:
    10.0.1.111:
    10.0.1.112:
    10.0.1.113:
      master: yes # 标识 10.0.1.113 为初始 Master
```

> 上述配置会在 `10.0.1.111` 、`10.0.1.112` 和 `10.0.1.113` 中各安装 1 个 Redis 实例和 1 个哨兵实例，Redis 实例占用 `instance_port` 指定的端口，哨兵实例占用 `sentinel_port` 指定的端口。

5、执行 Playbook，执行完成后输出如下：

```bash
$ ansible-playbook -i hosts.yml run.yml
...
TASK [start_service : 配置主从关系] **************************************************************************************************************************************************************************************************
skipping: [10.0.1.113]
changed: [10.0.1.111]
changed: [10.0.1.112]

TASK [start_service : 启动哨兵实例] **************************************************************************************************************************************************************************************************
ok: [10.0.1.111]
ok: [10.0.1.112]
ok: [10.0.1.113]

PLAY RECAP *********************************************************************************************************************************************************************************************************************
10.0.1.111                 : ok=21   changed=16   unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
10.0.1.112                 : ok=19   changed=16   unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
10.0.1.113                 : ok=18   changed=15   unreachable=0    failed=0    skipped=2    rescued=0    ignored=0 
```

6、检查复制关系：

```bash
$ redis-cli -a 123 -h 10.0.1.111 -p 6379 info replication
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
# Replication
role:slave
master_host:10.0.1.113
master_port:6379
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_repl_offset:28627
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:f01b45687ad4f3dbab11f5d33890393a1e2ca7fd
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:28627
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:28627

$ redis-cli -a 123 -h 10.0.1.112 -p 6379 info replication
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
# Replication
role:slave
master_host:10.0.1.113
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:35235
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:f01b45687ad4f3dbab11f5d33890393a1e2ca7fd
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:35235
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:35235
```
