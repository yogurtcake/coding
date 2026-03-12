# 富途API接入指南

## 概述

本系统支持与富途证券API对接，实现资产数据的自动同步。请注意：**本系统仅用于资产统计和查看，不支持任何交易操作**。

## 接入方式

### 方式一：富途OpenAPI（推荐）

富途提供官方OpenAPI，支持获取账户资产、持仓、订单等信息。

#### 1. 申请API权限

1. 登录富途牛牛APP或网页版
2. 进入「我的」→「设置」→「API管理」
3. 申请OpenAPI权限
4. 获取API Key和Secret

#### 2. 配置步骤

在系统设置中填写以下信息：

```json
{
  "apiKey": "your_api_key_here",
  "apiSecret": "your_api_secret_here",
  "region": "hk",  // 或 "us" 美股
  "accountType": "individual"  // 个人账户
}
```

#### 3. 支持的API接口

| 接口 | 功能 | 频率限制 |
|------|------|----------|
| `/v1/account/cash` | 获取现金信息 | 10次/秒 |
| `/v1/account/portfolio` | 获取持仓信息 | 10次/秒 |
| `/v1/account/orders` | 获取订单历史 | 5次/秒 |
| `/v1/account/trades` | 获取成交记录 | 5次/秒 |
| `/v1/account/assets` | 获取总资产 | 10次/秒 |

### 方式二：富途Moomoo API

Moomoo（富途国际版）也提供类似的API服务。

#### 申请步骤

1. 登录Moomoo APP
2. 进入「Account」→「Settings」→「API」
3. 创建API Key
4. 配置Webhook回调（可选）

### 方式三：手动导入（备用方案）

如果无法申请API，支持手动导入数据：

1. 从富途牛牛导出CSV交易记录
2. 在系统中选择「导入数据」
3. 上传CSV文件
4. 系统自动解析并更新数据

## 安全性说明

### API密钥存储

- API密钥使用AES-256加密存储
- 密钥仅保存在本地浏览器localStorage
- 不会上传到任何服务器
- 建议定期更换API密钥

### 数据传输

- 所有API请求通过HTTPS加密传输
- 支持IP白名单限制
- 支持设置API调用频率限制

### 权限控制

- 仅申请只读权限（read-only）
- 不申请交易权限
- 不申请资金转出权限

## 配置示例

### 前端配置

```typescript
// src/config/futu.ts
export const futuConfig = {
  // 富途香港API
  hk: {
    baseUrl: 'https://openapi.futunn.com',
    apiKey: process.env.FUTU_API_KEY_HK,
    apiSecret: process.env.FUTU_API_SECRET_HK,
  },
  // 富途美国API (Moomoo)
  us: {
    baseUrl: 'https://openapi.moomoo.com',
    apiKey: process.env.FUTU_API_KEY_US,
    apiSecret: process.env.FUTU_API_SECRET_US,
  }
};
```

### 数据同步服务

```typescript
// src/services/futuApi.ts
class FutuApiService {
  async syncAccountData(accountId: string) {
    const [cash, portfolio, orders] = await Promise.all([
      this.getCash(accountId),
      this.getPortfolio(accountId),
      this.getOrders(accountId),
    ]);
    
    // 更新本地数据
    await this.updateLocalData({ cash, portfolio, orders });
  }
  
  async getCash(accountId: string) {
    // 调用富途API获取现金信息
  }
  
  async getPortfolio(accountId: string) {
    // 调用富途API获取持仓信息
  }
  
  async getOrders(accountId: string) {
    // 调用富途API获取订单历史
  }
}
```

## 常见问题

### Q: 申请API需要什么条件？
A: 需要完成实名认证，账户资产达到一定门槛（通常10万港币以上）。

### Q: API调用频率有限制吗？
A: 是的，不同接口有不同的频率限制，详见上方表格。

### Q: 数据同步有延迟吗？
A: API数据通常有15分钟延迟，实时数据需要申请更高权限。

### Q: 可以同时接入多个账户吗？
A: 可以，系统支持同时接入多个富途账户。

### Q: API密钥泄露怎么办？
A: 立即在富途后台撤销该密钥，重新生成新的密钥。

## 技术支持

- 富途OpenAPI文档：https://www.futunn.com/OpenAPI
- Moomoo API文档：https://www.moomoo.com/api
- 技术支持邮箱：api@futunn.com
