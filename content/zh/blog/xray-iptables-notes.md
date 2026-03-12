---
title: "Xray + iptables 安装与配置笔记"
date: 2026-03-12T04:50:00+08:00
summary: 记录 Xray 安装、config.json 配置与 iptables 透明代理的完整步骤——涵盖强制代理模式、健康检查回退机制与 systemd 开机自启配置。
description: 一份可直接参考的 Xray 与 iptables 安装和分流配置笔记。
tags:
  - Xray
  - iptables
  - Linux
  - 代理
  - 网络
---

> 这是一份**适合分享**的通用笔记，不包含任何真实节点、UUID、公钥、密码或其他私密信息。
>
> 所有命令、脚本、配置都以代码块形式给出，方便直接复制到 Typora 或其他 Markdown 阅读器中查看。
>
> **提醒：** 本文中的 `iptables`、`systemctl`、写入 `/usr/local/bin/` 与 `/etc/systemd/system/` 等操作都需要 `sudo` 权限。正式操作前，建议先确认自己可以通过 SSH、控制台或另一种方式重新接入机器，避免误操作后把自己挡在外面。
>
> 如果你只是想照着做，建议直接从 <a href="#section-5" class="appendix-back-btn">↓ 第 5 节</a> 开始往下看。

## 1. 这套架构是在做什么？

这份笔记讲的是一种很常见的本机流量分发架构：

- **iptables** 负责把本机流量“导流”到指定入口
- **Xray** 负责接住这些流量、识别目标、再决定应该走哪条出口

你可以把它理解成：

- `iptables` 是**交通警察**
- `Xray` 是**分流调度中心**

`iptables` 本身不会理解“这个流量是不是 OpenAI / ChatGPT / Codex”，它更擅长做的是：

- 把流量送到某个端口
- 放行某些地址段
- 拒绝某些协议或端口

而 `Xray` 更擅长做的是：

- 判断流量应该走哪个出口
- 区分不同域名或目标
- 提供透明代理、SOCKS、HTTP 等入口

## 2. 这套流量分发流程是怎么工作的？

### 2.1 流程分解

1. **应用程序发起请求**  
   例如浏览器、`curl`、OpenClaw、其他命令行工具。
2. **iptables 拦截本机发出的 TCP 流量**  
   如果目标是内网或回环地址，就直接放行；如果目标是需要代理的外部流量，就把它重定向到本地 `12345`。
3. **Xray 的透明代理入口接住这部分流量**  
   这里通常就是 `dokodemo-door` inbound。
4. **Xray 根据 routing 规则做判断**  
   OpenAI / ChatGPT / Codex 相关流量走节点 1；其他普通流量走节点 2 或直连；内网流量走 `direct`。
5. **最终由对应出口把流量送出去**  
   所以真正决定“走哪条线”的是 Xray，真正决定“哪些流量先送进 Xray”的是 iptables。

### 2.2 一句话理解

```text
iptables 负责把流量送进 Xray，Xray 负责把流量分发到正确的出口。
```

## 3. 结构图

![Xray + iptables 架构图](/media/blog/xray-iptables-architecture.jpg)

下面这张图可以帮助你快速理解组件之间的关系。

<p class="figure-caption"><strong>Xray 与 iptables 的本机流量分发关系示意图</strong></p>

## 4. 使用说明

这份笔记按下面的顺序组织：

5. **Xray 安装与配置的完整流程**
6. **iptables 配置流程**
   - 情况 1：强制透明代理，不做条件判断
   - 情况 2：检测 Xray 健康状态，避免完全断网
7. **参考与排障命令**
8. **附录**
   - 附录 A：单出口配置模板
   - 附录 B：双出口配置模板
   - 附录 C：iptables Bash 命令集中区

如果你只想抓主线，可以按下面顺序做：

1. 安装 Xray
2. 写好并测试 `config.json`
3. 确认 `12345 / 10808 / 10809` 端口监听正常
4. 再选择 `iptables` 方案
   - 要么用“强制透明代理”
   - 要么用“健康检查 + 自动回退”
5. 最后再做 `systemd` 开机自启

## 5. Xray 安装与配置流程 {#section-5}

### 5.1 Xray 是干嘛的？

在这套架构里，Xray 负责：

- 提供透明代理入口，例如 `12345`
- 提供本地代理入口，例如 `10808 / 10809`
- 根据路由规则判断流量该走哪条出口

在本文场景里，Xray 主要承担的是：

```text
接住 iptables 导过来的流量，然后根据 routing 规则做分流。
```

本节先给出一个**不中断的完整流程**。

### 5.2 安装 Xray

