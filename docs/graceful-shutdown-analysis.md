# Graceful Shutdown 功能分析

## 配置文件示例

在 `fly.toml` 中添加以下配置：

```toml
[deploy]
  strategy = "bluegreen"

  [deploy.graceful_shutdown]
    enabled = true
    endpoint = "/shutdown"  # 必填：graceful shutdown 的 HTTP 端点
    port = 8080            # 可选：默认使用应用的 internal_port
    interval = "10s"       # 可选：轮询间隔，默认 10 秒
    timeout = "5m"         # 可选：超时时间，默认使用部署的 wait_timeout
```

### 配置字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enabled` | bool | false | 是否启用优雅关闭 |
| `endpoint` | string | - | **必填**，应用的优雅关闭端点（如 `/shutdown`） |
| `port` | int | internal_port | 应用监听端口，默认使用 `http_service.internal_port` |
| `interval` | duration | 10s | 轮询检查间隔 |
| `timeout` | duration | deploy.wait_timeout | 最长等待时间 |

## 工作原理：触发 + 轮询 + Machine State 优先验证

### 核心流程

优雅关闭分为**两个步骤**：

#### 步骤 1：触发优雅关闭（Trigger）

向应用的 `/shutdown` 端点发送 GET 请求，通知应用开始优雅关闭。

- 接受 HTTP 200 或 100 作为成功响应
- 如果触发失败，仅记录警告，继续进入轮询阶段

#### 步骤 2：轮询等待完成（Poll）

定期检查应用状态，**优先使用 Machine State API**，确认机器已完全停止。

```
每个轮询周期（默认 10 秒）：
  1. 【优先】检查 Machine State（通过 Flaps API）
     - stopped/stopping → ✅ 立即完成，停止轮询
     - started → 继续检查 HTTP 端点

  2. 检查 HTTP 端点（仅在 Machine 仍运行时）
     - 200/100 → 继续轮询（应用仍在运行/处理中）
     - 连接失败 → 双重验证 Machine State
       ├─ stopped → ✅ 确认完成
       └─ started → ⚠️ 标记为 shutting-down，继续轮询
     - 其他状态码 → 双重验证 Machine State
       ├─ stopped → ✅ 确认完成
       └─ started → 标记为 shutting-down，继续轮询
```

### 关键设计原则

1. **Machine State 优先**：每次轮询先检查 Machine State，如果已 stopped 则立即完成，不再进行 HTTP 检查
2. **双重验证**：HTTP 失败时必须验证 Machine State，避免网络问题导致误判
3. **容错处理**：触发失败不影响后续轮询，确保可靠性

### 状态判断矩阵

| Machine State | HTTP 响应 | 结果 | 说明 |
|--------------|----------|------|------|
| stopped/stopping | - | ✅ **立即完成** | Machine 已关闭，无需 HTTP 检查 |
| started | 200/100 | ⏳ 继续轮询 | 应用正常运行或正在处理 |
| started | 连接失败 | ⚠️ 继续轮询 | 可能是网络问题或应用正在关闭 |
| started | 其他状态码 | ⏳ 继续轮询 | 非预期状态 |
| - (API 失败) | 连接失败 | ⚠️ 继续轮询 | 无法确认，谨慎处理 |
| - (API 失败) | 200/100 | ⏳ 继续轮询 | 应用仍在运行 |

## 完整的部署流程

在 Blue-Green 部署中，优雅关闭的执行位置：

```
1. 创建 Green Machines                    [CreateGreenMachines]
   ↓
2. 等待 Green Machines 启动              [WaitForGreenMachinesToBeStarted]
   ↓
3. 等待 Green Machines 健康检查通过      [WaitForGreenMachinesToBeHealthy]
   ↓
4. 标记 Green Machines 就绪，接收流量    [MarkGreenMachinesAsReadyForTraffic]
   ↓
5. 标记 Blue Machines 为可安全删除       [TagBlueMachinesAsSafeForDeletion]
   ↓
6. 等待 10 秒（让 fly-proxy 发现新机器）  [waitBeforeCordon]
   ↓
7. 隔离 Blue Machines（停止接收新流量）  [CordonBlueMachines]
   ↓
8. 等待 10 秒（让 fly-proxy 忘记旧机器）  [waitBeforeStop]
   ↓
9. 停止 Blue Machines（发送 SIGTERM）    [StopBlueMachines]
   ↓
10. 🔥 执行优雅关闭等待 🔥                [WaitForGracefulShutdown]  ← 在这里！
   ↓
11. 等待 Blue Machines 完全停止           [WaitForBlueMachinesToBeStopped]
   ↓
12. 销毁 Blue Machines                    [DestroyBlueMachines]
```

