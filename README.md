## 🕳️ 踩坑记录：Ansible 批量重启导致节点卡死（Emergency Mode）

### 📌 问题背景
在执行包含 `reboot` 任务的 Ansible Playbook 后，`server2` 和 `server3` 两台节点机器重启失败，Xshell 连接超时。在 VMware 控制台查看，发现系统卡在启动界面，并最终进入 `Emergency Mode`（紧急模式），提示需要输入 root 密码维护。
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
- **面试回答模板**：“我曾遇到过因克隆虚拟机导致 UUID 变更，Ansible 批量重启后节点卡死在 Emergency Mode。我通过 `blkid` 对比 `/etc/fstab` 定位到挂载失败，然后重挂载根目录为读写，更新 UUID 并添加 `nofail` 选项，成功恢复。这让我深刻理解了 Linux 启动流程与磁盘挂载机制。”