```bash
# 安装 Xray（必须使用 sudo，且整条命令保持单行执行）
sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

> 注意：不要把 `@ install` 换到下一行，否则会触发 `install` 命令报错。

### 5.3 查看是否安装成功

```bash
# 查看 Xray 版本
/usr/local/bin/xray version
```

**预期输出示例：**

```text
Xray 1.x.x (Xray, Penetrates Everything.)
```

如果提示 `No such file or directory` 或 `command not found`，说明安装还没成功。

### 5.4 确认配置文件位置

```bash
# 查看默认配置文件是否存在
ls -l /usr/local/etc/xray/config.json
```

**预期输出示例：**

```text
-rw-r--r-- 1 root root 1234 ... /usr/local/etc/xray/config.json
```

如果文件不存在，也没关系，下一步可以手动创建。

### 5.5 使用 nano 编辑配置文件 {#section-55}

```bash
# 用 nano 编辑 Xray 配置文件
sudo nano /usr/local/etc/xray/config.json
```

这里的”模板”是用于生成 `config.json` 的基础骨架，决定不同流量最终走哪条出口。

如果你的需求是：

- OpenAI / ChatGPT / Codex / GPT 相关流量走**节点 1**
- 其他流量**直连**

那么请使用：<a href="#appendix-a" class="appendix-back-btn">↓ 附录 A：单出口配置模板</a>

如果你的需求是：

- OpenAI / ChatGPT / Codex / GPT 相关流量走**节点 1**
- 其他流量走**节点 2**

那么请使用：<a href="#appendix-b" class="appendix-back-btn">↓ 附录 B：双出口配置模板</a>

> 实用提醒：如果你不想手工逐项填配置，可以把”代理节点链接（单个或多个）+ 你要使用的模板”发给商用模型，让它先生成一份可用 `config.json` 初稿，再回到本机自行核对与测试。为了降低敏感信息留痕风险，建议优先使用**临时聊天**或一次性会话来处理节点内容。

然后把你选好的附录模板内容完整复制进去，并把占位符替换成你自己的参数。

#### 5.5.1 nano 保存与退出方法

```text
Ctrl + O   -> 写入（保存）文件
Enter      -> 确认文件名
Ctrl + X   -> 退出 nano
```

如果你改错了，不想保存，直接：

```text
Ctrl + X
按 N
```

即可退出且不保存。

#### 5.5.2 参考文档

```text
https://www.nano-editor.org/docs.php
https://xtls.github.io/config/
```

### 5.6 检查配置语法

```bash
# 检查配置是否可被 Xray 正常解析
sudo /usr/local/bin/xray run -test -config /usr/local/etc/xray/config.json
```

**预期输出示例：**

```text
Xray 1.x.x ...
Configuration OK.
```

如果这里报错，不要继续启动服务，先回到 `nano` 里修配置。

### 5.7 启用并启动 Xray

```bash
# 重载 systemd 配置
sudo systemctl daemon-reload

# 设置开机自启
sudo systemctl enable xray

# 启动或重启 Xray
sudo systemctl restart xray

# 查看 Xray 当前状态
sudo systemctl status xray --no-pager -n 50
```

**预期输出示例：**

```text
● xray.service - Xray Service
     Loaded: loaded (...)
     Active: active (running)
```

如果这里看到的是：

```text
Active: failed
```

说明配置或启动参数还有问题，需要先看日志排查。

### 5.8 检查监听端口是否正常

```bash
# 查看透明代理、SOCKS、HTTP 入口端口是否已监听
ss -ltnup | grep -E '12345|10808|10809'
```

**预期输出示例：**

```text
tcp   LISTEN 0 4096 0.0.0.0:12345    0.0.0.0:*
tcp   LISTEN 0 4096 127.0.0.1:10808  0.0.0.0:*
tcp   LISTEN 0 4096 127.0.0.1:10809  0.0.0.0:*
udp   UNCONN 0 0    127.0.0.1:10808  0.0.0.0:*
```

你至少应该看到：

- `12345`
- `10808`
- `10809`

都已经出现。到这里，Xray 本身的安装与配置流程就完成了。

## 6. iptables 配置流程

### 6.1 iptables 是干嘛的？

在这套架构里，iptables 负责：

- 拦截本机发出的流量
- 决定哪些目标地址直接放行
- 把需要代理的流量送到 Xray 的透明代理入口

换句话说：

```text
Xray 决定“流量往哪走”，iptables 决定“哪些流量先送进 Xray”。
```

这一节开始进入透明代理规则。这里分成两种情况。

## 7. 情况 1：你需要”强制透明代理”，不做条件判断

如果你的需求是：

- 所有外部 TCP 都必须被导向 Xray
- Xray 异常时你也接受网络可能失败
- 你不需要自动回退到直连

那么你可以使用下面这套**强制透明代理**规则。

### 7.1 用 nano 写入脚本

先创建脚本文件：

```bash
sudo nano /usr/local/bin/xray-iptables-apply.sh
```

把下面内容完整贴进去：

```bash
#!/usr/bin/env bash
set -e

