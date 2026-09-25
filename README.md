# Ansible 企业级服务器自动化运维实战 

## 📖 项目简介
本项目基于 RHEL 9 构建，使用 1 台控制节点与 2 台被管节点，通过 Ansible 的 Playbook 和 Roles，实现多台服务器的**系统标准化初始化**与**Web/DB 服务批量部署**。旨在解决手动运维效率低、易遗漏、环境不一致的痛点。

## 🏗️ 架构拓扑
- **控制节点 (server1)**: 负责运行 Ansible 剧本。
- **Web 服务器组 (web_servers)**: `server2`，部署 Nginx和Redis。
- **数据库服务器组 (db_servers)**: `server3`，部署 MySQL。

## 📂 目录结构
```text
ansible-auto-ops/
├── inventory.ini          # 主机清单 (包含 web_servers 和 db_servers 分组)
├── redis.yml              # redis启动文件
|—— nginx.yml              # nginx启动文件
├── deploy-web.yml         # Web 组部署剧本
├── deploy-db.yml          # DB 组部署剧本
├── roles/                 # 角色目录
│   ├── nginx/             # Nginx 角色 (vars,tasks, templates, handlers)
│   ├── redis/             # Redis 角色
│   └── mysql/             # MySQL 角色
└── README.md
```

---

## 🚀 阶段一：系统标准化初始化 

### 📌 功能说明
对所有被管节点进行统一的基础环境配置，确保所有服务器的初始状态一致。

1. **统一 Yum 源**：挂载本地yum源（RHEL9镜像文件）
2. **时间同步**：设置时区为 `Asia/Shanghai`
3. **关闭 SELinux**：修改 `/etc/selinux/config` 永久生效，并执行 `setenforce 0` 临时生效。
4. **配置ssh免密登录**： 修改/etc/hosts,并执行'ssh-keygen';'ssh-copy-id server'分发密钥
### 📸 运行效果截图
> <img width="746" height="307" alt="image" src="https://github.com/user-attachments/assets/d538e838-47fc-4516-a71f-43829e3d2542" />

---

## 🚀 阶段二：基于 Roles 的批量服务部署

### 📌 功能说明
利用 Ansible 的 Roles 模块化特性，针对不同分组批量部署服务。

- **Web 组 (server2)**：部署 Nginx 和 Redis。使用 `template` 模块动态渲染 `nginx.conf.j2`，通过 Jinja2 变量 `{{ nginx_port }}` 修改nginx访问端口，并修改防火墙（或在/etc/firewalld/service创建自定义服务文件）和selinux策略（若为permissive或disabled则不用）。配置文件变更后，通过 `notify` 触发 `handler` 自动重启 Nginx。
- **DB 组 (server3)**：部署 MySQL。完成安装、配置分发与自动启动。

### 💻 执行命令
```bash
# 批量部署 Web 组 (Nginx + Redis)
ansible-playbook -i inventory.ini deploy-web.yml

# 批量部署 DB 组 (MySQL)
ansible-playbook -i inventory.ini deploy-db.yml
```

### 📸 运行效果截图

#### 1. 批量部署 Nginx 与 Redis
| Playbook 执行过程 | 浏览器访问 Nginx 验证 |
| :---: | :---: |
| <img width="450"  alt="image" src="https://github.com/user-attachments/assets/a3c2dc43-ca2b-4ffc-8be5-0b0f160dbd1f" />| <img width="450" alt="image" src="https://github.com/user-attachments/assets/907355c3-0a2d-4a41-89dd-a5fef701e4c1" />|
| **说明**：Ansible 批量下发 Nginx 配置 | **说明**：服务成功响应， RHEL 的 Nginx 默认测试页这，证明安装、启动、防火墙策略都已生效 |

#### 2. Nginx 配置文件动态渲染验证
<img width="586" height="355" alt="image" src="https://github.com/user-attachments/assets/b4c188c1-7e43-4b0d-aeb7-40e168af69c6" />

*图3：登录 server2 查看 `/etc/nginx/nginx.conf`，`listen` 已被正确渲染为8080*

#### 3. 批量部署 MySQL (DB 组)
<img width="1037" height="487" alt="image" src="https://github.com/user-attachments/assets/c853db43-945d-475a-9f9e-bdf0c3cc67d7" />
*图4：在控制节点playbook运行成功，无报错*
<img width="1014" height="311" alt="image" src="https://github.com/user-attachments/assets/5511842c-a6d4-4e55-bdc6-61861a1711c2" />
*图5：MySQL 服务在 server3 上成功启动，`systemctl status mysqld` 显示 active (running)*

---
## 🚀 阶段三：Python 脚本与动态状态采集

