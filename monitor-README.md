# 设备监控告警系统

系统监控设备（桌面云、笔记本、台式机）的CPU、内存、磁盘使用率、运行时长等指标，当指标超过阈值时通过企业微信发送告警通知。

## 系统架构

### 核心模块

1. **监控数据接收** (`Monitor.java`)
   - 接收PowerShell脚本上报的设备监控数据
   - RESTful API: `POST /api/metrics`
   - 数据保存到MySQL数据库

2. **设备-人员映射管理**
   - 通过Excel文件维护设备MAC地址与企微用户ID的映射关系
   - 定时任务自动同步Excel数据到数据库
   - Excel文件路径：`D:/device-mapping.xlsx`（可在配置文件中修改）

3. **告警扫描与发送** (`AlertScheduler.java`)
   - 定时扫描数据库中的设备监控数据
   - 根据配置的告警规则判断是否需要发送告警
   - 通过企业微信API发送告警消息给对应用户

4. **企业微信集成** (`WeComService.java`)
   - 获取企业微信Access Token
   - 发送文本消息给指定用户
   - 支持重试机制

### 数据流程

```
PowerShell脚本 -> Monitor API -> MySQL数据库
                                         ↓
Excel文件 -> ExcelSyncScheduler -> 设备映射表
                                         ↓
                            AlertScheduler扫描
                                         ↓
                                企微消息发送
```

## 数据库设计

### 表结构

#### 1. device_user_mapping（设备-人员关联表）
- `id`: 主键
- `mac_address`: MAC地址（唯一索引）
- `wecom_userid`: 企微用户ID
- `remark`: 备注
- `create_time`: 创建时间
- `update_time`: 更新时间

#### 2. metric_record（监控数据记录表）
- `id`: 主键
- `mac_address`: MAC地址（有索引）
- `computer_name`: 计算机名称
- `user_name`: 用户名
- `disk_usage`: 磁盘使用率(%)
- `c_total_gb`: C盘总容量(GB)
- `c_used_gb`: C盘已用容量(GB)
- `c_free_gb`: C盘剩余容量(GB)
- `cpu_usage`: CPU使用率(%)
- `ram_usage`: 内存使用率(%)
- `uptime_hours`: 运行时长(小时)
- `create_time`: 创建时间

#### 3. alert_history（告警历史记录表）
- `id`: 主键
- `mac_address`: MAC地址
- `computer_name`: 计算机名称
- `user_name`: 用户名
- `wecom_userid`: 企微用户ID
- `alert_type`: 告警类型（UPTIME/DISK）
- `alert_content`: 告警内容
- `send_status`: 发送状态（0-失败，1-成功）
- `retry_count`: 重试次数
- `error_msg`: 错误信息
- `create_time`: 创建时间

## 配置说明

### application.yml配置项

#### 数据库配置
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/monitor_db?useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: 123456
```

#### Excel文件配置
```yaml
excel:
  file-path: D:/device-mapping.xlsx  # Excel文件路径
  sync-cron: "0 */1 * * * ?"  # 同步频率（每5分钟）
```

#### 企业微信配置
```yaml
wecom:
  corp-id: wwxxxxxx  # 企业ID
  corp-secret: xxxxxx  # 应用密钥
  agent-id: 1000002  # 应用ID（AgentId）
  api-base-url: https://qyapi.weixin.qq.com
```

#### 告警规则配置
```yaml
alert:
  rules:
    uptime:
      enabled: true  # 是否启用开机时长告警
      max-days: 1  # 超过多少天未关机告警
    disk:
      enabled: true  # 是否启用磁盘告警
      threshold: 30  # 磁盘使用率阈值(%)
  deduplication:
    hours: 24  # 去重时间窗口（小时）
  scan-cron: "0 */1 * * * ?"  # 告警扫描频率（每5分钟）