# 创建 XRAY 链，已存在则忽略
sudo iptables -t nat -N XRAY 2>/dev/null || true

# 拒绝 UDP 443，避免 QUIC 绕过 TCP 分流（先去重再添加）
while sudo iptables -C OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null; do
  sudo iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable
done
sudo iptables -A OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable

# 确保 OUTPUT 到 XRAY 只有一条跳转规则（先去重再添加）
while sudo iptables -t nat -C OUTPUT -p tcp -j XRAY 2>/dev/null; do
  sudo iptables -t nat -D OUTPUT -p tcp -j XRAY
done
sudo iptables -t nat -A OUTPUT -p tcp -j XRAY

# 先清空 XRAY 链，避免重复执行时规则不断叠加
sudo iptables -t nat -F XRAY

# 本地回环地址不走代理
sudo iptables -t nat -A XRAY -d 127.0.0.0/8 -j RETURN

# 10.0.0.0/8 内网不走代理
sudo iptables -t nat -A XRAY -d 10.0.0.0/8 -j RETURN

# 172.16.0.0/12 内网不走代理
sudo iptables -t nat -A XRAY -d 172.16.0.0/12 -j RETURN

# 192.168.0.0/16 内网不走代理
sudo iptables -t nat -A XRAY -d 192.168.0.0/16 -j RETURN

# 组播地址不走代理
sudo iptables -t nat -A XRAY -d 224.0.0.0/4 -j RETURN

# 保留地址段不走代理
sudo iptables -t nat -A XRAY -d 240.0.0.0/4 -j RETURN

# 跳过特定 UID，避免某些系统账户被透明代理
sudo iptables -t nat -A XRAY -m owner --uid-owner 65534 -j RETURN

# 其余 TCP 流量全部重定向到 Xray 的透明代理入口 12345
sudo iptables -t nat -A XRAY -p tcp -j REDIRECT --to-ports 12345
```

### 7.2 给脚本赋权并执行

```bash
# 赋予执行权限
sudo chmod +x /usr/local/bin/xray-iptables-apply.sh

# 执行脚本
sudo /usr/local/bin/xray-iptables-apply.sh
```

### 7.3 执行写入脚本后应该看到什么

```bash
# 导出当前 iptables 规则
sudo iptables-save
```

**预期输出示例：**

```text
*filter
-A OUTPUT -p udp -m udp --dport 443 -j REJECT --reject-with icmp-port-unreachable
COMMIT
*nat
:XRAY - [0:0]
-A OUTPUT -p tcp -j XRAY
-A XRAY -d 127.0.0.0/8 -j RETURN
...
-A XRAY -p tcp -j REDIRECT --to-ports 12345
COMMIT
```

重点看这几项是否存在：

- `-A OUTPUT -p tcp -j XRAY`
- `-A XRAY -p tcp -j REDIRECT --to-ports 12345`
- `-A OUTPUT -p udp --dport 443 -j REJECT ...`

### 7.4 如何验证透明代理是否真的生效

```bash
# 1) 确认 Xray 端口还在监听
ss -ltnup | grep -E '12345|10808|10809'

# 2) 查看当前出口 IP（走 SOCKS 代理）
curl -x socks5h://127.0.0.1:10808 https://api.ipify.org

# 3) 访问一个你明确希望走代理的站点
curl -I https://api.openai.com --connect-timeout 10

# 4) 观察 Xray 日志
journalctl -u xray -n 100 --no-pager
```

如果你发现：

- `iptables-save` 里规则都在
- 但访问外网还是完全不通

那么优先排查：

1. `xray.service` 是否真的正常运行
2. `12345` 端口是否还在监听
3. routing 规则是否把目标错误地送进了不可用出口
4. 目标机器本身是否能正常直连外网

### 7.5 teardown 脚本

```bash
sudo nano /usr/local/bin/xray-iptables-teardown.sh
```

写入以下内容：

```bash
#!/usr/bin/env bash
set -e

# 删除 UDP 443 拒绝规则
sudo iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null || true

# 删除 OUTPUT 到 XRAY 的跳转
sudo iptables -t nat -D OUTPUT -p tcp -j XRAY || true

# 清空 XRAY 链中的规则
sudo iptables -t nat -F XRAY || true