### 关键时间点

- **Cordon 后**：Blue Machines 不再接收新请求，但仍在处理现有连接
- **Stop 后**：Blue Machines 收到 SIGTERM 信号，应用开始优雅关闭
- **优雅关闭期间**：
  1. 触发 `/shutdown` 端点，通知应用开始清理
  2. 轮询检查 Machine State（优先）和 HTTP 端点
  3. 直到 Machine State 显示 stopped，才认为完成

## 代码实现详解

### 1. 触发优雅关闭（`WaitForGracefulShutdown` L1088-1116）

```go
// Step 1: Trigger graceful shutdown
triggerCtx, triggerCancel := context.WithTimeout(ctx, 30*time.Second)
defer triggerCancel()

req, err := http.NewRequestWithContext(triggerCtx, "GET", url, http.NoBody)
if err != nil {
    return fmt.Errorf("failed to create request: %w", err)
}

resp, err := httpClient.Do(req)
if err != nil {
    // Log warning but continue with polling
    fmt.Fprintf(bg.io.ErrOut, "  Warning: failed to trigger graceful shutdown for machine %s: %v\n", bg.colorize.Bold(machineID), err)
} else {
    resp.Body.Close()
    // Accept 100 (Continue) or 200 as success
    if resp.StatusCode != 100 && resp.StatusCode != 200 {
        fmt.Fprintf(bg.io.ErrOut, "  Warning: unexpected status code %d when triggering graceful shutdown for machine %s\n", resp.StatusCode, bg.colorize.Bold(machineID))
    }
}
```

**特点**：
- 30 秒超时
- 失败仅记录警告，不中断流程
- 接受 HTTP 100 或 200 作为成功

### 2. 轮询检查（`pollForGracefulShutdownComplete` L1130-1252）

```go
func pollForGracefulShutdownComplete(...) error {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    checkMachineState := func() (bool, error) {
        machine, err := bg.flaps.Get(waitCtx, bg.app.Name, realMachineID)
        if err != nil {
            return false, err
        }
        // 检查是否已停止
        return machine.State == "stopped" || machine.State == "stopping", nil
    }

    for {
        select {
        case <-ticker.C:
            // 【优先】检查 Machine State
            stopped, err := checkMachineState()
            if err == nil && stopped {
                // Machine 已停止，立即完成，不再检查 HTTP
                machineIDToState[machineID] = "stopped"
                return nil
            }

            // Machine 仍运行，检查 HTTP 端点
            resp, err := httpClient.Do(req)
            if err != nil {
                // 连接失败，双重验证
                stopped, machErr := checkMachineState()
                if machErr == nil && stopped {
                    return nil  // ✅ 确认已停止
                }
                // 可能是网络问题或正在关闭
                machineIDToState[machineID] = "shutting-down"
                continue
            }

            if statusCode == 200 || statusCode == 100 {
                // 应用仍在运行
                continue
            } else {
                // 非预期状态码，双重验证
                stopped, _ := checkMachineState()
                if stopped {
                    return nil  // ✅ 确认已停止
                }
                continue
            }
        }
    }
}
```

### 并发处理（`WaitForGracefulShutdown` L1084-1118）

```go
// 为每个 Blue Machine 并发执行优雅关闭
p := pool.New().
    WithErrors().
    WithFirstError().
    WithMaxGoroutines(bg.maxConcurrent)

for _, mach := range bg.blueMachines {
    p.Go(func() error {
        // 1. 触发优雅关闭
        // 2. 轮询检查 Machine State + HTTP
        return pollForGracefulShutdownComplete(...)
    })
}

return p.Wait()  // 等待所有机器完成
```

### 状态渲染

使用 `renderMachineStates` 实时显示每个 Machine 的状态：

```
Machine 148ed21d791e98 - triggering     # 触发优雅关闭中
Machine 5683064f794098 - checking       # 开始检查
Machine e784e2f94c8d58 - processing     # 处理中（收到 100）
Machine 91857e9e3eee28 - running        # 运行中（收到 200）
Machine 7a92f1c3b45d67 - shutting-down  # 正在关闭（端点不可达）
Machine 2b8c4e5f6a7d89 - stopped        # 已停止（Machine State 确认）
```

### 网络连接

使用 **WireGuard 隧道** 连接到 Fly 私有网络：

