# Fucto Systemctl 部署文档

本文档介绍如何使用 systemctl 将 Fucto 项目部署为系统服务。

## 系统要求

- Linux 系统（支持 systemd，如 Ubuntu 16.04+, CentOS 7+, Debian 8+）
- Python 3.10+
- Root 或 sudo 权限

## 部署步骤

### 1. 准备部署目录

```bash
# 创建应用目录
sudo mkdir -p /root/app/fucto
cd /root/app/fucto
```

### 2. 上传项目文件

将以下文件上传到 `/root/app/fucto/` 目录：
- `openai_api_server.py`
- `requirements.txt`
- `cookies.txt` （需要手动创建）
- `cto2api.service`

### 3. 安装 Python 依赖

```bash
# 安装系统依赖
sudo apt update
sudo apt install -y python3 python3-pip python3-venv

# 创建虚拟环境
cd /root/app/fucto
sudo python3 -m venv venv
sudo source venv/bin/activate

# 安装 Python 依赖
sudo pip install -r requirements.txt
```

### 4. 配置 Cookie 文件

创建 `cookies.txt` 文件：
```bash
sudo nano /root/app/fucto/cookies.txt
```

文件内容格式：
```
# Cookie 示例，请替换为实际有效的 Cookie
__client=your_cookie_content_here
__client=another_cookie_here
```

### 5. 部署 Systemd 服务

```bash
# 复制服务文件到 systemd 目录
sudo cp cto2api.service /etc/systemd/system/

# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 启用服务（开机自启）
sudo systemctl enable cto2api.service

# 启动服务
sudo systemctl start cto2api.service
```

### 6. 验证服务状态

```bash
# 查看服务状态
sudo systemctl status cto2api.service

# 查看服务日志
sudo journalctl -u cto2api.service -f

# 测试 API 接口
curl -X GET http://localhost:8000/
curl -X GET http://localhost:8000/v1/models
```

## 服务管理命令

### 基本操作

```bash
# 启动服务
sudo systemctl start cto2api.service

# 停止服务
sudo systemctl stop cto2api.service

# 重启服务
sudo systemctl restart cto2api.service

# 查看服务状态
sudo systemctl status cto2api.service

# 禁用服务（取消开机自启）
sudo systemctl disable cto2api.service
```

### 日志管理

```bash
# 查看实时日志
sudo journalctl -u cto2api.service -f

# 查看最近 100 条日志
sudo journalctl -u cto2api.service -n 100

# 查看特定时间段的日志
sudo journalctl -u cto2api.service --since "2024-01-01" --until "2024-01-02"

# 清理旧日志
sudo journalctl --vacuum-time=7d
```

## 配置文件说明

### Systemd 服务配置 (`cto2api.service`)

```ini
[Unit]
Description=fucto  Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/app/fucto
ExecStart=/root/app/fucto/venv/bin/python -m uvicorn openai_api_server:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**配置说明：**
- `WorkingDirectory`: 应用程序的工作目录
- `ExecStart`: 启动命令，使用虚拟环境中的 Python
- `Restart=always`: 服务异常退出时自动重启
- `RestartSec=10`: 重启间隔时间（秒）
- `StandardOutput=journal`: 输出重定向到系统日志

## 常见问题处理

### 1. 服务启动失败

```bash
# 查看详细错误信息
sudo journalctl -u cto2api.service -n 50

# 检查 Python 环境
sudo /root/app/fucto/venv/bin/python --version

# 检查依赖是否完整
sudo /root/app/fucto/venv/bin/pip list
```

### 2. Cookie 文件问题

```bash
# 检查 Cookie 文件权限
sudo ls -la /root/app/fucto/cookies.txt

# 检查文件内容格式
sudo cat /root/app/fucto/cookies.txt
```

### 3. 端口被占用

```bash
# 查看端口占用情况
sudo netstat -tlnp | grep 8000
# 或使用
sudo ss -tlnp | grep 8000

# 如需修改端口，编辑服务文件中的端口号
sudo nano /etc/systemd/system/cto2api.service
# 修改后重新加载配置
sudo systemctl daemon-reload
sudo systemctl restart cto2api.service
```

### 4. 权限问题

确保所有文件都有正确的权限：
```bash
# 设置正确的文件权限
sudo chown -R root:root /root/app/fucto
sudo chmod 644 /root/app/fucto/cookies.txt
sudo chmod 644 /root/app/fucto/openai_api_server.py
sudo chmod 644 /root/app/fucto/requirements.txt
```

## 监控和维护

### 1. 设置监控脚本

创建监控脚本 `/root/app/fucto/monitor.sh`：
```bash
#!/bin/bash
# 检查服务状态
if ! systemctl is-active --quiet cto2api.service; then
    echo "$(date): fucto service is down, attempting restart" >> /var/log/fucto-monitor.log
    systemctl restart cto2api.service
fi
```

设置定时任务：
```bash
# 添加到 crontab，每5分钟检查一次
echo "*/5 * * * * /root/app/fucto/monitor.sh" | sudo crontab -
```

### 2. 日志轮转配置

创建日志轮转配置 `/etc/logrotate.d/fucto`：
```
/var/log/fucto-monitor.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    create 644 root root
}
```

## 备份和恢复

### 1. 备份重要文件

```bash
# 创建备份脚本
#!/bin/bash
BACKUP_DIR="/backup/fucto/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# 备份配置文件
cp /root/app/fucto/cookies.txt $BACKUP_DIR/
cp /etc/systemd/system/cto2api.service $BACKUP_DIR/

# 压缩备份
tar -czf $BACKUP_DIR.tar.gz $BACKUP_DIR
rm -rf $BACKUP_DIR
```

### 2. 恢复操作

```bash
# 停止服务
sudo systemctl stop cto2api.service

# 恢复文件
sudo cp backup/fucto/20240101/cookies.txt /root/app/fucto/
sudo cp backup/fucto/20240101/cto2api.service /etc/systemd/system/

# 重新加载并启动
sudo systemctl daemon-reload
sudo systemctl start cto2api.service
```

## 安全建议

1. **定期更新 Cookie**：定期检查和更新 `cookies.txt` 中的 Cookie
2. **监控日志**：定期检查服务日志，及时发现异常
3. **网络安全**：考虑配置防火墙，只允许必要的端口访问
4. **用户权限**：如非必要，建议创建专用用户运行服务，而非 root

---

**部署完成后，Fucto API 服务将在 `http://服务器IP:8000` 上运行，提供与 OpenAI 兼容的 API 接口。**