# 删除 XRAY 自定义链
sudo iptables -t nat -X XRAY || true
```

保存后赋权：

```bash
sudo chmod +x /usr/local/bin/xray-iptables-teardown.sh
```

需要恢复直连时执行：

```bash
sudo /usr/local/bin/xray-iptables-teardown.sh
```

### 7.6 teardown 后应该看到什么

```bash
# 再次导出当前 iptables 规则
sudo iptables-save
```

**预期输出示例：**

```text
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
COMMIT
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
COMMIT
```

也就是说，之前的 `XRAY` 链和相关跳转应该已经不存在了。

### 7.7 做成开机自启（适用于强制透明代理方案）

如果你选择的是“情况 1：强制透明代理”，那么仅仅手动执行一次脚本还不够。因为机器重启后，iptables 规则通常不会自动恢复。

比较直接的做法是：

- Xray 服务开机启动
- 再额外创建一个 `systemd` 服务，在开机后执行 `xray-iptables-apply.sh`

#### 7.7.1 创建 systemd 服务

```bash
sudo nano /etc/systemd/system/xray-iptables-apply.service
```

写入以下内容：

```ini
[Unit]
Description=Apply Xray iptables redirect rules
After=network-online.target xray.service
Wants=network-online.target
Requires=xray.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/xray-iptables-apply.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

#### 7.7.2 启用开机自启

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 确保 Xray 本身开机自启
sudo systemctl enable --now xray

# 启用 iptables 规则服务
sudo systemctl enable --now xray-iptables-apply.service
```

#### 7.7.3 检查服务状态

```bash
sudo systemctl status xray --no-pager
sudo systemctl status xray-iptables-apply.service --no-pager
```

如果一切正常，你应该看到：

```text
xray.service: active (running)
xray-iptables-apply.service: active (exited)
```

这里 `active (exited)` 是正常现象，因为这个服务只是执行一次脚本，不是长期驻留进程。

#### 7.7.4 如果要取消开机自启

```bash
sudo systemctl disable --now xray-iptables-apply.service
```

然后再执行 teardown 脚本恢复直连：

```bash
sudo /usr/local/bin/xray-iptables-teardown.sh
```

## 8. 情况 2：你需要检测 Xray 是否工作，并在异常时防止完全断网

如果你的需求是：

- Xray 正常时，流量照常被透明代理
- Xray 异常时，不想让整机外网 TCP 全挂掉
- 希望通过脚本自动切换 iptables 状态

那么你应该使用：

```text
健康检查脚本 + 两套 iptables 状态
```

也就是：

- Xray 正常 -> 装代理规则
- Xray 异常 -> 卸载代理规则，切回直连

### 8.1 用 nano 写入健康检查脚本

```bash
sudo nano /usr/local/bin/xray-iptables-healthcheck.sh
```

写入以下内容：

```bash
#!/usr/bin/env bash
set -e

XRAY_CHAIN="XRAY"
XRAY_PORT="12345"

xray_is_healthy() {
  # 先判断 xray 服务是否处于 active
  if systemctl is-active --quiet xray; then

    # 再判断 12345 端口是否确实在监听
    if ss -ltn | awk '{print $4}' | grep -qE '(^|:)12345$'; then

      # 两者都满足，则认为 Xray 健康
      return 0
    fi
  fi

  # 否则判定为不健康
  return 1
}

apply_proxy_rules() {
  # 创建 XRAY 链，已存在则忽略
  sudo iptables -t nat -N "$XRAY_CHAIN" 2>/dev/null || true

  # 先清空 XRAY 链，避免重复堆叠旧规则
  sudo iptables -t nat -F "$XRAY_CHAIN"

  # 确保 UDP 443 拒绝规则存在
  sudo iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null || true
  sudo iptables -A OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable

  # 确保 OUTPUT 中有跳转到 XRAY 链的入口
  while sudo iptables -t nat -C OUTPUT -p tcp -j "$XRAY_CHAIN" 2>/dev/null; do
    sudo iptables -t nat -D OUTPUT -p tcp -j "$XRAY_CHAIN"
  done
  sudo iptables -t nat -A OUTPUT -p tcp -j "$XRAY_CHAIN"

  # 本地回环地址直连
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 127.0.0.0/8 -j RETURN

  # 10.0.0.0/8 内网直连
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 10.0.0.0/8 -j RETURN

  # 172.16.0.0/12 内网直连
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 172.16.0.0/12 -j RETURN

  # 192.168.0.0/16 内网直连
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 192.168.0.0/16 -j RETURN

  # 组播地址直连
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 224.0.0.0/4 -j RETURN

  # 保留地址段直连
  sudo iptables -t nat -A "$XRAY_CHAIN" -d 240.0.0.0/4 -j RETURN

  # 跳过特定 UID
  sudo iptables -t nat -A "$XRAY_CHAIN" -m owner --uid-owner 65534 -j RETURN

  # 把其余 TCP 重定向到 Xray 透明代理入口
  sudo iptables -t nat -A "$XRAY_CHAIN" -p tcp -j REDIRECT --to-ports "$XRAY_PORT"
}

