## LayerZero Devtools 加载 lz-evm-sdk-v2 的完整实现流程

### 1. 入口点：toolbox-hardhat 包

```typescript
// packages/toolbox-hardhat/src/index.ts
extendConfig((config, userConfig) => {
    const layerZero = userConfig.layerZero
    
    // 默认从 lz-evm-sdk-v2 加载部署信息
    const deploymentSourcePackages = layerZero?.deploymentSourcePackages ?? ['@layerzerolabs/lz-evm-sdk-v2']
    
    // 创建配置扩展器
    const withDeployments = withLayerZeroDeployments(...deploymentSourcePackages)
    
    // 应用配置扩展
    const { external } = withArtifacts(withDeployments(userConfig))
})
```

### 2. 核心实现：withLayerZeroDeployments 函数

```typescript
// packages/devtools-evm-hardhat/src/config.ts
export const withLayerZeroDeployments = (...packageNames: string[]) => {
    // 1. 解析包路径到 deployments 目录
    const resolvedDeploymentsDirectories = packageNames
        .map(resolvePackageDirectory)  // 解析包的实际文件系统路径
        .map((resolvedPackagePath) => join(resolvedPackagePath, 'deployments'))

    return (config: HardhatUserConfig) => ({
        ...config,
        external: {
            ...config.external,
            deployments: Object.fromEntries(
                Object.entries(config.networks ?? {}).flatMap(([networkName, networkConfig]) => {
                    const eid = networkConfig?.eid
                    
                    if (eid == null) return []
                    
                    try {
                        // 2. 将 EID 转换为 LayerZero 网络名称
                        const layerZeroNetworkName = endpointIdToNetwork(eid, networkConfig?.isLocalEid)
                        
                        // 3. 构建部署目录路径
                        const layerZeroNetworkDeploymentsDirectories = resolvedDeploymentsDirectories.map(
                            (deploymentsDirectory) => join(deploymentsDirectory, layerZeroNetworkName)
                        )
                        
                        return [[networkName, layerZeroNetworkDeploymentsDirectories]]
                    } catch (error) {
                        return []
                    }
                })
            )
        }
    })
}
```

### 3. EID 到网络名称的映射：endpointIdToNetwork 函数

```typescript
// 来自 @layerzerolabs/lz-definitions 包
const layerZeroNetworkName = endpointIdToNetwork(eid, networkConfig?.isLocalEid)
```

这个函数将 EID（如 `EndpointId.AVALANCHE_V2_TESTNET`）转换为标准的网络名称（如 `"avalanche-testnet"`）。

### 4. 最终的文件路径结构

假设你的配置是：
```typescript
networks: {
    fuji: {
        eid: EndpointId.AVALANCHE_V2_TESTNET
    }
}
```

最终会生成这样的路径映射：
```typescript
external: {
    deployments: {
        "fuji": [
            "/path/to/node_modules/@layerzerolabs/lz-evm-sdk-v2/deployments/avalanche-testnet"
        ]
    }
}
```

### 6. lz-evm-sdk-v2 中的部署文件结构

`@layerzerolabs/lz-evm-sdk-v2` 包内部的结构应该是：
```
@layerzerolabs/lz-evm-sdk-v2/
├── deployments/
│   ├── ethereum-mainnet/
│   │   ├── EndpointV2.json
│   │   └── ...其他合约部署文件
│   ├── avalanche-testnet/
│   │   ├── EndpointV2.json
│   │   └── ...其他合约部署文件
│   └── ...其他网络
```

### 7. 在部署脚本中的使用

```typescript
// 在你的部署脚本中
const endpointV2Deployment = await hre.deployments.get('EndpointV2')
// 这会从外部部署目录中读取 EndpointV2 的部署信息
```

### 总结

整个流程的关键点是：

1. **配置阶段**：`toolbox-hardhat` 在 hardhat 配置加载时自动运行
2. **包解析**：通过 `require.resolve` 找到 `lz-evm-sdk-v2` 包的实际安装路径
3. **EID 映射**：将 hardhat 网络配置中的 `eid` 转换为 LayerZero 标准的网络名称
4. **路径构建**：构建到具体部署文件的完整路径
5. **hardhat-deploy 集成**：通过 hardhat 的 `external.deployments` 配置，让 hardhat-deploy 能够读取这些外部部署文件

这样，开发者就可以在部署脚本中直接使用 `hre.deployments.get('EndpointV2')` 来获取 LayerZero 官方部署的合约地址和 ABI，而不需要手动维护这些信息。


## 与 hardhat-deploy 的集成


## 1. 注入给 Hardhat

```typescript
// packages/toolbox-hardhat/src/index.ts
if (external != null) {
    Object.assign(config, { external })
}
```

## 2. 这是 hardhat-deploy 预留的结构

`external.deployments` 是 **hardhat-deploy** 插件预留的配置结构。

## hardhat-deploy 是如何处理的

### 1. 配置注入阶段

当 `toolbox-hardhat` 运行时，它会：

1. **解析包路径**：找到 `@layerzerolabs/lz-evm-sdk-v2` 包的实际安装路径
2. **构建部署目录路径**：指向 `node_modules/@layerzerolabs/lz-evm-sdk-v2/deployments/{network-name}`
3. **注入到 hardhat 配置**：通过 `Object.assign(config, { external })` 将路径信息注入

### 2. hardhat-deploy 读取阶段

当你的部署脚本调用 `hre.deployments.get('EndpointV2')` 时，hardhat-deploy 会：

1. **检查 external.deployments 配置**：找到当前网络对应的外部部署目录路径
2. **读取部署文件**：从指定的目录中读取 `EndpointV2.json` 文件
3. **返回部署信息**：包含合约地址、ABI、交易哈希等信息

### 3. 具体的文件路径映射

假设你的配置是：
```typescript
networks: {
    fuji: {
        eid: EndpointId.AVALANCHE_V2_TESTNET
    }
}
```

最终会生成这样的配置：
```typescript
config.external = {
    deployments: {
        "fuji": [
            "/path/to/node_modules/@layerzerolabs/lz-evm-sdk-v2/deployments/avalanche-testnet"
        ]
    }
}
```

### 4. 部署文件的结构

`@layerzerolabs/lz-evm-sdk-v2` 包中的部署文件结构：
```
@layerzerolabs/lz-evm-sdk-v2/
├── deployments/
│   ├── ethereum-mainnet/
│   │   ├── EndpointV2.json  ← 包含合约地址、ABI等
│   │   └── ...其他合约
│   ├── avalanche-testnet/
│   │   ├── EndpointV2.json  ← 包含合约地址、ABI等
│   │   └── ...其他合约
│   └── ...其他网络
```

### 5. 工作流程总结

```
1. toolbox-hardhat 启动
   ↓
2. 解析 lz-evm-sdk-v2 包路径
   ↓
3. 将部署目录路径注入到 config.external.deployments
   ↓
4. hardhat-deploy 读取这个配置
   ↓
5. 当调用 hre.deployments.get('EndpointV2') 时
   ↓
6. hardhat-deploy 从外部目录读取部署文件
   ↓
7. 返回合约的部署信息
```

所以整个机制是：
- **LayerZero devtools** 负责配置注入（告诉 hardhat-deploy 去哪里找文件）
- **hardhat-deploy** 负责实际的文件读取和数据处理
- **开发者** 通过标准的 `hre.deployments.get()` API 获取数据

这是一个非常优雅的设计，通过配置注入的方式，让 hardhat-deploy 能够读取外部包中的部署信息，而不需要修改 hardhat-deploy 的核心逻辑。