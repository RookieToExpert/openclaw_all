# dcluster-ansible.md - D 集群物理机 Ansible 命令模板

## 1. 手动进入跳板机流程

```text
本地 OpenClaw exec
  → ssh 堡垒机 10.140.3.216:5906
  → JumpServer 中输入 220.33
  → sudo su sensetime
  → 执行 ansible 单 IP
```

登录：

```bash
ssh -p 5906 test6@10.140.3.216
```

到 JumpServer `Opt>` 后输入：

```text
220.33
```

到跳板机后：

```bash
sudo su sensetime
```

正确执行位置：

```text
sensetime@host-10-140-220-33
```

---

## 2. expect 自动进入跳板机模板

```expect
set timeout 30
spawn ssh -p 5906 test6@10.140.3.216

expect {
    "password:" { send "6RV&QnwPrq" }
    timeout { puts "bastion login timeout"; exit 1 }
}

expect {
    "Opt>" { send "220.33" }
    timeout { puts "JumpServer Opt prompt timeout"; exit 1 }
}

expect {
    -re {test6@host-10-140-220-33.*[$#]} { send "sudo su sensetime" }
    timeout { puts "jump host prompt timeout"; exit 1 }
}

set timeout 120
expect {
    -re {sensetime@host-10-140-220-33.*[$#]} {
        # 到达这里后才允许执行 ansible 单 IP 命令
    }
    timeout { puts "switch to sensetime timeout"; exit 1 }
}
```

---

## 3. 只读单机查询模板

以下命令都必须在跳板机 sensetime 用户下执行。

```bash
ansible all -i '<目标IP>,' -m shell -a 'hostname'
ansible all -i '<目标IP>,' -m shell -a 'date'
ansible all -i '<目标IP>,' -m shell -a 'ls -la ~sensetime/'
ansible all -i '<目标IP>,' -m shell -a 'df -h'
ansible all -i '<目标IP>,' -m shell -a 'free -h'
ansible all -i '<目标IP>,' -m shell -a 'mx-smi'
ansible all -i '<目标IP>,' -m shell -a 'ip addr'
ansible all -i '<目标IP>,' -m shell -a 'ip route'
ansible all -i '<目标IP>,' -m shell -a 'cat /etc/resolv.conf'
ansible all -i '<目标IP>,' -m shell -a 'ps -ef | head -50'
ansible all -i '<目标IP>,' -m shell -a 'tail -n 100 <log-path>'
```

### 3.1 宿主机训练网 / 数据网只读互通检查模板

以下命令都必须在跳板机 sensetime 用户下执行。

查询目标机器的全局 IPv4 网卡，过滤 Kubernetes 虚拟地址噪音：

```bash
ansible all -i '<目标IP>,' -m shell -a "ip -br -4 addr scope global | grep -v '^kube-ipvs0'"
```

按常见训练网 / 数据网接口名缩小候选范围：

```bash
ansible all -i '<目标IP>,' -m shell -a "ip -br -4 addr scope global | egrep '^(eth|bond|data)' | grep -v '^kube-ipvs0'"
```

从源机器指定源 IP ping 目标同网段 IP：

```bash
ansible all -i '<源机器管理IP>,' -m shell -a "ping -I '<源训练网IP>' -c 3 -W 1 '<目标训练网IP>'"
```

双向验证时，在目标机器反向执行：

```bash
ansible all -i '<目标机器管理IP>,' -m shell -a "ping -I '<目标训练网IP>' -c 3 -W 1 '<源训练网IP>'"
```

结果记录格式：

```text
源机器管理 IP:
目标机器管理 IP:
源训练网 IP:
目标训练网 IP:
方向:
丢包率:
结论:
```

只读边界：

* `ip -br addr`、`ip route`、`ping` 属于只读检查。
* 不修改网卡、路由、iptables、服务或系统配置。
* 不自动扩大到更多机器；扩大范围前先说明节点范围和原因。

---

## 4. 单机命令转义规则

普通命令：

```bash
ansible all -i '10.12.x.x,' -m shell -a 'hostname'
```

命令本身包含单引号时，优先改用双引号并转义 `$`：

```bash
ansible all -i '10.12.x.x,' -m shell -a "awk '{print \$1}' /etc/hosts"
```

复杂命令、长命令、多行命令，优先让用户确认后写临时脚本。
创建临时脚本属于写操作。

---

## 5. 批量 Ansible

批量 Ansible 也必须在跳板机 sensetime 用户下执行。

```bash
export ANSIBLE_CONFIG=/home/sensetime/ansible/ansible.cfg
ansible d-cluster -i /home/sensetime/ansible/d-cluster.ini -m shell -a '<命令>'
ansible-playbook -i /home/sensetime/ansible/d-cluster.ini playbook.yml
```

批量写操作必须先确认，优先先抽一台验证，再扩大范围。