apply_direct_rules() {
  # 删除 OUTPUT 到 XRAY 的跳转，让 TCP 回到直连
  while sudo iptables -t nat -C OUTPUT -p tcp -j "$XRAY_CHAIN" 2>/dev/null; do
    sudo iptables -t nat -D OUTPUT -p tcp -j "$XRAY_CHAIN"
  done

  # 清空 XRAY 链
  sudo iptables -t nat -F "$XRAY_CHAIN" 2>/dev/null || true

  # Xray 不健康时，清理 UDP 443 REJECT，避免规则累积并恢复直连
  while sudo iptables -C OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null; do
    sudo iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable
  done
}

if xray_is_healthy; then
  echo "Xray healthy -> apply proxy rules"
  apply_proxy_rules
else
  echo "Xray unhealthy -> switch to direct mode"
  apply_direct_rules
fi
```

### 8.2 给健康检查脚本赋权并先手动测试一次

```bash
# 赋予执行权限
sudo chmod +x /usr/local/bin/xray-iptables-healthcheck.sh

# 先手动执行一次，确认脚本本身没有语法或权限问题
sudo /usr/local/bin/xray-iptables-healthcheck.sh
```

### 8.3 systemd 定时执行健康检查脚本

这样配置完成后，systemd 会每分钟执行一次健康检查脚本：

- 如果 Xray 正常，就维持透明代理规则
- 如果 Xray 异常，就尽量把规则切回直连

#### 8.3.1 创建 systemd service

```bash
sudo nano /etc/systemd/system/xray-iptables-healthcheck.service
```

```ini
[Unit]
Description=Xray iptables healthcheck
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/xray-iptables-healthcheck.sh
```

#### 8.3.2 创建 systemd timer

```bash
sudo nano /etc/systemd/system/xray-iptables-healthcheck.timer
```

```ini
[Unit]
Description=Run Xray iptables healthcheck every minute

[Timer]
OnBootSec=30s
OnUnitActiveSec=60s
Unit=xray-iptables-healthcheck.service

[Install]
WantedBy=timers.target
```

#### 8.3.3 启用 timer

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 设置定时器开机自启
sudo systemctl enable xray-iptables-healthcheck.timer

# 启动定时器
sudo systemctl start xray-iptables-healthcheck.timer

# 查看定时器是否已经生效
sudo systemctl list-timers --all | grep xray
```

**预期输出示例：**

```text
... xray-iptables-healthcheck.timer loaded active waiting ...
```

## 9. 参考与排障命令

### 9.1 Xray 服务控制命令

```bash
# 启动 Xray
sudo systemctl start xray

# 停止 Xray
sudo systemctl stop xray

# 重启 Xray
sudo systemctl restart xray

# 尝试重新加载配置（前提是服务支持 reload）
sudo systemctl reload xray

# 设置开机自启
sudo systemctl enable xray

# 取消开机自启
sudo systemctl disable xray

# 查看 Xray 服务状态
sudo systemctl status xray --no-pager -n 50
```

### 9.2 日志、端口与连接测试

```bash
# 查看最近 100 行 Xray 日志
journalctl -u xray -n 100 --no-pager

# 实时跟踪 Xray 日志
journalctl -u xray -f

# 查看关键端口监听状态
ss -ltnup | grep -E '12345|10808|10809'

# 导出当前 iptables 规则
sudo iptables-save

# 查看 nft 后端规则
sudo nft list ruleset

# 通过 SOCKS 查询出口 IP
curl -x socks5h://127.0.0.1:10808 https://api.ipify.org

# 通过 HTTP 查询出口 IP
curl -x http://127.0.0.1:10809 https://api.ipify.org
```

### 9.3 健康检查 timer 常用命令（启用 timer 后执行）

**1) 检查 xray 是否正常运行**

```bash
systemctl is-active xray
```

预期（正常）：

```text
active
```

**2) 检查 timer 是否在等待触发**

```bash
systemctl is-active xray-iptables-healthcheck.timer
```

预期（正常）：

```text
active
```

**3) 查看最近一次 healthcheck 是否成功执行**

```bash
systemctl status xray-iptables-healthcheck.service --no-pager -n 20
```

预期（正常）里应看到类似：

```text
code=exited, status=0/SUCCESS
```

