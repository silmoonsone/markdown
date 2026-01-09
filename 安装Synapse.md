- 拉取镜像

```bash
docker pull matrixdotorg/synapse:latest
```

- 使用镜像内部工具生成配置
  注意im.host.com改成自己的域名\主机名

```bash
docker run -it --rm \
  -v synapse-data:/data \
  -e SYNAPSE_SERVER_NAME=im.host.com \
  -e SYNAPSE_REPORT_STATS=no \
  -e SYNAPSE_HTTP_PORT=8008 \
  -e SYNAPSE_CONFIG_DIR=/data \
  -e SYNAPSE_DATA_DIR=/data \
  -e TZ=Asia/Shanghai \
  -e UID=1000 \
  -e GID=1000 \
  matrixdotorg/synapse:latest generate
```

- 创建PostgreSQL
   注意参数使用 Collation&Character type "C"

- 更新配置文件设置数据库连接信息

```yaml
database:
  name: psycopg2
  args:
    user: #username
    password: #password
    database: #database
    host: pgsql0.host.host
    port: 5432
```

- 运行容器

```bash
docker run -d \
  -v synapse-data:/data \
  -p 8008:8008 \
  --name synapse \
  matrixdotorg/synapse:latest
```

- 创建第一个用户

```bash
docker exec -it synapse register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008
```

- 安装Web管理

```bash
docker run -d -p 8080:80 awesometechnologies/synapse-admin
```

- 启用全集用户搜索和查找

```yaml
user_directory:
  enabled: true
  search_all_users: true
```

- 设置限制

```yaml
###############################################################################
# 注册（Registration）
###############################################################################

# 开放注册：客户端可直接注册
enable_registration: true

# 允许无验证注册：不走邮箱/验证码时需要
enable_registration_without_verification: true

# 不启用验证码（公网建议配合反代/WAF/限流）
enable_registration_captcha: false


###############################################################################
# 注册 Token（Registration Tokens）— 先注释，未来打开即变成“有 token 才能注册”
###############################################################################
#registration_requires_token: true


###############################################################################
# 管理员注册共享密钥（Admin shared secret）— 可选：用于管理员/自动化创建账号
###############################################################################
#registration_shared_secret: "REPLACE_WITH_STRONG_RANDOM"


###############################################################################
# 限流（Rate limiting）— 先注释，未来需要时整段取消注释即可
###############################################################################
#ratelimiting:
#  # 发消息限流（按账号）
#  rc_message:
#    per_second: 5.0
#    burst_count: 100
#
#  # 注册限流（按 IP）
#  # 目标：每 IP 每小时最多 5 次注册
#  # 计算：5 / 3600 = 0.0013888889 次/秒
#  rc_registration:
#    per_second: 0.0013888889
#    burst_count: 5
#
#  # 登录限流
#  rc_login:
#    # 同一 IP 的登录请求
#    address:
#      per_second: 5.0
#      burst_count: 50
#
#    # 同一账号的登录请求
#    account:
#      per_second: 5.0
#      burst_count: 50
#
#    # 同一账号的“失败登录尝试”
#    failed_attempts:
#      per_second: 0.3
#      burst_count: 5
#
#  # 管理员批量删消息限流
#  rc_admin_redaction:
#    per_second: 5.0
#    burst_count: 100
#
#  # 加房间限流（本地房间 / 远端房间）
#  rc_joins:
#    local:
#      per_second: 5.0
#      burst_count: 50
#    remote:
#      per_second: 1.0
#      burst_count: 20
#
#  # 邀请限流
#  rc_invites:
#    per_room:
#      per_second: 2.0
#      burst_count: 50
#    per_user:
#      per_second: 0.5
#      burst_count: 20
#
#  # 联邦入站请求限流
#  rc_federation:
#    window_size: 1000
#    sleep_limit: 200
#    sleep_delay: 10
#    reject_limit: 500
#    concurrent: 50

```
