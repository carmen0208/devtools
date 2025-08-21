### 2. 发送代币时的配置流程

当你发送代币时，配置会按以下顺序起作用：

#### **第一步：对等节点验证** (`configureOAppPeers`)
```typescript
// 检查源网络是否信任目标网络
const hasPeer = await sdk.hasPeer(to.eid, to.address)
if (!hasPeer) {
    // 建立信任关系
    return [await sdk.setPeer(to.eid, to.address)]
}
```
**作用**: 确保源网络上的 OApp 信任目标网络上的 OApp，这是跨链通信的前提。

#### **第二步：发送库配置** (`configureSendLibraries`)
```typescript
// 设置消息发送时使用的库
return [await endpointSdk.setSendLibrary(from.address, to.eid, config.sendLibrary)]
```
**作用**: 指定哪个库负责处理消息的发送逻辑，包括：
- 消息打包
- 费用计算
- 发送确认

#### **第三步：发送配置** (`configureSendConfig`)
这是**最关键的配置**，包含两个重要部分：

##### **A. Executor 配置** (`executorConfig`)
```typescript
configType: CONFIG_TYPE_EXECUTOR = 1
```
**作用**: 配置消息在目标网络上的**执行逻辑**
- **Gas 限制**: 目标网络执行时的 gas 上限
- **Value 设置**: 随消息发送的原生代币数量
- **执行顺序**: 消息处理的优先级
- **错误处理**: 执行失败时的回滚策略

##### **B. ULN 配置** (`ulnConfig`)
```typescript
configType: CONFIG_TYPE_ULN = 2
```
**作用**: 配置**Universal LayerZero Network** 参数
- **验证器设置**: 哪些验证器负责验证消息
- **确认阈值**: 需要多少个验证器确认
- **超时设置**: 消息验证的超时时间
- **费用分配**: 验证费用的分配方式

### 3. 具体配置示例分析

让我用一个具体的代币发送场景来说明：

```typescript
connections: [
  {
    from: { eid: 101, address: "0x123..." },  // Ethereum 上的 OApp
    to: { eid: 102, address: "0x789..." },    // BSC 上的 OApp
    config: {
      sendLibrary: "0xabc...",  // 发送库
      sendConfig: {
        executorConfig: {
          gas: 500000n,         // 目标网络执行 gas 限制
          value: 0n             // 不发送原生代币
        },
        ulnConfig: {
          // ULN 网络配置
          confirmations: 3,     // 需要3个验证器确认
          timeout: 3600n        // 1小时超时
        }
      }
    }
  }
]
```

### 4. 发送代币时的完整流程

#### **1. 消息发送阶段**
```typescript
// 使用配置的发送库
const sendLibrary = await endpointSdk.getSendLibrary(from.address, to.eid)
// 发送库处理消息打包和发送
```

#### **2. 网络传输阶段**
```typescript
// ULN 网络使用配置的验证器
// 等待足够的确认数量
// 处理超时情况
```

#### **3. 目标网络执行阶段**
```typescript
// 使用配置的执行器参数
// 在 gas 限制内执行
// 处理 value 转账
```

### 5. 关键配置的作用机制

#### **Executor 配置的作用**
```typescript
// 在目标网络上执行时
if (gasUsed > executorConfig.gas) {
    // 交易失败，回滚
}
if (msg.value != executorConfig.value) {
    // 验证 value 匹配
}
```

#### **ULN 配置的作用**
```typescript
// 消息验证过程
const confirmations = await uln.getConfirmations(messageId)
if (confirmations < ulnConfig.confirmations) {
    // 等待更多确认
}
if (timeElapsed > ulnConfig.timeout) {
    // 超时处理
}
```

### 6. 配置的层次结构

```
OApp 配置
├── 网络层 (Peers)
│   └── 建立信任关系
├── 库层 (Libraries)
│   ├── 发送库 (消息发送逻辑)
│   └── 接收库 (消息接收逻辑)
├── 执行层 (Executor)
│   ├── Gas 管理
│   ├── Value 处理
│   └── 执行策略
└── 网络层 (ULN)
    ├── 验证器管理
    ├── 确认机制
    └── 费用分配
```

### 7. 实际应用场景

#### **场景1: 高价值代币传输**
```typescript
executorConfig: {
  gas: 1000000n,      // 高 gas 限制确保执行成功
  value: 0n           // 不发送原生代币
},
ulnConfig: {
  confirmations: 5,   // 更多确认提高安全性
  timeout: 7200n      // 更长超时时间
}
```

#### **场景2: 快速小额传输**
```typescript
executorConfig: {
  gas: 200000n,       // 较低 gas 限制
  value: 0n
},
ulnConfig: {
  confirmations: 2,   // 较少确认提高速度
  timeout: 1800n      // 较短超时
}
```

### 8. 总结

这些配置共同构建了一个**完整的跨链通信系统**：

- **Peers**: 建立网络间的信任关系
- **Libraries**: 提供消息处理的具体实现
- **Executor**: 控制目标网络的执行环境
- **ULN**: 管理跨链消息的验证和确认

当你发送代币时，系统会按照这些配置：
1. 验证目标网络是否可信
2. 使用指定的发送库处理消息
3. 按照 ULN 配置进行跨链验证
4. 在目标网络按照执行器配置执行接收逻辑

这样确保了跨链代币传输的安全性、可靠性和效率。