**4) 检查透明代理跳转是否存在（xray 健康时）**

```bash
sudo iptables -t nat -S OUTPUT | grep -c -- "-A OUTPUT -p tcp -j XRAY"
```

预期（健康时）：

```text
1
```

**5) 检查 UDP 443 REJECT 条数（应避免重复）**

```bash
sudo iptables -S OUTPUT | grep -c -- "--dport 443 -j REJECT --reject-with icmp-port-unreachable"
```

预期（健康时）：

```text
1
```

**6) 检查 XRAY 链里是否有透明重定向**

```bash
sudo iptables -t nat -S XRAY | grep -c -- "--to-ports 12345"
```

预期（健康时）：

```text
1
```

**7) 端口监听检查**

```bash
ss -ltnup | grep -E '12345|10808|10809'
```

预期（正常）应至少包含 `:12345`、`:10808`、`:10809` 三个监听项。

如果出现这些结果，通常表示异常：

- `systemctl is-active xray` 返回 `inactive`
- healthcheck service 出现 `status=1/FAILURE`
- `jump/udp/redir` 计数不是预期值，例如重复大于 `1`

## 10. 最后检查清单

### 10.1 Xray 本体是否正常

```bash
sudo systemctl status xray --no-pager
ss -ltnup | grep -E '12345|10808|10809'
```

你应该至少看到：

- `xray.service` 是 `active (running)`
- `12345 / 10808 / 10809` 已监听

### 10.2 代理出口是否符合预期

```bash
curl -x socks5h://127.0.0.1:10808 https://api.ipify.org
curl -x http://127.0.0.1:10809 https://api.ipify.org
```

如果这里查到的 IP 不对，就要回头检查：

- 节点参数有没有填错
- routing 是否把流量送到了错误出口
- 节点本身是否可用

### 10.3 iptables 规则是否符合你的方案

```bash
sudo iptables-save
```

你需要确认自己现在到底是哪一种模式：

- **强制透明代理模式**：规则应该长期存在
- **健康检查回退模式**：规则会随 Xray 健康状态动态变化

### 10.4 开机自启是否符合预期

如果你使用的是强制透明代理方案，检查：

```bash
sudo systemctl status xray-iptables-apply.service --no-pager
```

如果你使用的是健康检查方案，检查：

```bash
sudo systemctl status xray-iptables-healthcheck.timer --no-pager
sudo systemctl list-timers --all | grep xray
```

### 10.5 出现异常时先看哪里

优先按这个顺序排查：

1. `sudo systemctl status xray --no-pager`
2. `journalctl -u xray -n 100 --no-pager`
3. `ss -ltnup | grep -E '12345|10808|10809'`
4. `sudo iptables-save`
5. `curl -x socks5h://127.0.0.1:10808 https://api.ipify.org`

## 11. 相关文件位置

当前工作区里和这份分享版笔记直接相关的文件有：

```text
tmp/xray-config/xray-single-exit-template.json
tmp/xray-config/xray-dual-exit-template.json
tmp/xray-config/Xray-iptables-notes.md
```

如果按绝对路径看：

```text
/home/claw1/.openclaw/workspace/tmp/xray-config/xray-single-exit-template.json
/home/claw1/.openclaw/workspace/tmp/xray-config/xray-dual-exit-template.json
/home/claw1/.openclaw/workspace/tmp/xray-config/Xray-iptables-notes.md
```

## 附录 A：单出口配置模板 {#appendix-a}

