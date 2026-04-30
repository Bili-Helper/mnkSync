# mnkSync — RocketMQ + Raw Input 的多机键鼠同步器（喀拉壁球实机验证）

本项目是一套基于 **RocketMQ 广播通信**与 **Windows 原始输入（Raw Input）/DirectInput 采集**的网络键鼠同步器：  
将主控机的键盘与鼠标**相对位移/按键事件**实时发布到消息队列，使多台从机在不同账号/不同客户端中执行一致输入，从而实现“一套键鼠同时控制多台电脑”的同步操作。

> 适用场景：窗口锁鼠标/游戏相对位移输入（如第一人称/战术射击/需捕获 Raw Mouse Delta 的游戏）。  
> 针对“喀拉壁球”场景完成关键改造与实机测试。

## 实机演示（多机多账号同步）
- 测试视频
  
  https://github.com/user-attachments/assets/02ae7243-8aaa-44f2-bbdd-68bb8e3535b3

---

## 核心特性（基于本仓库实际改动）
### 1) Raw Mouse 相对位移同步（解决锁鼠标/回正问题）
传统 `WH_MOUSE_LL` 低级钩子读取的是**屏幕坐标**，遇到游戏“鼠标回正到屏幕中心/隐藏光标”会导致同步失真。  
本项目已将 Master 鼠标采集替换为 **JInput DirectAndRawInputEnvironmentPlugin**（`RawMouseInputJInput`），直接读取鼠标 **相对移动量 deltaX/deltaY**，从根源上适配游戏输入模型。

- Master：`Client/DNFMaster.java` 使用 `Hooker/RawMouseInputJInput`
- MouseMessage 扩展：新增 `deltaX/deltaY/wheelDelta/isRelativeMode`
- Slave：`MessageConsumer/MouseMessageConsumer.java` 支持相对移动注入（`SendInput` + dx/dy）

### 2) RocketMQ 一对多广播同步（顺序消费保证输入有序）
键鼠事件天然有序（尤其鼠标拖拽、按下/抬起序列），本项目通过 RocketMQ：
- Master 发布 KeyboardActions / MouseActions
- Slave 广播订阅并顺序消费，避免并发打乱输入序列

### 3) 低延迟与线程隔离
- 鼠标事件采用 `producer.sendOneway()` 单向发送，降低阻塞对采集线程的影响
- 监听线程独立运行，并在 Master 增加 shutdown hook 做资源回收

---

## 架构概览

```
[Master]
  RawMouseInputJInput / Keyboard Hook
        |
        v
   RocketMQ Topics (MouseActions / KeyboardActions / ControlMessage)
        |
        v
[Slave x N]
  顺序消费 -> 反序列化 -> SendInput 注入
```

---

## 快速开始

### 0) 环境要求
- Windows（输入采集与注入依赖 Win32）
- Java 8+
- RocketMQ（建议 Docker 部署）

### 1) 部署 RocketMQ（Docker）
仓库内已提供部署笔记：`rocketmq_delopy.md`（按环境可改 IP/路径）。

### 2) 配置
配置文件示例（本仓库已新增）：
- `config.properties` 或 `src/main/java/Client/config.properties`

字段：
- `NameServerAddr = <ip>:9876`
- `SlaveName = <slave-id>`

### 3) 启动顺序（推荐）
1. 启动 RocketMQ NameServer + Broker + Dashboard
2. Master 机器启动 `DNFMaster`（主控端采集并发布）
3. 各 Slave 机器启动 `DNFSlave`（订阅并注入）
4. 使用 Dashboard 向 `ControlMessage` 发送 tag：
   - `enable`：开启同步
   - `disable`：关闭同步

---

## 关键实现说明（面向开发者）

### A. 为什么要引入 JInput？
- 游戏常使用 **Raw Input/DirectInput** 获取相对位移，而不是读屏幕坐标
- `WH_MOUSE_LL` 在“鼠标回正/锁定中心”的场景下很难还原真实相对位移
- JInput 的 Direct+Raw 插件能提供更贴近游戏输入的 delta 数据

### B. MouseMessage 扩展字段
为适配相对位移注入，本仓库在 `Entity/MouseMessage` 中新增：
- `deltaX`, `deltaY`：相对位移
- `wheelDelta`：滚轮
- `isRelativeMode`：相对模式标记（用于区分坐标模式/相对模式）

### C. 从机注入策略
`MouseMessageConsumer` 对 `MOUSEEVENTF_MOVE`：
- 若存在 `deltaX/deltaY`：使用 `SendInput` 进行相对移动注入
- 否则回退到 `SetCursorPos(x,y)` 的坐标模式

---

## 注意事项 / 风险提示
- 本项目用于输入同步研究与测试，请自行评估目标软件/游戏的用户协议与风险。
- 多机同步可能触发目标软件的行为检测；请谨慎使用并控制频率。
- `target/native-libs/*.dll` 属于 JInput 原生依赖，建议后续改为打包阶段自动生成/拷贝，避免将二进制直接提交进仓库。

---

## 相关文件索引（便于阅读代码）
- Master 入口：`src/main/java/Client/DNFMaster.java`
- Slave 入口：`src/main/java/Client/DNFSlave.java`
- Raw Mouse 采集：`src/main/java/Hooker/RawMouseInputJInput.java`
- 消费与注入：`src/main/java/MessageConsumer/MouseMessageConsumer.java`
- RocketMQ 部署：`rocketmq_delopy.md`
