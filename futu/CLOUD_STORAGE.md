# 云端存储方案

## 概述

本系统支持将数据同步到云端，实现多设备数据同步和备份。我们提供多种云端存储方案，确保您的数据安全。

## 支持的云端存储

### 方案一：Supabase（推荐）

Supabase是开源的Firebase替代品，提供PostgreSQL数据库和身份验证服务。

#### 优势
- 免费额度充足（500MB数据库，2GB存储）
- 数据加密存储
- 支持行级安全策略（RLS）
- 开源可自建

#### 配置步骤

1. 注册Supabase账号：https://supabase.com
2. 创建新项目
3. 在Database中创建以下表：

```sql
-- 用户表
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email TEXT UNIQUE NOT NULL,
  encrypted_password TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

-- 账户数据表
CREATE TABLE accounts (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id),
  broker TEXT NOT NULL,
  account_number TEXT NOT NULL,
  name TEXT,
  data JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- 资产历史表
CREATE TABLE asset_history (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  account_id UUID REFERENCES accounts(id),
  date DATE NOT NULL,
  value DECIMAL(18,4) NOT NULL,
  profit DECIMAL(18,4),
  cash_hkd DECIMAL(18,4),
  cash_usd DECIMAL(18,4),
  created_at TIMESTAMP DEFAULT NOW()
);

-- 持仓表
CREATE TABLE holdings (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  account_id UUID REFERENCES accounts(id),
  code TEXT NOT NULL,
  name TEXT,
  quantity INTEGER NOT NULL,
  avg_cost DECIMAL(18,4),
  current_price DECIMAL(18,4),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- 订单表
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  account_id UUID REFERENCES accounts(id),
  order_id TEXT NOT NULL,
  code TEXT NOT NULL,
  direction TEXT NOT NULL,
  price DECIMAL(18,4),
  quantity INTEGER,
  status TEXT,
  order_time TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);
```

4. 启用行级安全策略：

```sql
-- 用户只能访问自己的数据
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can only access their own accounts" ON accounts
  FOR ALL USING (auth.uid() = user_id);
```

5. 在系统中配置Supabase连接：

```typescript
// src/config/supabase.ts
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = 'https://your-project.supabase.co';
const supabaseKey = 'your-anon-key';

export const supabase = createClient(supabaseUrl, supabaseKey);
```

### 方案二：Firebase

Google Firebase提供实时数据库和身份验证服务。

#### 配置步骤

1. 创建Firebase项目：https://console.firebase.google.com
2. 启用Firestore数据库
3. 配置安全规则：

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    match /accounts/{accountId} {
      allow read, write: if request.auth != null && resource.data.userId == request.auth.uid;
    }
  }
}
```

### 方案三：自建服务器

如果您有自己的服务器，可以部署私有API服务。

#### Docker部署示例

```dockerfile
# Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/futu_stats
      - JWT_SECRET=your-secret-key
    depends_on:
      - db
  
  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=futu_stats
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

## 数据加密方案

### 客户端加密（推荐）

数据在发送到云端之前进行加密，确保云端也无法读取您的敏感信息。

```typescript
// src/utils/encryption.ts
import CryptoJS from 'crypto-js';

const ENCRYPTION_KEY = 'your-master-password'; // 用户设置的加密密码

export function encryptData(data: any): string {
  const jsonString = JSON.stringify(data);
  return CryptoJS.AES.encrypt(jsonString, ENCRYPTION_KEY).toString();
}

export function decryptData(encryptedData: string): any {
  const bytes = CryptoJS.AES.decrypt(encryptedData, ENCRYPTION_KEY);
  const jsonString = bytes.toString(CryptoJS.enc.Utf8);
  return JSON.parse(jsonString);
}
```

### 端到端加密

使用用户的登录密码派生加密密钥，实现端到端加密。

```typescript
// 派生加密密钥
async function deriveKey(password: string, salt: string): Promise<CryptoKey> {
  const encoder = new TextEncoder();
  const keyMaterial = await crypto.subtle.importKey(
    'raw',
    encoder.encode(password),
    'PBKDF2',
    false,
    ['deriveBits', 'deriveKey']
  );
  
  return crypto.subtle.deriveKey(
    {
      name: 'PBKDF2',
      salt: encoder.encode(salt),
      iterations: 100000,
      hash: 'SHA-256'
    },
    keyMaterial,
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt', 'decrypt']
  );
}
```

## 数据同步策略

### 增量同步

只同步变化的数据，减少网络传输。

```typescript
async function syncData() {
  const lastSyncTime = localStorage.getItem('lastSyncTime');
  
  // 获取云端最新数据
  const cloudData = await fetchCloudData(lastSyncTime);
  
  // 获取本地变化
  const localChanges = getLocalChanges(lastSyncTime);
  
  // 合并数据
  const mergedData = mergeData(cloudData, localChanges);
  
  // 上传本地变化
  await uploadChanges(localChanges);
  
  // 更新本地数据
  updateLocalData(mergedData);
  
  // 更新同步时间
  localStorage.setItem('lastSyncTime', Date.now().toString());
}
```

### 冲突解决

当云端和本地数据冲突时，提供多种解决策略：

1. **最新优先**：使用时间戳，保留最新的数据
2. **云端优先**：始终以云端数据为准
3. **本地优先**：始终以本地数据为准
4. **手动合并**：提示用户手动选择

## 隐私保护

### 数据最小化

- 仅存储必要的资产统计数据
- 不存储完整的交易详情
- 敏感信息（如API密钥）本地加密存储

### 匿名化

- 用户可以选择匿名使用
- 不收集个人身份信息
- 分析数据脱敏处理

### 数据删除

- 用户可以随时删除云端数据
- 提供「一键清除所有数据」功能
- 删除后数据不可恢复

## 安全建议

1. **使用强密码**：至少12位，包含大小写字母、数字和特殊字符
2. **启用双因素认证**：为云端账户启用2FA
3. **定期更换API密钥**：建议每3个月更换一次
4. **检查登录历史**：定期查看账户登录记录
5. **备份本地数据**：定期导出数据备份到本地

## 常见问题

### Q: 云端存储安全吗？
A: 我们采用银行级别的加密技术，数据在传输和存储过程中都经过加密。同时支持客户端加密，即使云端服务商也无法读取您的数据。

### Q: 可以离线使用吗？
A: 可以，系统支持离线模式。数据会先保存在本地，联网后自动同步到云端。

### Q: 如何迁移数据到另一个设备？
A: 登录同一个云端账户，数据会自动同步到新设备。或者使用「导出/导入」功能手动迁移。

### Q: 云端存储收费吗？
A: Supabase和Firebase都有免费额度，一般个人使用足够。超出免费额度后按使用量收费。

### Q: 数据会保留多久？
A: 免费账户数据保留1年，付费账户永久保留。长期不登录的账户数据可能会被清理。