<a href="#section-55" class="appendix-back-btn">↑ 返回 §5.5 使用 nano 编辑配置文件</a>

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "redir-in",
      "listen": "0.0.0.0",
      "port": 12345,
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp",
        "followRedirect": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "streamSettings": {
        "sockopt": {
          "tproxy": "redirect"
        }
      }
    },
    {
      "tag": "socks-in",
      "listen": "127.0.0.1",
      "port": 10808,
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    },
    {
      "tag": "http-in",
      "listen": "127.0.0.1",
      "port": 10809,
      "protocol": "http",
      "settings": {}
    }
  ],
  "outbounds": [
    {
      "tag": "node1-openai",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "YOUR_NODE1_SERVER",
            "port": 443,
            "users": [
              {
                "id": "YOUR_UUID",
                "encryption": "none",
                "flow": "xtls-rprx-vision"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "serverName": "YOUR_SERVER_NAME",
          "fingerprint": "random",
          "publicKey": "YOUR_PUBLIC_KEY",
          "shortId": "",
          "spiderX": "/"
        }
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "block",
      "protocol": "blackhole",
      "settings": {}
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "ip": [
          "geoip:private",
          "192.168.2.0/24"
        ],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "domain": [
          "domain:openai.com",
          "domain:chatgpt.com",
          "domain:oaistatic.com",
          "domain:oaiusercontent.com",
          "domain:openaiapi-site.azureedge.net",
          "domain:auth.openai.com",
          "domain:platform.openai.com",
          "domain:api.openai.com",
          "domain:cdn.openai.com",
          "domain:chat.openai.com",
          "domain:ab.chatgpt.com",
          "domain:files.oaiusercontent.com",
          "domain:chatgpt.livekit.cloud",
          "domain:featuregates.org",
          "regexp:(^|\\.)codex\\.",
          "regexp:(^|\\.)gpt\\.",
          "regexp:(^|\\.)openai\\.",
          "regexp:(^|\\.)chatgpt\\.",
          "keyword:openai",
          "keyword:chatgpt",
          "keyword:codex",
          "keyword:gpt"
        ],
        "outboundTag": "node1-openai"
      },
      {
        "type": "field",
        "network": "tcp,udp",
        "outboundTag": "direct"
      }
    ]
  }
}
```

## 附录 B：双出口配置模板 {#appendix-b}

<a href="#section-55" class="appendix-back-btn">↑ 返回 §5.5 使用 nano 编辑配置文件</a>

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "redir-in",
      "listen": "0.0.0.0",
      "port": 12345,
      "protocol": "dokodemo-door",
      "settings": {
        "network": "tcp",
        "followRedirect": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "streamSettings": {
        "sockopt": {
          "tproxy": "redirect"
        }
      }
    },
    {
      "tag": "socks-in",
      "listen": "127.0.0.1",
      "port": 10808,
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    },
    {
      "tag": "http-in",
      "listen": "127.0.0.1",
      "port": 10809,
      "protocol": "http",
      "settings": {}
    }
  ],
  "outbounds": [
    {
      "tag": "node1-openai",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "YOUR_NODE1_SERVER",
            "port": 443,
            "users": [
              {
                "id": "YOUR_UUID",
                "encryption": "none",
                "flow": "xtls-rprx-vision"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "serverName": "YOUR_SERVER_NAME",
          "fingerprint": "random",
          "publicKey": "YOUR_PUBLIC_KEY",
          "shortId": "",
          "spiderX": "/"
        }
      }
    },
    {
      "tag": "node2-default",
      "protocol": "trojan",
      "settings": {
        "servers": [
          {
            "address": "YOUR_NODE2_SERVER",
            "port": 443,
            "password": "YOUR_PASSWORD"
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "tls",
        "tlsSettings": {
          "serverName": "YOUR_NODE2_SERVER_NAME",
          "allowInsecure": true,
          "alpn": ["http/1.1"]
        }
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "block",
      "protocol": "blackhole",
      "settings": {}
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "ip": [
          "geoip:private",
          "192.168.2.0/24"
        ],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "domain": [
          "domain:openai.com",
          "domain:chatgpt.com",
          "domain:oaistatic.com",
          "domain:oaiusercontent.com",
          "domain:openaiapi-site.azureedge.net",
          "domain:auth.openai.com",
          "domain:platform.openai.com",
          "domain:api.openai.com",
          "domain:cdn.openai.com",
          "domain:chat.openai.com",
          "domain:ab.chatgpt.com",
          "domain:files.oaiusercontent.com",
          "domain:chatgpt.livekit.cloud",
          "domain:featuregates.org",
          "regexp:(^|\\.)codex\\.",
          "regexp:(^|\\.)gpt\\.",
          "regexp:(^|\\.)openai\\.",
          "regexp:(^|\\.)chatgpt\\.",
          "keyword:openai",
          "keyword:chatgpt",
          "keyword:codex",
          "keyword:gpt"
        ],
        "outboundTag": "node1-openai"
      },
      {
        "type": "field",
        "network": "tcp,udp",
        "outboundTag": "node2-default"
      }
    ]
  }
}
```

## 附录 C：iptables Bash 命令集中区

> 说明：本附录内容与正文第 8、9 节重复，目的是提供“集中复制区”。正文原文保持不变。

### C.1 强制透明代理：apply 脚本

