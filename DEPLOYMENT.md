# 博客系统部署详细流程
```markdown
DEPLOYMENT.md
├── 一、云服务器准备
├── 二、部署 JDK 环境
├── 三、部署 MySQL 数据库
├── 四、部署 Tomcat
├── 五、部署 Nginx 反向代理
├── 六、安全组配置
├── 七、日志与排错
└── 八、Prometheus 监控体系搭建 ← 新增
    ├── 8.1 架构说明
    ├── 8.2 部署 Exporter（Node/Nginx/Tomcat/MySQL）
    ├── 8.3 验证 Exporter
    ├── 8.4 配置 Prometheus 抓取
    ├── 8.5 导入 Grafana 仪表盘
    ├── 8.6 验证清单
    └── 8.7 管理节点 告警规则配置
````

## 一、云服务器准备
1. 登录阿里云控制台，创建 1 台 ECS 实例
2. 修改主机名：
```bash
    hostnamectl set-hostname myblog
    bash    #刷新
```

## 二、部署 JDK 环境
1. 创建安装包目录并上传(也可以直接联网下载相关包jdk)
2. 解压：
```bash
   tar -xzf jdk-8u241-linux-x64.tar.gz -C /opt/
```
3. 配置环境变量确保都可以加载到(vi /etc/profile)：
```bash
   export JAVA_HOME=/opt/jdk1.8.0_241
   export PATH=$JAVA_HOME/bin:$PATH
   export CLASSPATH=$JAVA_HOME/lib:$CLASSPATH
```
4. 生效验证：
```
   source /etc/profile && java -version
```

## 三、部署 MySQL 数据库
1. 安装 MySQL 官方仓库并安装
```
    dnf install -y https://dev.mysql.com/gemysql80-community-release-el8-6.noarch.rpm
```
2. 查看是否启动MySQL8.0仓库
```
    dnf repolist enabled | grep mysql
```
3. 安装MySQL8
```
    dnf install -y mysql-community-server
```
4. 启动MySQL相关服务
```
    systemctl start mysqld
    systemctl enable mysqld
    systemctl status mysqld
```
5. 获取临时密码：
```
   grep password /var/log/mysqld.log
```
6. 安全初始化：
```
   mysql_secure_installation
```
7. 导入项目数据：
```
   mysql -uroot -p < schema.sql
   mysql -uroot -p < data.sql
```
## 四、部署 Tomcat(也可以直接联网下载相关包)
1. 解压：
```
   tar -xzf apache-tomcat-9.0.97.tar.gz -C /opt/
```
2. 部署项目包：
```
   rm -rf /opt/apache-tomcat-9.0.97/webapps/ROOT
   cp ROOT.war /opt/apache-tomcat-9.0.97/webapps/
```
3. 启动：
```
   ./opt/apache-tomcat-9.0.97/bin/startup.sh
```

## 五、部署 Nginx 反向代理
1. 安装启动
2. 修改核心配置：
```
   location / {
       proxy_pass http://127.0.0.1:8080/;
   }
```
3. 重载生效：
```
   nginx -s reload
```

## 六、安全组配置
| 端口 | 协议 | 用途 |
|------|------|--------|
| 22   | TCP  | SSH 连接 |
| 80   | TCP  | Nginx 对外服务 |
| 8080 | TCP  | Tomcat 直接访问 |

## 七、监控与告警
- 开通阿里云云监控
- 创建告警规则：CPU/内存超过95%即短信告警
- 实时日志排查：
```
  tail -100f /opt/apache-tomcat-9.0.97/logs/catalina.out
  tail -100f /var/log/nginx/access.log
```

## 八、Prometheus 监控体系搭建
关于下载太慢了，就用镜像，或者直接在github上下载相关包，再复制

### 8.1 监控架构说明
- 被监控节点：myblog ECS（Nginx、Tomcat、MySQL、Node Exporter）
- 管理节点：独立 ECS（Prometheus + Grafana + Alertmanager）

### 8.2 被监控节点：部署 Exporter

#### Node Exporter（系统资源）
```bash
cd /opt
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar -xzf node_exporter-1.8.2.linux-amd64.tar.gz
mv node_exporter-1.8.2.linux-amd64 node_exporter

cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
ExecStart=/opt/node_exporter/node_exporter --web.listen-address=:9100
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start node_exporter
systemctl enable node_exporter
```

#### Nginx Exporter（Nginx 监控）
```bash
# 1. 开启 Nginx 状态页
vim /etc/nginx/nginx.conf
# 在 server 块中添加：
# location /nginx_status {
#     stub_status on;
#     allow 127.0.0.1;
#     allow 172.23.0.0/16;  # 内网段
#     deny all;
# }
nginx -s reload

# 2. 部署 Exporter
cd /opt
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.1.0/nginx-prometheus-exporter_1.1.0_linux_amd64.tar.gz
tar -xzf nginx-prometheus-exporter_1.1.0_linux_amd64.tar.gz

cat > /etc/systemd/system/nginx-exporter.service << 'EOF'
[Unit]
Description=Nginx Prometheus Exporter
After=network.target