```go
// 建立 Agent 客户端
agentClient, err := agent.Establish(ctx, apiClient)

// 连接到组织的隧道
dialer, err := agentClient.ConnectToTunnel(ctx, orgSlug, "", true)

// 使用隧道拨号器
httpClient := &http.Client{
    Transport: &http.Transport{
        DialContext: dialer.DialContext,
    },
}
```

### IPv6 地址处理（`buildGracefulShutdownURL` L1121-1129）

```go
func buildGracefulShutdownURL(privateIP string, port int, endpoint string) string {
    ip := net.ParseIP(privateIP)
    if ip != nil && ip.To4() == nil {
        // IPv6 需要用方括号包裹
        return fmt.Sprintf("http://[%s]:%d%s", privateIP, port, endpoint)
    }
    return fmt.Sprintf("http://%s:%d%s", privateIP, port, endpoint)
}
```

## 应用端实现示例

### Go 应用示例

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "sync/atomic"
    "syscall"
    "time"
)

type Server struct {
    server       *http.Server
    shuttingDown atomic.Bool
    activeConns  atomic.Int32
}

func (s *Server) shutdownHandler(w http.ResponseWriter, r *http.Request) {
    // 触发优雅关闭
    if !s.shuttingDown.Load() {
        s.shuttingDown.Store(true)
        fmt.Println("Graceful shutdown triggered via /shutdown endpoint")

        // 启动优雅关闭流程
        go s.gracefulShutdown()

        // 返回 100 表示已开始处理
        w.WriteHeader(http.StatusContinue)
        fmt.Fprintf(w, "Graceful shutdown initiated")
        return
    }

    // 后续调用：报告关闭进度
    active := s.activeConns.Load()
    if active == 0 {
        // 返回 200 表示可以关闭了
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, "Ready to shutdown, no active connections")
    } else {
        // 返回 100 表示仍在处理
        w.WriteHeader(http.StatusContinue)
        fmt.Fprintf(w, "Still processing %d connections", active)
    }
}

func (s *Server) gracefulShutdown() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
    defer cancel()

    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            // 超时，强制退出
            fmt.Println("Graceful shutdown timeout, forcing exit")
            os.Exit(0)
        case <-ticker.C:
            if s.activeConns.Load() == 0 {
                // 所有连接已处理完毕
                fmt.Println("All connections closed, shutting down server")
                s.server.Shutdown(context.Background())
                os.Exit(0)
            }
        }
    }
}

func (s *Server) handleRequest(w http.ResponseWriter, r *http.Request) {
    if s.shuttingDown.Load() {
        w.WriteHeader(http.StatusServiceUnavailable)
        fmt.Fprintf(w, "Server is shutting down")
        return
    }

    s.activeConns.Add(1)
    defer s.activeConns.Add(-1)

    // 模拟处理
    time.Sleep(100 * time.Millisecond)
    w.WriteHeader(http.StatusOK)
    fmt.Fprintf(w, "Hello World")
}

func main() {
    server := &Server{}

    mux := http.NewServeMux()
    mux.HandleFunc("/shutdown", server.shutdownHandler)
    mux.HandleFunc("/", server.handleRequest)

    server.server = &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    // 处理 SIGTERM（Blue-Green 部署中的停止信号）
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGTERM, syscall.SIGINT)

    go func() {
        <-sigChan
        fmt.Println("Received SIGTERM, starting graceful shutdown")
        server.shuttingDown.Store(true)
        server.gracefulShutdown()
    }()

    fmt.Println("Server starting on :8080")
    if err := server.server.ListenAndServe(); err != http.ErrServerClosed {
        fmt.Printf("Server error: %v\n", err)
    }
}
```

### Node.js 应用示例

```javascript
const http = require('http');

let shuttingDown = false;
let activeConnections = 0;

const server = http.createServer((req, res) => {
    if (req.url === '/shutdown') {
        // 触发优雅关闭
        if (!shuttingDown) {
            shuttingDown = true;
            console.log('Graceful shutdown triggered via /shutdown endpoint');

            // 停止接受新连接
            server.close(() => {
                console.log('Server closed, exiting...');
                process.exit(0);
            });
        }

        // 报告关闭进度
        if (activeConnections === 0) {
            res.writeHead(200);
            res.end('Ready to shutdown, no active connections');
        } else {
            res.writeHead(100);
            res.end(`Still processing ${activeConnections} connections`);
        }
        return;
    }

    // 拒绝新请求
    if (shuttingDown) {
        res.writeHead(503);
        res.end('Server is shutting down');
        return;
    }

    // 处理其他请求
    activeConnections++;
    setTimeout(() => {
        res.writeHead(200);
        res.end('Hello World');
        activeConnections--;
    }, 100);
});

