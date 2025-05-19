# Zeek 部署

- [Zeek 部署](#zeek-部署)
  - [PF\_RING（ZC）安装（可选，推荐）](#pf_ringzc安装可选推荐)
  - [Postfix 安装](#postfix-安装)
  - [Zeek](#zeek)
    - [安装](#安装)
    - [配置](#配置)
      - [`node.cfg`](#nodecfg)
      - [`networks.cfg`](#networkscfg)
      - [`zeekctl.cfg`](#zeekctlcfg)

## PF_RING（ZC）安装（可选，推荐）

[官方链接](https://packages.ntop.org/apt-stable/)

以 Ubuntu 24.04 LTS 为例：

```bash
apt-get install software-properties-common wget
add-apt-repository universe
wget https://packages.ntop.org/apt-stable/24.04/all/apt-ntop-stable.deb

# 下面这行和前面分开执行
apt install ./apt-ntop-stable.deb

# 安装完上面，系统知道了 ntop 的软件源，就该更新本地的软件包列表，以便 apt 能够从中获取最新的软件包信息
apt update

# 之后就可以安装相关软件了
apt install pfring pfring-dkms
```

## Postfix 安装

```bash
apt install postfix mailutils
```

为了防止垃圾邮件，阿里云默认会限制或阻止其 ECS (云服务器) 实例通过 TCP 端口 25 向外部直接发送邮件，这里使用阿里云的邮件推送服务，配置 Postfix 作为客户端，通过一个“中继主机 (smarthost)”来发送所有邮件。

这里先准备一个域名，我用的 windsland.top，然后根据[阿里云邮件推送概览](https://dm.console.aliyun.com/#/directmail/Home/cn-hangzhou)的步骤进行配置。有两步：创建发信域名和创建发信地址

创建发信域名时，选择先创建子域名 mail.windsland.top，然后到域名控制台添加几条域名解析配置。

创建发信地址时，填入 `alerts@mail.windsland.top` ，类型选择触发邮件。

配置完上面的前置工作，再回到 postfix 的配置。

主要配置文件是 `etc/postfix/main.cf`，如下：

```ini
# See /usr/share/postfix/main.cf.dist for a commented, more complete version


# Debian specific:  Specifying a file name will cause the first
# line of that file to be used as the name.  The Debian default
# is /etc/mailname.
#myorigin = /etc/mailname

smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 3.6 on
# fresh installs.
compatibility_level = 3.6



# TLS parameters
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_security_level=may

smtp_tls_CApath=/etc/ssl/certs
smtp_tls_security_level=encrypt
smtp_use_tls = yes
smtp_tls_wrappermode = yes
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

smtputf8_enable = no

smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
myhostname = iZ2zegn1ssy7ysfggftg4lZ
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname
mydestination = $myhostname, localhost.localdomain, localhost
relayhost = [smtpdm.aliyun.com]:465

# SASL Authentication for smarthost (Aliyun)
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous

# Sender address rewriting
sender_canonical_maps = hash:/etc/postfix/sender_canonical

mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all
```

修改 `main.cf` 后，执行以下命令：

```bash
# 创建 sasl_passwd 文件
nano /etc/postfix/sasl_passwd
```

内容添加 `[smtpdm.aliyun.com]:465    SMTP用户名:SMTP密码`

```bash
# 然后设置权限并生成数据库文件
chmod 600 /etc/postfix/sasl_passwd
postmap /etc/postfix/sasl_passwd

# 创建 sender_canonical 文件
nano /etc/postfix/sender_canonical
```

内容添加 `root    alerts@mail.windsland.top`

```bash
postmap /etc/postfix/sender_canonical
# 重启服务
systemctl restart postfix
# 测试并检查日志
echo "这是来自 Postfix (通过阿里云中继) 的测试邮件内容" | mail -s "Postfix 阿里云中继测试" winds_land@foxmail.com
sudo tail -f /var/log/mail.log
```

测试成功，收到测试邮件，postfix 安装完成

## Zeek

### 安装

1. [部署Zeek](https://docs.zeek.org/en/lts/install.html)（暂不考虑docker）

   ```bash
   echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | sudo tee /etc/apt/sources.list.d/security:zeek.list
   curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
   sudo apt update
   sudo apt install zeek
   ```

2. 将zeek添加到环境变量

   ```bash
   echo 'export PATH=$PATH:/opt/zeek/bin' >> ~/.bashrc
   source ~/.bashrc
   ```

3. 检测安装完毕

   ```bash
   zeek --version
   which zeek
   ```

### 配置

最重要的是安装目录下 `etc/node.cfg`，修改是必要的。
修改配置后执行 `zeekctl deploy`，便可重新尝试部署，输入无误便可。
`zeekctl status` 便可检查所以节点的状态。

#### `node.cfg`

```ini
# # Example ZeekControl node configuration.
# #
# # This example has a standalone node ready to go except for possibly changing
# # the sniffing interface.

# # This is a complete standalone configuration.  Most likely you will
# # only need to change the interface.
# [zeek]
# type=standalone
# host=localhost
# interface=eth0

# Below is an example clustered configuration. If you use this,
# remove the [zeek] node above.

[logger-1]
type=logger
host=localhost

[manager]
type=manager
host=localhost

[proxy-1]
type=proxy
host=localhost

[worker-1]
type=worker
host=localhost
interface=eth0
lb_method=pf_ring
lb_procs=2

[worker-2]
type=worker
host=localhost
interface=eth0
lb_method=pf_ring
lb_procs=2
```

独立配置这里不再描述，直接介绍集群配置，上面是修改完的配置。

#### `networks.cfg`

作用是告诉 Zeek 哪些 IP 地址段属于你的“本地网络”。这对于 Zeek 分析流量（例如，区分内部流量、外部流量、入站连接、出站连接等）非常重要

```ini
# List of local networks in CIDR notation, optionally followed by a descriptive
# tag. Private address space defined by Zeek's Site::private_address_space set
# (see scripts/base/utils/site.zeek) is automatically considered local. You can
# disable this auto-inclusion by setting zeekctl's PrivateAddressSpaceIsLocal
# option to 0.
#
# Examples of valid prefixes:
#
# 1.2.3.0/24        Admin network
# 2607:f140::/32    Student network
```

什么时候需要修改这个文件：

- 如果你的“本地网络”使用了公共 IP 地址： 例如，你的校园网或公司网络可能拥有一些公共 IP 地址段，这些地址对于你的组织来说是“内部”的。
- 如果你想为特定的本地子网添加描述性标签： 即使是私有地址空间，你也可以为特定的子网（例如 192.168.1.0/24 Student WiFi）添加标签，这有助于后续分析。
- 如果你的本地网络结构非常复杂，并且你想明确定义所有本地段，而不是仅仅依赖自动识别。
- 如果你禁用了 PrivateAddressSpaceIsLocal，那么你就必须在这里明确列出所有本地网络，包括私有地址。

#### `zeekctl.cfg`

```ini
## Global ZeekControl configuration file.

###############################################
# Mail Options

# Recipient address for all emails sent out by Zeek and ZeekControl.
MailTo = winds_land@foxmail.com

# Mail connection summary reports each log rotation interval.  A value of 1
# means mail connection summaries, and a value of 0 means do not mail
# connection summaries.  This option has no effect if the trace-summary
# script is not available.
MailConnectionSummary = 1

# Interval in seconds for sending mail connection summary reports.  A value
# of 0 means only send connection summaries each log rotation interval.
MailConnectionSummaryInterval = 14400

# Lower threshold (in percentage of disk space) for space available on the
# disk that holds SpoolDir. If less space is available, "zeekctl cron" starts
# sending out warning emails.  A value of 0 disables this feature.
MinDiskSpace = 10

# Send mail when "zeekctl cron" notices the availability of a host in the
# cluster to have changed.  A value of 1 means send mail when a host status
# changes, and a value of 0 means do not send mail.
MailHostUpDown = 1

###############################################
# Logging Options

# Rotation interval in seconds for log files on manager (or standalone) node.
# A value of 0 disables log rotation.
LogRotationInterval = 3600

# Expiration interval for archived log files in LogDir.  Files older than this
# will be deleted by "zeekctl cron".  The interval is an integer followed by
# one of these time units:  day, hr, min.  A value of 0 means that logs
# never expire.
LogExpireInterval = 7day

# Enable ZeekControl to write statistics to the stats.log file.  A value of 1
# means write to stats.log, and a value of 0 means do not write to stats.log.
StatsLogEnable = 1

# Number of days that entries in the stats.log file are kept.  Entries older
# than this many days will be removed by "zeekctl cron".  A value of 0 means
# that entries never expire.
StatsLogExpireInterval = 30

###############################################
# Other Options

# Show all output of the zeekctl status command.  If set to 1, then all output
# is shown.  If set to 0, then zeekctl status will not collect or show the peer
# information (and the command will run faster).
StatusCmdShowAll = 0

# Number of days that crash directories are kept.  Crash directories older
# than this many days will be removed by "zeekctl cron".  A value of 0 means
# that crash directories never expire.
CrashExpireInterval = 0

# Site-specific policy script to load. Zeek will look for this in
# $PREFIX/share/zeek/site. A default local.zeek comes preinstalled
# and can be customized as desired.
SitePolicyScripts = local.zeek

# Location of the log directory where log files will be archived each rotation
# interval.
LogDir = /opt/zeek/logs

# Location of the spool directory where files and data that are currently being
# written are stored.
SpoolDir = /opt/zeek/spool

# Location of the directory in which the databases for Broker datastore backed
# Zeek tables are stored.
BrokerDBDir = /opt/zeek/spool/brokerstore

# Default base directory for file extraction.
#
# The FileExtract module's prefix option will default be set to this
# value with the Cluster::node value appended.
FileExtractDir = /opt/zeek/spool/extract_files

# Location of other configuration files that can be used to customize
# ZeekControl operation (e.g. local networks, nodes).
CfgDir = /opt/zeek/etc
```

以上三个文件配置完，重新 deploy。