### 📌 功能说明
在前三个阶段的基础上，本阶段实现了**节点状态的自动化采集与集中入库**。
利用 Ansible 批量向被管节点分发纯 Python 原生脚本（零第三方依赖），采集各节点的 CPU/内存状态，并通过 `delegate_to: localhost` 机制，将数据回传至控制节点本地的 SQLite 数据库落盘。

**核心优势**：
- **零依赖采集**：直接读取 Linux `/proc/stat` 和 `/proc/meminfo` 文件，无需安装 `psutil` 等第三方库，避免多版本 Python 环境导致的依赖冲突。
- **集中式存储**：被管节点保持纯净，无需安装数据库客户端或暴露数据库密码，仅由控制节点统一负责数据落库。

### 📂 目录结构
```text
ansible-auto-ops/
├── scripts/
│   ├── get_status.py      # 采集脚本（原生读取/proc）
│   └── write_db.py        # 写库脚本（SQLite）
├── collect-status.yml     # 采集与入库总剧本
└── README.md
```

### 🐍 核心代码解析

#### 1. 采集脚本 `scripts/get_status.py`
通过 Python 原生标准库读取 Linux 内核文件，实现零依赖采集：
```python
#!/usr/bin/env python3
import json, socket, time

def get_cpu_usage():
    with open('/proc/stat', 'r') as f: line1 = f.readline()
    parts1 = [float(x) for x in line1.split()[1:]]
    time.sleep(0.5)
    with open('/proc/stat', 'r') as f: line2 = f.readline()
    parts2 = [float(x) for x in line2.split()[1:]]
    idle_delta = parts2[3] - parts1[3]
    total_delta = sum(parts2) - sum(parts1)
    return round((1 - (idle_delta / total_delta)) * 100, 2) if total_delta > 0 else 0

def get_mem_usage():
    mem_total = mem_available = 0
    with open('/proc/meminfo', 'r') as f:
        for line in f:
            if 'MemTotal' in line: mem_total = float(line.split()[1])
            if 'MemAvailable' in line: mem_available = float(line.split()[1]); break
    return round((1 - (mem_available / mem_total)) * 100, 2) if mem_total > 0 else 0

print(json.dumps({
    "hostname": socket.gethostname(),
    "cpu_percent": get_cpu_usage(),
    "mem_percent": get_mem_usage()
}))
```

#### 2. 写库脚本 `scripts/write_db.py`
接收参数，写入控制节点本地的 SQLite 数据库：
```python
#!/usr/bin/env python3
import sqlite3, sys, json

data = json.loads(sys.argv[1])
conn = sqlite3.connect('status.db')
cursor = conn.cursor()
cursor.execute('''CREATE TABLE IF NOT EXISTS node_status (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    hostname TEXT, cpu REAL, mem REAL,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP)''')
cursor.execute('INSERT INTO node_status (hostname, cpu, mem) VALUES (?, ?, ?)',
    (data['hostname'], data['cpu_percent'], data['mem_percent']))
conn.commit()
conn.close()
print(f"Successfully recorded status for {data['hostname']}")
```

#### 3. 执行剧本 `collect-status.yml`
利用 `delegate_to: localhost` 在控制节点本地执行写库动作：
```yaml
---
- name: 批量采集节点状态并入库
  hosts: all
  gather_facts: no
  tasks:
    - name: 分发采集脚本
      ansible.builtin.copy:
        src: scripts/get_status.py
        dest: /tmp/get_status.py
        mode: '0755'

    - name: 执行采集脚本获取状态
      ansible.builtin.command: python3 /tmp/get_status.py
      register: status_result

    - name: 将采集结果写入控制节点本地数据库
      ansible.builtin.command: python3 scripts/write_db.py '{{ status_result.stdout }}'
      delegate_to: localhost
      become: no
```

### 💻 执行与验证
```bash
# 执行采集剧本
ansible-playbook collect-status.yml

# 验证数据库落盘结果
sqlite3 status.db "SELECT * FROM node_status;"
```

### 📸 运行效果截图

<img width="1035" height="397" alt="image" src="https://github.com/user-attachments/assets/fc703dca-8ca7-4ba4-b761-aae8d0007523" />
*图1：Ansible 批量执行采集剧本，PLAY RECAP 显示 ok=3 changed=3 failed=0*

<img width="752" height="88" alt="image" src="https://github.com/user-attachments/assets/baf675d5-2da9-4fc4-b0aa-7a9a53cbd21f" />
*图2：控制节点执行 SQLite 查询，成功获取 server2 与 server3 的 CPU/内存数据及时间戳*