// 处理 SIGTERM
process.on('SIGTERM', () => {
    console.log('Received SIGTERM, starting graceful shutdown');
    shuttingDown = true;

    server.close(() => {
        console.log('Server closed, exiting...');
        process.exit(0);
    });
});

server.listen(8080, () => {
    console.log('Server starting on :8080');
});
```

### 简化版（依赖 Machine State 检测）

如果应用不需要复杂的状态报告，可以简化为：

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
    if (req.url === '/shutdown') {
        res.writeHead(200);
        res.end('Shutting down');

        // 关闭服务器
        server.close(() => {
            console.log('Server closed');
            process.exit(0);
        });
        return;
    }

    res.writeHead(200);
    res.end('Hello World');
});

server.listen(8080);
```

**注意**：简化版依赖 flyctl 的 Machine State 检测来确认关闭完成。

## HTTP 状态码约定

| 状态码 | 含义 | flyctl 行为 |
|-------|------|-----------|
| **200 OK** | 应用正常运行或准备关闭 | 继续轮询 |
| **100 Continue** | 正在处理优雅关闭 | 继续轮询 |
| **连接失败** | 端口已关闭 | 双重验证 Machine State |
| **其他（4xx, 5xx）** | 非预期状态 | 双重验证 Machine State |

## 部署流程示例

```bash
$ fly deploy --strategy bluegreen

Verifying if app can be safely deployed

Creating green machines
  Created machine 148ed21d791e98

Waiting for all green machines to start
  Machine 148ed21d791e98 - started

Waiting for all green machines to be healthy
  Machine 148ed21d791e98 - 1/1 passing

Marking green machines as ready
  Machine 148ed21d791e98 now ready

Checkpointing deployment, this may take a few seconds...

Waiting before cordoning all blue machines

Cordoning blue machines (停止接收新流量)
  Machine 5683064f794098 cordoned

Waiting before stopping all blue machines

Stopping all blue machines (发送 SIGTERM)

Waiting for graceful shutdown of blue machines
  Machine 5683064f794098 - triggering   # 触发 /shutdown
  Machine 5683064f794098 - processing   # 应用处理中
  Machine 5683064f794098 - shutting-down # 端点不可达
  Machine 5683064f794098 - stopped      # Machine State 确认已停止

Waiting for all blue machines to stop
  Machine 5683064f794098 - stopped

Destroying all blue machines
  Machine 5683064f794098 destroyed

Deployment Complete
```

## 总结

### 核心特点

1. **两阶段流程**：触发（Trigger） + 轮询（Poll）
2. **Machine State 优先**：每次轮询先检查 Machine State，如果已 stopped 则立即完成
3. **双重验证**：HTTP 失败时验证 Machine State，避免误判
4. **容错处理**：触发失败不影响轮询，确保可靠性
5. **并发处理**：同时检测多个 Machine
6. **隧道连接**：通过私有网络安全访问应用
7. **实时反馈**：状态变化实时显示

### 判断逻辑总结

```
每个轮询周期：

  1. 优先检查 Machine State
     - stopped/stopping → ✅ 立即完成，停止轮询
     - started → 继续检查 HTTP

  2. 检查 HTTP 端点（仅在 Machine 仍运行时）
     - 200/100 → 继续轮询
     - 连接失败 → 双重验证 Machine State
     - 其他状态码 → 双重验证 Machine State
```

### 执行位置

优雅关闭在 **Stop Blue Machines 之后** 执行：

1. Cordon（隔离流量）
2. Wait（10秒）
3. **Stop（发送 SIGTERM）** ← SIGTERM 信号在这里发送
4. **Graceful Shutdown Wait** ← 优雅关闭在这里
5. Wait for Stopped（等待完全停止）
6. Destroy（销毁机器）

这样设计的好处：
- ✅ 应用已收到 SIGTERM，可以开始清理
- ✅ `/shutdown` 端点通知应用加速关闭流程
- ✅ Machine State 检测可以准确判断是否真正停止
- ✅ 即使触发失败，SIGTERM 也能让应用关闭

---

**文件位置**：`internal/command/deploy/strategy_bluegreen.go`
**核心函数**：
- `WaitForGracefulShutdown()` - L1010-1118：主入口，触发并轮询
- `pollForGracefulShutdownComplete()` - L1130-1252：轮询逻辑，Machine State 优先
- `buildGracefulShutdownURL()` - L1121-1129：构建 HTTP URL（支持 IPv6）

**配置文件**：`internal/appconfig/config.go`
- `GracefulShutdownConfig` - L220-225：配置结构定义