```bash
#!/usr/bin/env bash
set -e

iptables -t nat -N XRAY 2>/dev/null || true

# UDP 443 (QUIC) 去重后添加，避免绕过 TCP 透明代理
while iptables -C OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null; do
  iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable
done
iptables -A OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable

# OUTPUT -> XRAY 跳转去重后添加，确保仅一条
while iptables -t nat -C OUTPUT -p tcp -j XRAY 2>/dev/null; do
  iptables -t nat -D OUTPUT -p tcp -j XRAY
done
iptables -t nat -A OUTPUT -p tcp -j XRAY

# 重建 XRAY 链
iptables -t nat -F XRAY
iptables -t nat -A XRAY -d 127.0.0.0/8 -j RETURN
iptables -t nat -A XRAY -d 10.0.0.0/8 -j RETURN
iptables -t nat -A XRAY -d 172.16.0.0/12 -j RETURN
iptables -t nat -A XRAY -d 192.168.0.0/16 -j RETURN
iptables -t nat -A XRAY -d 224.0.0.0/4 -j RETURN
iptables -t nat -A XRAY -d 240.0.0.0/4 -j RETURN
iptables -t nat -A XRAY -m owner --uid-owner 65534 -j RETURN
iptables -t nat -A XRAY -p tcp -j REDIRECT --to-ports 12345
```

### C.2 强制透明代理：teardown 脚本

```bash
#!/usr/bin/env bash
set -e
sudo iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null || true
sudo iptables -t nat -D OUTPUT -p tcp -j XRAY || true
sudo iptables -t nat -F XRAY || true
sudo iptables -t nat -X XRAY || true
```

### C.3 强制模式：常用执行命令

```bash
sudo nano /usr/local/bin/xray-iptables-apply.sh
sudo chmod +x /usr/local/bin/xray-iptables-apply.sh
sudo /usr/local/bin/xray-iptables-apply.sh
sudo nano /usr/local/bin/xray-iptables-teardown.sh
sudo chmod +x /usr/local/bin/xray-iptables-teardown.sh
sudo /usr/local/bin/xray-iptables-teardown.sh
sudo iptables-save
```

### C.4 健康检查方案：healthcheck 脚本

```bash
#!/usr/bin/env bash
set -e

XRAY_CHAIN="XRAY"
XRAY_PORT="12345"

xray_is_healthy() {
  systemctl is-active --quiet xray && ss -ltn | awk '{print $4}' | grep -qE '(^|:)12345$'
}

apply_proxy_rules() {
  iptables -t nat -N "$XRAY_CHAIN" 2>/dev/null || true
  iptables -t nat -F "$XRAY_CHAIN"

  # UDP 443 (QUIC) 去重后添加，避免绕过 TCP 透明代理
  while iptables -C OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null; do
    iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable
  done
  iptables -A OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable

  # OUTPUT -> XRAY 跳转去重后添加，确保仅一条
  while iptables -t nat -C OUTPUT -p tcp -j "$XRAY_CHAIN" 2>/dev/null; do
    iptables -t nat -D OUTPUT -p tcp -j "$XRAY_CHAIN"
  done
  iptables -t nat -A OUTPUT -p tcp -j "$XRAY_CHAIN"

  iptables -t nat -A "$XRAY_CHAIN" -d 127.0.0.0/8 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -d 10.0.0.0/8 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -d 172.16.0.0/12 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -d 192.168.0.0/16 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -d 224.0.0.0/4 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -d 240.0.0.0/4 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -m owner --uid-owner 65534 -j RETURN
  iptables -t nat -A "$XRAY_CHAIN" -p tcp -j REDIRECT --to-ports "$XRAY_PORT"
}

apply_direct_rules() {
  while iptables -t nat -C OUTPUT -p tcp -j "$XRAY_CHAIN" 2>/dev/null; do
    iptables -t nat -D OUTPUT -p tcp -j "$XRAY_CHAIN"
  done
  iptables -t nat -F "$XRAY_CHAIN" 2>/dev/null || true

  # Xray 不健康时，清理 UDP 443 REJECT，避免规则累积并恢复直连
  while iptables -C OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable 2>/dev/null; do
    iptables -D OUTPUT -p udp --dport 443 -j REJECT \
  --reject-with icmp-port-unreachable
  done
}

if xray_is_healthy; then
  echo "Xray healthy -> apply proxy rules"
  apply_proxy_rules
else
  echo "Xray unhealthy -> switch to direct mode"
  apply_direct_rules
fi
```

### C.5 健康检查方案：systemd 文件与启用命令

```bash
sudo nano /usr/local/bin/xray-iptables-healthcheck.sh
sudo chmod +x /usr/local/bin/xray-iptables-healthcheck.sh
sudo /usr/local/bin/xray-iptables-healthcheck.sh
sudo nano /etc/systemd/system/xray-iptables-healthcheck.service
sudo nano /etc/systemd/system/xray-iptables-healthcheck.timer
sudo systemctl daemon-reload
sudo systemctl enable xray-iptables-healthcheck.timer
sudo systemctl start xray-iptables-healthcheck.timer
sudo systemctl list-timers --all | grep xray
```