```

### Excel文件格式

Excel文件必须包含以下列（按顺序）：

| MAC地址 | 企微用户ID | 用户名 | 备注 |
|---------|-----------|--------|------|
| 00-1B-44-11-3A-B7 | zhangsan | 张三 | 示例数据1 |
| 00-1B-44-11-3A-B8 | lisi | 李四 | |
| 00-1B-44-11-3A-B9 | wangwu | 王五 | 已离职 |

**字段说明：**
- **MAC地址**：设备的MAC地址，用于匹配监控数据。如果MAC地址为"UNKNOWN"或空，该记录将被跳过
- **企微用户ID**：企业微信用户ID，用于发送告警消息。如果为空，该设备不会收到告警
- **用户名**：用户名
- **备注**：备注信息（可选）

**使用说明：**
- 用户入职分配电脑：在Excel中新增一行，填写MAC地址和企微用户ID
- 用户离职：直接从Excel中删除对应的记录行
- 系统以Excel为准，定时同步到数据库。Excel中有记录的设备才会收到告警，删除记录后下次同步该设备将不再收到告警

## API接口

### POST /api/metrics

接收设备监控数据上报。

**请求体示例：**
```json
{
  "macAddress": "00-1B-44-11-3A-B7",
  "computerName": "PC-001",
  "userName": "张三",
  "diskUsage": 75.5,
  "cTotalGB": 500.0,
  "cUsedGB": 377.5,
  "cFreeGB": 122.5,
  "cpuUsage": 45.2,
  "ramUsage": 60.8,
  "uptimeHours": 120.5
}
```

**字段说明：**
- `macAddress`：MAC地址。如果为"UNKNOWN"或空，数据将被拒绝保存
- `computerName`：计算机名称
- `userName`：用户名
- `diskUsage`：磁盘使用率(%)
- `cTotalGB`：C盘总容量(GB)
- `cUsedGB`：C盘已用容量(GB)
- `cFreeGB`：C盘剩余容量(GB)
- `cpuUsage`：CPU使用率(%)
- `ramUsage`：内存使用率(%)
- `uptimeHours`：运行时长(小时)

## 告警规则

### 开机时长告警
- 当设备连续运行时间超过配置的`max-days`天数时，发送告警
- 告警内容：提示用户重启电脑以维持性能

### 磁盘使用率告警
- 当磁盘使用率超过配置的`threshold`阈值时，发送告警
- 告警内容：提示用户清理磁盘空间

### 告警去重机制
- 相同设备、相同告警类型在配置的`deduplication.hours`时间窗口内只发送一次
- 避免重复告警骚扰用户

## 部署说明

### 1. 数据库初始化

执行 `src/main/resources/db/schema.sql` 创建数据库表。

### 2. 配置文件

修改 `application.yml` 中的配置项：
- 数据库连接信息
- Excel文件路径
- 企业微信配置（corp-id、corp-secret、agent-id）
- 告警规则阈值

### 3. 准备Excel文件

在指定路径（默认`D:/device-mapping.xlsx`）创建Excel文件，按照格式填入设备MAC地址和企微用户ID。

### 4. 企业微信配置

1. 登录企业微信管理后台
2. 创建应用，获取`corp-id`、`corp-secret`、`agent-id`
3. 配置IP白名单（如果企业微信要求）
4. 将配置信息填入`application.yml`

### 5. 启动应用

```bash
mvn spring-boot:run
```

或打包后运行：

```bash
mvn clean package
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

## PowerShell脚本示例

```powershell
# 获取MAC地址
$macAddress = (Get-NetAdapter | Where-Object {$_.Status -eq "Up"} | Select-Object -First 1).MacAddress
if (-not $macAddress) {
    $macAddress = "UNKNOWN"
}

# 获取监控数据
$disk = Get-WmiObject Win32_LogicalDisk -Filter "DeviceID='C:'"
$diskUsage = [math]::Round((($disk.Size - $disk.FreeSpace) / $disk.Size) * 100, 2)

$cpuUsage = (Get-Counter '\Processor(_Total)\% Processor Time').CounterSamples.CookedValue
$ramUsage = (Get-Counter '\Memory\Available MBytes').CounterSamples.CookedValue
$totalRam = (Get-WmiObject Win32_ComputerSystem).TotalPhysicalMemory / 1GB
$ramUsagePercent = [math]::Round((($totalRam - ($ramUsage / 1024)) / $totalRam) * 100, 2)

$uptime = (Get-Uptime).TotalHours

# 构建JSON数据
$data = @{
    macAddress = $macAddress
    computerName = $env:COMPUTERNAME
    userName = $env:USERNAME
    diskUsage = $diskUsage
    cTotalGB = [math]::Round($disk.Size / 1GB, 2)
    cUsedGB = [math]::Round(($disk.Size - $disk.FreeSpace) / 1GB, 2)
    cFreeGB = [math]::Round($disk.FreeSpace / 1GB, 2)
    cpuUsage = [math]::Round($cpuUsage, 2)
    ramUsage = $ramUsagePercent
    uptimeHours = [math]::Round($uptime, 2)
} | ConvertTo-Json

# 发送到API
Invoke-RestMethod -Uri "http://your-server:9090/api/metrics" -Method Post -Body $data -ContentType "application/json"
```

## 注意事项

1. **MAC地址处理**：系统会自动过滤MAC地址为"UNKNOWN"或空的监控数据，不会保存到数据库
2. **Excel同步**：定时任务会根据配置的频率自动同步Excel文件到数据库。如果Excel中的MAC地址为"UNKNOWN"或空，该记录会被跳过
3. **告警匹配**：告警匹配使用MAC地址，不依赖计算机名称或用户名。确保Excel中的MAC地址与脚本上报的MAC地址一致
4. **企业微信IP白名单**：如果企业微信配置了IP白名单，需要将服务器IP添加到白名单中
5. **数据存储**：所有监控数据都会保存到数据库，可用于历史数据分析。数据量大时注意数据库性能和存储空间

## 常见问题

### 1. 企业微信消息发送失败（errcode=40013）
- 检查`corp-id`是否正确
- 确认企业微信应用已创建并获取到正确的ID

### 2. 企业微信消息发送失败（errcode=60020）
- 检查服务器IP是否在企业微信的IP白名单中
- 登录企业微信管理后台，在应用设置中添加服务器IP

### 3. 告警未发送
- 检查Excel中的企微用户ID是否正确
- 确认Excel中有该设备的记录
- 查看日志确认是否有错误信息

### 4. MAC地址为UNKNOWN
- 如果脚本无法获取MAC地址，会使用"UNKNOWN"
- 系统会自动过滤这些数据，不会保存和告警
- 需要修复脚本中的MAC地址获取逻辑

## 技术栈

- Spring Boot 3.5.4
- MyBatis
- MySQL
- EasyExcel
- Hutool
- Lombok
