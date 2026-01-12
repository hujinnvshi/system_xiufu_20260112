# SecSmart 网络威胁监测系统 - 问题诊断报告

**诊断日期**: 2026-01-12
**系统地址**: 172.16.196.203
**系统名称**: AllstarOS
**问题现象**: Web 页面显示"正在启动"，系统功能异常

---

## 一、产品概述

### 1.1 产品信息

**产品名称**: 定制化专用生产数字化网络威胁监测与智能分析系统
**产品简称**: SecSmart (SecSmart Server)
**产品类型**: 工业网络安全监测与分析平台
**版本信息**: 3.0.3-ISA-dev-SNAPSHOT

### 1.2 主要功能

- 工业网络流量监测与分析
- 设备拓扑发现与管理
- 安全事件检测与告警
- 威胁情报与策略管理
- 日志审计与报表生成
- 支持多种工业协议（Modbus, S7, OPC-UA, Ethernet/IP, DNP3, IEC104 等）

---

## 二、系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                         用户界面层                            │
│                   Vue.js 单页应用 (SPA)                       │
│                   - 设备监控                                  │
│                   - 拓扑展示                                  │
│                   - 事件告警                                  │
│                   - 策略管理                                  │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTPS (443)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                      Web 服务层                              │
│                      Nginx 反向代理                          │
│                   - SSL/TLS 终止                            │
│                   - 静态资源服务                             │
│                   - API 请求代理                             │
│                   - 负载均衡                                 │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┴────────────────┐
         ▼                                ▼
┌────────────────────┐          ┌──────────────────────────┐
│  Python 中间件层    │          │   Java 应用层            │
│  routes.pyc        │          │   com.secsmart.server.  │
│  - 路由控制         │          │   App                    │
│  - 时间监控         │          │   - REST API 服务        │
│  - 进程管理         │          │   - 设备管理模块         │
│  pm_main.pyc        │          │   - 策略引擎             │
│  - DPI 管理         │          │   - 威胁分析引擎         │
│  control_agent      │          │   - 流量分析模块         │
│  - 控制代理         │          │   - 事件处理模块         │
└────────────────────┘          │   - 拓扑发现模块         │
                               │   - 用户管理模块         │
                               └──────┬───────────────────┘
                                      │
         ┌────────────────────────────┴────────────────┐
         ▼                ▼                ▼             ▼