### 🕳️ 踩坑记录与优化
- **问题 1**：初次使用 `psutil` 库采集，在多版本 Python 的 RHEL 9 环境中报错 `module 'psutil' has no attribute 'cpu_percent'`。
- **优化方案**：放弃第三方依赖，直接使用 Python 原生读取 `/proc/stat` 和 `/proc/meminfo`，实现“零依赖”采集，大幅提升了脚本的跨环境兼容性和稳定性。
- **问题 2**：使用了错误的 `ansible.builtin.local_action` 模块以及多余的 `-C` 参数，导致剧本执行失败。
- **优化方案**：改用 `ansible.builtin.command` 配合 `delegate_to: localhost` 关键字，实现了规范的本地任务委托机制。

### 
**“为什么用 SQLite 而不是 MySQL？为什么不直接在远程节点写入数据库？”**
> “在这个自动化运维项目中，我选择了 SQLite 作为轻量级存储。因为 SQLite 是嵌入式文件数据库，无需额外部署服务或开放端口，数据全部内敛在控制节点本地，极大地保证了安全性。同时，我通过 `delegate_to: localhost` 机制，避免了在被管节点安装数据库客户端或暴露数据库密码的风险。在生产环境中，如果数据量大，我会将其替换为 Prometheus 或 MySQL 集群。”

## ⚠️ 安全说明
为保证服务器安全，上传至 GitHub 的代码已进行脱敏处理：
1. `inventory.ini` 中的公网 IP 已替换为内网 IP 或占位符。
2. 所有截图均隐去了真实的公网 IP 和敏感信息。
3. 项目不包含任何 SSH 私钥及明文密码。
## 🕳️ 踩坑记录：Ansible 批量重启导致节点卡死（Emergency Mode）

### 📌 问题背景
在执行ansible批量禁止selinux时，包含 `reboot` 任务的 Ansible Playbook 后，`server2` 和 `server3` 两台节点机器重启失败，Xshell 连接超时。在 VMware 控制台查看，发现系统卡在启动界面，并最终进入 `Emergency Mode`（紧急模式），提示需要输入 root 密码维护。
### 📸 运行效果截图

*图1：Ansible运行剧本报错，试图重连，网络不通*
<img width="1031" height="534" alt="file 2" src="https://github.com/user-attachments/assets/da979c82-6bad-4bf6-9299-28271793269f" />
*图2：节点重启找不到磁盘*
<img width="1036" height="605" alt="file" src="https://github.com/user-attachments/assets/209b3242-c1b2-46a8-a866-62fbaaade6e9" />
*图3：节点重启卡死，进入 Emergency Mode*
<img width="1000" height="757" alt="file1" src="https://github.com/user-attachments/assets/56c2b0ed-f61f-4783-9ca0-be96a05a2160" />




### 🔍 排查过程
1. **观察报错信息**：VMware 控制台显示 `A start job is running for /dev/disk/by-uuid/10150ca7...`，随后报错 `Dependency failed for /boot`。
2. **进入紧急模式**：输入 root 密码进入救援终端。
3. **定位问题**：执行 `cat /etc/fstab` 发现挂载配置中依然使用的是**旧的 UUID**。
4. **验证真实磁盘**：执行 `blkid` 查看当前实际磁盘，发现由于此前克隆/重建过虚拟机，新磁盘的 UUID 与 `/etc/fstab` 里的旧 UUID 不一致，导致系统开机时找不到分区而卡死。

### 🛠️ 解决步骤

**1. 重挂载根目录为可写（救援模式默认只读）**
```bash
mount -o remount,rw /
```

**2. 查看当前真实的磁盘 UUID**
```bash
blkid
```
*记录下 `/boot` 和 `/boot/efi` 对应的新 UUID。*

**3. 修改 fstab 文件，替换旧 UUID**
```bash
vi /etc/fstab
```
*将 `UUID=10150ca7...` 和 `UUID=A4CD-FB83...` 替换为 `blkid` 查询到的新 UUID。*
*⚠️ 注意：千万不要盲目注释掉 `/boot` 和 `/boot/efi` 两行，这会导致系统彻底无法引导！*

**4. 保存并重启**
按 `Esc`，输入 `:wq` 保存退出，然后重启系统。
```bash
reboot
```

### 💡 复盘
- **不要盲目操作**：遇到挂载失败，第一反应应该是 `blkid` 验证真实磁盘状态，而不是直接删掉 `/etc/fstab` 里的行。盲目注释 `/boot` 会导致 `grub rescue>` 彻底黑屏。
- **虚拟机克隆陷阱**：克隆虚拟机后，除了修复 `fstab`，还应该清理 `/etc/machine-id` 和网卡持久化规则，避免网络冲突。
- **健壮性优化**：在生产环境中，对于非核心挂载点，建议在 `/etc/fstab` 中添加 `nofail` 选项（如 `defaults,nofail`），确保单盘故障时系统依然能正常开机。
- **其他方法**：将/etc/selinux/conf中“SELINUX=enforcing”设为permissive或disablesd,分发此文件，注意生产环境中selinux的使用