[Service]
ExecStart=/opt/nginx-prometheus-exporter -nginx.scrape-uri=http://127.0.0.1/nginx_status -web.listen-address=:9113
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start nginx-exporter
systemctl enable nginx-exporter
```

#### Tomcat JMX Exporter（Tomcat 监控）
```bash
# 1. 下载 JMX Exporter JAR 包
cd /opt
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar
# 国内镜像加速下载
wget https://maven.aliyun.com/repository/public/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar

# 2. 创建配置文件
cat > /opt/tomcat_jmx.yml << 'EOF'
lowercaseOutputName: true
lowercaseOutputLabelNames: true
rules:
  - pattern: 'Catalina<type=ThreadPool, name="(\w+-?\w+)"><>(currentThreadCount|currentThreadsBusy|maxThreads)'
    name: tomcat_threads_$2
    labels:
      pool: "$1"
  - pattern: 'java.lang<type=Memory><>(\w+)'
    name: jvm_memory_$1_bytes
  - pattern: 'java.lang<type=GarbageCollector, name=(.+)><>(CollectionCount|CollectionTime)'
    name: jvm_gc_$2_total
    labels:
      collector: "$1"
EOF

# 3. 修改 Tomcat 启动脚本
vim /opt/apache-tomcat-9.0.97/bin/catalina.sh
# 在文件开头添加：
# JAVA_OPTS="$JAVA_OPTS -javaagent:/opt/jmx_prometheus_javaagent-0.20.0.jar=9114:/opt/tomcat_jmx.yml"

# 4. 重启 Tomcat
/opt/apache-tomcat-9.0.97/bin/shutdown.sh
/opt/apache-tomcat-9.0.97/bin/startup.sh
```

#### MySQL Exporter（MySQL 监控）
```bash
# 1. 创建监控专用账号
mysql -uroot -p << 'EOF'
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'Monitor123@';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;
EOF

# 2. 创建连接配置
cat > /etc/mysqld-exporter.cnf << 'EOF'
[client]
user=exporter
password=Monitor123@
host=127.0.0.1
EOF
chmod 600 /etc/mysqld-exporter.cnf

# 3. 部署 Exporter
cd /opt
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz
tar -xzf mysqld_exporter-0.15.1.linux-amd64.tar.gz
mv mysqld_exporter-0.15.1.linux-amd64 mysqld_exporter

cat > /etc/systemd/system/mysqld-exporter.service << 'EOF'
[Unit]
Description=MySQL Prometheus Exporter
After=network.target

[Service]
ExecStart=/opt/mysqld_exporter/mysqld_exporter --config.my-cnf=/etc/mysqld-exporter.cnf --web.listen-address=:9104
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start mysqld-exporter
systemctl enable mysqld-exporter
```

### 8.3 验证 Exporter 运行状态
```bash
# 检查所有 Exporter 端口是否正常监听
ss -tlnp | grep -E "9100|9113|9114|9104"

# 验证指标输出
curl http://127.0.0.1:9100/metrics | head
curl http://127.0.0.1:9113/metrics | grep nginx_up
curl http://127.0.0.1:9114/metrics | grep tomcat
curl http://127.0.0.1:9104/metrics | grep mysql_up
```

### 8.4 管理节点：配置 Prometheus 抓取
```bash
vim /etc/prometheus/prometheus.yml
# 在 scr ape_configs 中添加：
  - job_name: 'node'
    static_configs:
      - targets: ['博客内网ip:9100']

  - job_name: 'nginx'
    static_configs:
      - targets: ['博客内网ip:9113']

  - job_name: 'tomcat'
    static_configs:
      - targets: ['博客内网ip:9114']

  - job_name: 'mysql'
    static_configs:
      - targets: ['博客内网ip:9104']

重载配置：
curl -X POST http://localhost:9090/-/reload
```

### 8.5 管理节点：导入 Grafana 仪表盘
| 监控对象   | 仪表盘模板 ID |
| ---------- | ------------- |
| 系统资源   | 1860          |
| Nginx      | 12708         |
| Tomcat     | 8563          |
| MySQL      | 7362          |
|    |          |
__导入步骤__：Grafana → Dashboards → Import → 输入 ID → 选择 Prometheus 数据源 → Import

### 8.6 管理节点 监控验证清单
```
# Prometheus Targets 状态
curl http://localhost:9090/api/v1/targets | grep -E "health|job"
```

### 8.7 管理节点 告警规则配置
```markdown
/alert_rules/
├── system.yml # 系统资源告警（CPU/内存/磁盘/网络）
└── blog/ # 博客项目服务告警
   ├── nginx.yml # Nginx 告警规则
   ├── tomcat.yml # Tomcat 告警规则
   └── mysql.yml # MySQL 告警规则
```
**应用告警规则**
```bash
# 检查语法
promtool check rules /etc/prometheus/alert_rules/blog/*.yml
# 重载 Prometheus
curl -X POST http://localhost:9090/-/reload
# 验证规则已加载，输出告警组的名字
curl -s http://localhost:9090/api/v1/rules | jq '.data.groups[].name'
```
## Grafana 访问
 浏览器打开 http://<管理机IP>:3000 → 查看各个仪表盘数据是否正常显示