┌─────────────┐  ┌─────────────┐  ┌─────────┐  ┌────────────┐
│   MySQL     │  │   Kafka     │  │Zookeeper│  │ 其他组件   │
│  数据持久化  │  │  消息队列   │  │ 协调服务 │  │            │
│             │  │  事件流     │  │         │  │            │
│             │  │  数据同步   │  │         │  │            │
└─────────────┘  └─────────────┘  └─────────┘  └────────────┘
```

### 2.2 技术栈详情

#### 前端技术
- **框架**: Vue.js (单页应用)
- **UI**: 自定义组件库
- **通信**: Axios + REST API
- **部署**: Nginx 静态文件服务

#### 后端技术
- **主应用**: Java (com.secsmart.server.App)
- **Web 容器**: Jetty 9.4.44
- **REST 框架**: RESTEasy 3.0.10.Final
- **序列化**: Jackson 2.4.2
- **ORM**: MyBatis 3.5.6
- **数据库**: MySQL 5.x
- **消息队列**: Kafka 0.8.2
- **协调服务**: Zookeeper 3.4.13
- **中间件**: Python 3.x

#### 基础设施
- **操作系统**: Ubuntu 18.04.6 LTS (ARM64)
- **Web 服务器**: Nginx
- **系统管理**: systemd
- **日志**: log4j, systemd-journald

---

## 三、运行环境

### 3.1 系统信息

```bash
主机名: AllstarOS
IP 地址: 172.16.196.203
操作系统: Ubuntu 18.04.6 LTS (Bionic Beaver)
内核版本: Linux 4.19.0 #3 SMP PREEMPT
架构: aarch64 (ARM64)
系统运行时间: 约 2 天（自上次重启）
```

### 3.2 服务状态

| 服务名 | 状态 | 端口 | 说明 |
|--------|------|------|------|
| nginx.service | ✅ running | 80, 443 | Web 服务器 |
| mysql.service | ✅ running | 3306 | 数据库 |
| kafka.service | ✅ running | 9092 | 消息队列 |
| zookeeper.service | ✅ running | 2181 | 协调服务 |
| middleware.service | ❌ **failed** | - | **Java 主应用** |
| process-manager.service | ✅ running | - | 进程管理 |
| setup.service | ✅ running | - | 系统初始化 |
| time-monitor.service | ✅ running | - | 时间监控 |
| ssh.service | ✅ running | 22 | SSH 服务 |

### 3.3 进程列表

```
主要进程:
- nginx: master + 8 workers
- mysqld: MySQL 数据库
- java (Kafka): 消息队列
- java (Zookeeper): 协调服务
- python routes.pyc: 中间件路由
- python pm_main.pyc: 进程管理
- python control_agent: 控制代理
- java com.secsmart.server.App: ❌ 启动失败
```

---

## 四、问题诊断

### 4.1 问题现象

**用户报告**:
- 访问 https://172.16.196.203/ 显示"正在启动"
- 页面无法正常加载
- 系统功能不可用

**实际观察**:
- 前端页面可以访问（Vue.js SPA 已加载）
- 但后端 API 无法响应
- 页面一直停留在"正在启动"状态

### 4.2 核心问题

#### 问题 1: Java 应用启动失败 ⚠️ **关键问题**

**错误日志**:
```
Jan 12 04:30:30 AllstarOS bash[6210]: Exception in thread "main" java.lang.NoClassDefFoundError: kafka/serializer/Encoder
Jan 12 04:30:30 AllstarOS bash[6210]: Caused by: java.lang.ClassNotFoundException: kafka.serializer.Encoder
Jan 12 04:30:30 AllstarOS bash[6210]:     at com.secsmart.server.App.start(App.java:153)
Jan 12 04:30:30 AllstarOS bash[6210]:     at com.secsmart.server.App.main(App.java:54)
```

**问题分析**:

1. **缺失的类**: `kafka.serializer.Encoder`
   - 这是 Kafka 0.8.x 版本的旧 API
   - 在 Kafka 新版本中已被移除

2. **依赖冲突**:
   - 应用使用的 Kafka jar: `kafka_2.9.2-0.8.2.0.jar`
   - 这个版本的 Serializer 接口与代码不兼容

3. **类加载失败**:
   ```
   java.lang.ClassNotFoundException: kafka.serializer.Encoder
   ```
   - 类路径中找不到这个类
   - 可能是 jar 包版本不匹配或缺失

**影响链**:
```
Kafka API 不兼容
    ↓
Java 应用无法初始化 Kafka 生产者/消费者
    ↓
应用启动失败 (App.start() 方法异常)
    ↓
Jetty 容器未启动
    ↓
