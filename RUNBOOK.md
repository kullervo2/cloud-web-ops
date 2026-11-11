# Runbook · 云上 Web 服务故障处理手册

> \\\*\\\*项目\\\*\\\*：cloud-web-ops
> \\\*\\\*环境\\\*\\\*：阿里云 ECS（2C2G）/ Alibaba Cloud Linux 3.2104 / Nginx + PHP-FPM + MariaDB 10.5
> \\\*\\\*维护\\\*\\\*：<Kullervo2>
> \\\*\\\*最后更新\\\*\\\*：2026-11-11

\---

## 故障索引

|#|故障|级别|发现方式|恢复耗时|
|-|-|-|-|-|
|1|磁盘写满|P2|HighDiskUsage 告警|2 分钟|
|2|Nginx 进程被杀|P1|网站 502|3 秒（自愈）|
|3|fastcgi 路径错误|P1|网站 502|<X> 分钟|
|4|数据库连接数打满|P1|页面报 Too many connections|3 分钟|

**级别定义**：P0 = 服务不可用且无法登录；P1 = 服务不可用；P2 = 容量告警，服务可用

\---

## 通用排障原则

分层排查，从外到内。**没有确认上层，不要直接跳进应用代码。**

|层|查什么|命令|
|-|-|-|
|1 网络|安全组、firewalld、端口是否监听|`firewall-cmd --list-all`、`ss -tlnp`|
|2 主机|CPU / 内存 / 磁盘 / 负载|`top`、`free -h`、`df -h`|
|3 服务|systemd 状态、最近日志|`systemctl status X`、`journalctl -u X -n 50`|
|4 应用|Nginx / PHP 日志、数据库连接|`tail /var/log/nginx/error.log`、`SHOW PROCESSLIST`|

**状态码口诀**

* **502** = 路不通（后端连不上）
* **500** = 人到了但出事了（应用内部报错）
* **504** = 人到了但等太久（后端超时）

\---

## 故障 1：磁盘写满

**现象**
写文件报 `No space left on device`；网站可能返回 500；Prometheus 触发 `HighDiskUsage`。

**发现方式**
告警 webhook，或巡检脚本 `check.sh` 返回非 0。

**影响范围**
MariaDB 写入失败，业务不可用。

**排查过程**
df -h                                        # 定位是哪个分区满
df -i                                        # 排除 inode 耗尽
du -sh /\\\* 2>/dev/null | sort -h | tail -5    # 找大目录
lsof | grep deleted                          # 已删除但仍被进程占用的文件

**根因**
`/tmp/fill.img`（演练用 25G 填充文件）。

**处理步骤**
rm -f /tmp/fill.img
df -h

**恢复耗时**：2 分钟

**预防措施**

* 磁盘使用率 80% 告警（已配置）
* logrotate 日志轮转
* `/tmp` 定期清理

**复盘**
`df` 显示满但 `du` 找不到大文件时，通常是文件已删除但进程仍持有句柄，用 `lsof | grep deleted` 定位，重启对应进程释放空间。

\---

## 故障 2：Nginx 进程被杀

**现象**
网站 502 或 Connection refused。

**发现方式**
巡检脚本网站可用性检查失败，或用户报障。

**影响范围**
全站不可用。

**排查过程**
curl -I http://127.0.0.1
systemctl status nginx
ss -tlnp | grep :80
journalctl -u nginx -n 50 --no-pager

**根因**
进程被 `kill -9`；systemd 默认 `Restart=no`，不会自动拉起。

**处理步骤**
systemctl start nginx

**根治（已实施）**
# /etc/systemd/system/nginx.service.d/restart.conf
\\\[Service]
Restart=always
RestartSec=3

`systemctl daemon-reload` 后，进程崩溃 3 秒内自动恢复。

**恢复耗时**：手动 1 分钟；配置自愈后 3 秒

**预防措施**

* `Restart=always`
* 增加端口/进程存活监控

**复盘**
从"手动重启"升级为"自愈"，消除了故障重复发生的条件，比修复本身更有价值。

\---

## 故障 3：fastcgi 路径错误导致 502

**现象**
访问首页返回 502 Bad Gateway。

**发现方式**
用户报障，或巡检 `curl` 返回 502。

**排查过程**
curl -I http://127.0.0.1          # 502
tail -20 /var/log/nginx/error.log
# connect() to unix:/run/php-fpm/wrong.sock failed (2: No such file or directory)
ls -l /run/php-fpm/               # 实际 socket 是 www.sock
nginx -T | grep fastcgi\\\_pass      # 确认真正生效的配置

**根因**
`/etc/nginx/default.d/php.conf` 中 `fastcgi\\\_pass` 指向了不存在的 socket。

**处理步骤**
cp /root/drill/php.conf.bak /etc/nginx/default.d/php.conf
nginx -t \&\& systemctl reload nginx

**恢复耗时**：5 分钟

**预防措施**

* 变更前备份配置
* 变更流程固定为：备份 → 修改 → `nginx -t` → reload → **实际访问验证**

**复盘**
`nginx -t` 只校验语法，**不校验 upstream 是否可达**。socket 不存在时它照样报 success，真正错误只在请求到达时暴露。

\---

## 故障 4：数据库连接数打满

**现象**
页面显示 `连接失败: Too many connections`。

**发现方式**
用户报障，或监控 `Threads\\\_connected` 接近 `max\\\_connections`。

**排查过程**
mysql -e "SHOW STATUS LIKE 'Threads\_connected';"

mysql -e "SHOW PROCESSLIST;"

mysql -e "SHOW VARIABLES LIKE 'max\_connections';"

**根因**
`max\\\_connections` 被临时调低至 10，且存在 20 个 `SLEEP(600)` 长连接占满连接池。

**处理步骤**
pkill -f "SELECT SLEEP"
mysql -e "SET GLOBAL max\_connections = 151;"

**逃生口**
连接占满时普通用户无法登录，但 **SUPER 用户保留一条额外连接**。实在不行 `systemctl restart mariadb`（`SET GLOBAL` 不持久，重启恢复默认值）。

**恢复耗时**：3 分钟

**预防措施**

* 应用侧使用连接池，设置 `wait\\\_timeout`
* 监控 `Threads\\\_connected / max\\\_connections` 使用率
* **不要无脑调大 `max\\\_connections`**：每连接占内存，调大易 OOM，应先排查连接泄漏

**复盘**
`SHOW PROCESSLIST` 中长时间 `Sleep` 的连接是典型泄漏信号。正确顺序是"先找泄漏，再考虑调参"。

\---

附录：常用命令速查
# 服务管理
systemctl status|restart|reload <svc>
journalctl -u <svc> -n 50 --no-pager

# 端口与连接
ss -tlnp
ss -tnp | grep <port>

# 资源
df -h ; df -i ; free -h
top -bn1 | head -15

# Nginx
nginx -t
nginx -T | grep -n <keyword>
tail -50 /var/log/nginx/error.log

# MariaDB（管理操作）
mysql --defaults-extra-file=/root/.my\\\_root.cnf -e "<SQL>"

# 备份与恢复
/opt/scripts/mysql\\\_backup.sh
zcat /opt/backup/mysql/latest.sql.gz | mysql blog

# 巡检
/opt/scripts/check.sh ; echo "退出码: $?"