REST API 不可用 (/api/* 接口无法访问)
    ↓
前端无法获取后端数据
    ↓
页面一直显示"正在启动"
```

### 4.3 依赖分析

#### Kafka 相关 jar 包

```bash
/usr/share/server/lib/kafka_2.9.2-0.8.2.0.jar     # Kafka 客户端 (旧版本)
/usr/share/server/lib/kafka-clients-0.8.2.0.jar   # Kafka 新客户端
/usr/share/server/lib/zookeeper-3.4.13.jar        # Zookeeper 客户端
/usr/share/server/lib/zkclient-0.11.jar            # Zookeeper 客户端封装
/usr/share/server/lib/netty-3.10.6.Final.jar       # 网络库
```

#### 问题的技术根源

Kafka 0.8.x 到 0.9+ 版本的 API 重大变更：

**旧 API (0.8.x)**:
```java
import kafka.serializer.Encoder;
import kafka.serializer.Decoder;
```

**新 API (0.9+)**:
```java
import org.apache.kafka.common.serialization.Serializer;
import org.apache.kafka.common.serialization.Deserializer;
```

应用代码使用旧 API，但类路径中的 jar 可能不匹配。

### 4.4 其他发现

#### 发现 1: Nginx 静态资源 404 错误

```
[error] open() "/usr/share/nginx/html/middleware_ui/images/devices/security/.svg" failed
```

**影响**: 轻微，不影响核心功能
**说明**: 前端尝试加载不存在的 SVG 图标

#### 发现 2: 系统重启历史

从日志可以看到：
- 系统在 2026-01-09 06:42 启动
- 最近一次 systemctl 重启在 2026-01-12 03:53（我们修复后的重启）
- middleware.service 在每次重启后都启动失败

#### 发现 3: Python 中间件正常

Python 中间件运行正常：
- routes.pyc: 路由服务
- pm_main.pyc: 进程管理
- control_agent: 控制代理

但这些中间件依赖 Java 主应用，所以也无法正常工作。

---

## 五、详细诊断步骤

### 5.1 检查清单

#### ✅ 已完成的检查

1. [x] 系统基本信息收集
2. [x] 服务状态检查 (systemctl list-units)
3. [x] 进程状态检查 (ps aux)
4. [x] 端口监听检查 (netstat)
5. [x] Nginx 配置检查
6. [x] 日志文件分析
7. [x] 依赖关系分析
8. [x] 类路径分析

### 5.2 诊断命令记录

```bash
# 1. 检查服务状态
systemctl list-units --type=service --state=running

# 2. 检查主要进程
ps aux | grep -E '(java|python|nginx|mysql|kafka)'

# 3. 检查端口监听
netstat -tlnp | grep -E ':(80|443|3306|9092|2181)'

# 4. 检查 Nginx 配置
nginx -T

# 5. 查看应用日志
journalctl -u middleware.service -n 50

# 6. 检查 Kafka 相关文件
ls -la /usr/share/server/lib/kafka*.jar

# 7. 检查类路径
ps aux | grep 'com.secsmart.server.App' | tr ' ' '\n' | grep 'jar'
```

---

## 六、解决方案建议

### 6.1 修复方案（按优先级排序）

#### 方案 1: 添加兼容的 Kafka jar 包 ⭐ **推荐**

**操作**:
```bash
# 检查是否有备份或历史版本
ls -la /usr/share/server/lib/kafka*.jar*

# 添加缺失的 Encoder 类
# 需要找到应用代码中使用的具体 Encoder 实现
```

**优点**: 最小改动
**风险**: 低
**时间**: 短

#### 方案 2: 修复代码以使用新 Kafka API

**操作**:
1. 定位使用 `kafka.serializer.Encoder` 的代码
2. 重构为使用 `org.apache.kafka.common.serialization.Serializer`
3. 重新编译打包
4. 部署新版本

**优点**: 长期稳定
**风险**: 中（需要完整测试）
**时间**: 长

#### 方案 3: 降级到兼容的 Kafka 版本

**操作**:
```bash
# 检查系统是否有 Kafka 相关的系统包
dpkg -l | grep kafka

# 可能需要安装兼容的 kafka-clients 包
```

**优点**: 可能解决兼容性问题
**风险**: 可能影响其他组件
**时间**: 中

#### 方案 4: 从备份恢复

**前提**: 系统有可用的备份

**操作**:
1. 停止所有服务
2. 恢复 `/usr/share/server/lib/` 目录
3. 恢复配置文件
4. 重启服务

**优点**: 可靠
**风险**: 需要确认备份可用
**时间**: 短

### 6.2 临时应急方案

如果系统必须立即恢复，可以考虑：

**方案 A: 禁用 Kafka 功能**

1. 修改配置，临时禁用依赖 Kafka 的模块
2. 启动应用的核心功能
3. 后续再修复 Kafka 问题

**方案 B: 使用替代消息队列**

如果有其他消息队列实现（如系统内置的），可以临时切换。

---

## 七、关键文件清单

### 7.1 配置文件

| 文件路径 | 说明 |
|---------|------|
| `/etc/nginx/sites-available/default` | Nginx 配置 |
| `/data/sw/configuration.json` | 应用配置 |
| `/usr/share/kafka/config/server.properties` | Kafka 配置 |
| `/etc/mysql/my.cnf` | MySQL 配置 |
| `/lib/systemd/system/middleware.service` | Systemd 服务定义 |

### 7.2 应用文件

| 目录/文件 | 说明 |
|----------|------|
| `/usr/share/nginx/html/middleware_ui/` | 前端静态文件 |
| `/usr/share/server/lib/` | Java 应用依赖库 |
| `/usr/share/server/lib/server-*.jar` | 主应用 jar |
| `/usr/share/middleware_setup/` | Python 中间件 |
| `/data/sw/` | 应用数据目录 |

### 7.3 日志文件

| 文件路径 | 说明 |
|----------|------|
| `/preserve/logs/` | 应用日志目录 |
| `/var/log/nginx/` | Nginx 日志 |
| `/var/log/syslog` | 系统日志 |
| `journalctl -u middleware.service` | middleware 服务日志 |

---

## 八、预防措施

### 8.1 监控建议

1. **应用健康检查**
   ```bash
   # 添加到监控脚本
   curl -k https://127.0.0.1/api/health
   ```

2. **服务状态监控**
   ```bash
   systemctl is-active middleware.service
   ```

3. **日志监控**
   ```bash
   # 监控错误日志
   tail -f /preserve/logs/*.log | grep -i error
   ```

### 8.2 依赖管理建议

1. **建立依赖清单**
   - 记录所有第三方库的版本
   - 定期检查安全更新

2. **版本锁定**
   - 使用 Maven/Gradle 依赖管理
   - 锁定关键库的版本

3. **变更管理**
   - 任何升级前先测试
   - 保留回滚方案

---

## 九、技术附录

### 9.1 Kafka 版本兼容性

| Kafka 版本 | Serializer API | 状态 |
|------------|----------------|------|
| 0.8.x | kafka.serializer.Encoder | ❌ 已废弃 |
| 0.9.x | org.apache.kafka.common.serialization.Serializer | ✅ 推荐 |
| 0.10.x+ | org.apache.kafka.common.serialization.Serializer | ✅ 当前 |

### 9.2 常用诊断命令

```bash
# 检查 Java 类路径中的 jar
ps aux | grep java | tr ' ' '\n' | grep jar

# 查找特定的类
find /usr/share/server/lib -name "*.jar" -exec jar tf {} \; | grep Encoder

# 测试 Kafka 连接
echo "test" | kafkacat -P -t test

# 检查服务依赖
systemctl list-dependencies middleware.service

# 查看最近的系统变化
grep -i error /var/log/syslog | tail -50
```

### 9.3 快速诊断流程图

```
系统异常
    ↓
检查 Nginx (✅ 正常)
    ↓
检查前端页面 (✅ 可访问)
    ↓
检查 API 调用 (❌ 失败)
    ↓
检查 middleware.service (❌ 未运行)
    ↓
查看服务日志 (发现 Kafka 错误)
    ↓
确认根因: Kafka API 不兼容
```

---

## 十、联系和支持

### 10.1 产品信息

- **产品**: SecSmart 网络威胁监测系统
- **版本**: 3.0.3-ISA-dev-SNAPSHOT
- **厂商**: SecSmart (智网安科)

### 10.2 相关资源

- 应用日志: `/preserve/logs/`
- 服务管理: `systemctl status middleware.service`
- 配置管理: `/data/sw/configuration.json`

---

## 十一、更新日志

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-01-12 | 1.0 | 初始诊断报告 |

---

**报告生成日期**: 2026-01-12
**诊断人员**: Claude Code
**报告状态**: 待修复
