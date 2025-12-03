# 支付接口文档

本文详细介绍支付相关接口的调用方式、公共参数、签名与加密规则，以及创建订单、支付查询、退款、退款结果查询、支付/退款结果通知等业务接口。

## 目录

- [一、接口设计概述](#接口设计概述)
  - [1.1 统一参数结构](#统一参数结构)
  - [1.2 核心安全原则](#核心安全原则)
- [二、签名与加密规则](#签名规则)
  - [2.1 密钥交换与初始化](#0-密钥交换与初始化)
  - [2.2 签名加密方式](#1-签名加密方式)
  - [2.3 商户发送请求流程（发送方）](#2-商户发送请求流程发送方)
  - [2.4 商户接收响应/通知流程（接收方）](#3-商户接收响应通知流程接收方)
  - [2.5 签名加密流程图](#4-签名加密流程图)
  - [2.6 关键点与注意事项](#5-关键点与注意事项)
  - [2.7 完整签名加密流程示例](#6-完整签名加密流程示例)
- [三、接口环境与网关地址](#接口地址)
  - [3.1 测试环境](#测试环境)
  - [3.2 生产环境](#生产环境)
- [四、创建订单接口](#创建订单接口)
  - [4.1 接口信息](#接口信息)
  - [4.2 请求参数说明](#请求参数说明)
  - [4.3 支付方式说明](#支付方式说明)
  - [4.4 请求示例](#请求示例)
  - [4.5 返回参数说明](#返回参数说明)
  - [4.6 返回示例](#返回示例)
  - [4.7 payData 字段说明](#paydata-字段说明)
- [五、支付查询接口](#支付查询接口)
  - [5.1 接口信息](#接口信息-1)
  - [5.2 请求参数说明](#请求参数说明-1)
  - [5.3 请求示例](#请求示例-1)
  - [5.4 返回参数说明](#返回参数说明-1)
  - [5.5 返回示例](#返回示例-1)
- [六、退款接口](#退款接口)
  - [6.1 接口信息](#接口信息-2)
  - [6.2 请求参数说明](#请求参数说明-2)
  - [6.3 请求示例](#请求示例-2)
  - [6.4 返回参数说明](#返回参数说明-2)
  - [6.5 返回示例](#返回示例-2)
- [七、退款结果查询接口](#退款结果查询接口)
  - [7.1 接口信息](#接口信息-3)
  - [7.2 请求参数说明](#请求参数说明-3)
  - [7.3 请求示例](#请求示例-3)
  - [7.4 返回参数说明](#返回参数说明-3)
  - [7.5 返回示例](#返回示例-3)
- [八、支付结果通知接口](#支付结果通知接口)
  - [8.1 接口信息](#接口信息-4)
  - [8.2 请求参数说明](#请求参数说明-4)
  - [8.3 请求示例](#请求示例-4)
  - [8.4 商户处理流程](#商户处理流程)
- [九、退款结果通知接口](#退款结果通知接口)
  - [9.1 接口信息](#接口信息-5)
  - [9.2 请求参数说明](#请求参数说明-5)
  - [9.3 请求示例](#请求示例-5)
  - [9.4 商户处理流程](#商户处理流程-1)
- [十、订单状态码说明](#订单状态码解释)

## 接口设计概述

本章节说明所有接口的统一参数结构和安全设计原则，是理解后续各业务接口的基础。

### 统一参数结构

所有接口均采用 **公共参数 + bizContent** 的统一结构：

```json
{
  "merchantId": "商户ID（公共参数，明文）",
  "requestTime": "请求时间（公共参数，明文）",
  "encrypt_type": "AES（可选，使用加密时必填）",
  "bizContent": "业务数据（JSON字符串或AES加密后的Base64字符串）",
  "sign": "RSA2签名（不参与签名计算）"
}
```

### 核心安全原则

#### 1. 加密策略：只加密业务数据（bizContent）

- **加密对象**：只加密 `bizContent` 敏感业务数据
- **不加密对象**：公共参数（merchantId、requestTime、encrypt_type）
- **优势**：性能更好、便于系统路由和日志记录、调试友好

#### 2. 签名策略：对所有参数签名（除sign字段外）

- **签名对象**：所有参数，包括 `merchantId`、`requestTime`、`bizContent`、`encrypt_type`（如有）
- **不签名对象**：仅 `sign` 字段本身
- **目的**：防止任何参数被篡改，保证数据完整性，防止重放攻击

#### 3. 执行顺序：先加密业务数据，再对所有参数签名

```
步骤1：准备 bizContent 业务数据（JSON格式）
  ↓
步骤2：【可选】AES加密 bizContent（得到Base64密文）
  ↓
步骤3：组装所有参数（公共参数 + bizContent + encrypt_type）
  ↓
步骤4：对所有参数（除sign外）生成签名字符串
  ↓
步骤5：RSA2签名 + Base64编码，得到 sign
  ↓
步骤6：发送请求
```

#### 4. 验证顺序：先验签，再解密

```
步骤1：接收响应/通知
  ↓
步骤2：先验签（使用对方RSA公钥）
  ↓
步骤3：验签通过 → 继续 | 验签失败  → 丢弃数据
  ↓
步骤4：检查是否有 encrypt_type=AES
  ↓
步骤5：【如有加密】AES解密 bizContent
  ↓
步骤6：处理业务数据
```

## 签名规则

本章节介绍 RSA2 + AES 的密钥准备、签名/验签规则及推荐的调用顺序，建议在开发前完整阅读。

### 0. 密钥交换与初始化

在开始使用接口前，需要进行密钥交换与初始化：

1. **RSA密钥交换**：
   - 商户生成RSA密钥对（商户私钥 + 商户公钥）
   - 系统提供RSA密钥对（系统私钥 + 系统公钥）
   - **商户与系统交换RSA公钥**：商户保存系统公钥，系统保存商户公钥
   - **各自保存私钥**：商户保存自己的RSA私钥，系统保存自己的RSA私钥
   - 商户私钥用于签名（加签），系统公钥用于验签
   - 系统私钥用于签名（加签），商户公钥用于验签

2. **AES密钥生成**：
   - 系统为每个商户生成唯一的AES密钥
   - 商户与系统共同持有该AES密钥
   - AES密钥用于敏感业务数据的加密与解密

**密钥用途总结**：
- **商户RSA私钥**：商户对请求进行签名
- **系统RSA公钥**：商户验证系统响应的签名
- **系统RSA私钥**：系统对响应进行签名
- **商户RSA公钥**：系统验证商户请求的签名
- **AES密钥**：商户和系统共同持有，用于加密/解密业务数据

### 1. 签名加密方式

采用支付宝的RSA2+AES加签/验签加密方式，与支付宝对接流程完全一致：

- **RSA2签名**：使用RSA私钥对请求参数进行签名，签名算法为SHA256withRSA
- **AES加密**：对敏感业务数据进行AES加密传输

**核心原则**：**先加密业务数据，再对所有参数进行签名**

### 2. 商户发送请求流程（发送方）

#### 2.1 准备业务数据（bizContent）

首先将敏感业务数据组装成JSON格式的 `bizContent`。例如：

```json
{
  "buyerPhone": "18611111111",
  "goodsList": [
    {
      "goodsSkuId": 18212582980874649,
      "goodsNum": 2
    },
    {
      "goodsSkuId": 18212582980874650,
      "goodsNum": 1
    }
  ],
  "outOrderId": "OUT_ORDER_ID",
  "payType": "MINI_APP"
}
```

#### 2.2 AES加密业务数据（可选但推荐）

使用**AES密钥**对 `bizContent` 进行加密：

- **输入**：bizContent 的JSON字符串
- **输出**：加密后的字符串（Base64编码）
- **操作**：将原始 bizContent 替换为加密后的密文

**重要**：AES只加密 `bizContent` 业务数据内容，不加密公共参数（如merchantId、requestTime等）。

#### 2.3 组装所有请求参数

将加密（或未加密）的 bizContent 与公共参数放在一起：

**公共参数：**
- `merchantId`: 商户ID
- `requestTime`: 请求时间（格式：YYYY-MM-DD HH:MM:SS）
- `encrypt_type`: AES
- `bizContent`: 业务数据（加密后的密文或明文JSON字符串）

#### 2.4 生成待签名字符串

1. **过滤参数**：排除 `sign` 字段本身（签名不能自己签自己）
2. **排序参数**：将所有参与签名的参数按照参数名的**ASCII码从小到大排序**
3. **拼接字符串**：格式为 `key1=value1&key2=value2`

> **重要**：
> - 所有参数（除sign外）均参与签名，包括 `merchantId`、`requestTime`、`bizContent`、`encrypt_type`（如有）
> - 如果开启了加密，签名时使用的 `bizContent` 必须是**加密后的密文**

**示例待签名字符串（使用加密）：**
```
bizContent=AES_ENCRYPTED_BASE64_STRING&encrypt_type=AES&merchantId=YOUR_MERCHANT_ID&requestTime=2025-07-05 10:10:10
```

#### 2.5 RSA2签名（加签）

使用**商户应用私钥**对上一步生成的"待签名字符串"进行SHA256WithRSA签名，得到签名值。

#### 2.6 Base64编码

对签名结果进行Base64编码，得到最终的签名字符串 `sign`。

#### 2.7 发送请求

将生成的 `sign` 加入到请求参数中，对所有参数值进行**URL Encode**（如需要），通过HTTP POST发送给系统。

### 3. 商户接收响应/通知流程（接收方）

#### 3.1 接收数据

商户收到系统返回的JSON数据（同步响应）或POST表单数据（异步通知）。

#### 3.2 RSA2验签

**在处理任何业务逻辑、甚至解密之前，必须先验签**：

1. 提取系统返回的 `sign`
2. 提取需要验签的内容（同步返回中通常是响应数据；异步通知中是除 `sign` 外的所有参数）
3. 使用**平台公钥**验证签名是否通过
   - **失败**：说明数据可能被篡改或来源伪造，**立即丢弃，不进行解密**
   - **成功**：说明数据来源合法，继续下一步

#### 3.3 检查是否加密

检查返回参数中是否包含 `encrypt_type = AES`，或者根据系统配置判断是否开启了响应加密。

#### 3.4 AES解密

如果数据是加密的，使用**AES密钥**对密文内容进行解密：

- **输入**：AES密文
- **输出**：明文JSON字符串（真实的业务数据）

#### 3.5 处理业务

拿到明文数据后，进行订单状态更新等业务逻辑。

### 4. 签名加密流程图

#### 4.1 商户发送请求流程（完整流程图）

```
开始: 商户准备发送请求
  ↓
1. 准备业务参数（组装JSON格式的业务数据）
  ↓
判断：是否包含敏感数据？
  ├─ 是 → 2. AES加密业务数据（使用AES密钥加密，得到Base64编码的密文）
  │        ↓
  │       3. 组装系统参数（添加merchantId、requestTime、encrypt_type=AES、加密后的业务数据）
  │        ↓
  └─ 否 → 3. 组装系统参数（添加merchantId、requestTime等）
           ↓
4. 生成待签名字符串（排除sign字段，按ASCII码排序参数，拼接为key1=value1&key2=value2）
  ↓
5. RSA2签名（使用商户RSA私钥，SHA256WithRSA算法签名）
  ↓
6. Base64编码（对签名结果进行Base64编码，得到最终的sign值）
  ↓
7. 添加sign到请求参数（进行URL Encode，通过HTTP发送请求）
  ↓
结束: 请求发送完成
```

#### 4.2 商户接收响应/通知流程（完整流程图）

```
开始: 商户接收响应/通知
  ↓
1. 接收数据（同步响应JSON或异步通知POST）
  ↓
2. RSA2验签（提取sign字段，提取待验签参数，使用系统RSA公钥验证）
  ↓
判断：验签是否通过？
  ├─ 失败 → 丢弃数据（不进行任何处理，可能存在安全风险）
  │          ↓
  │         结束: 验证失败
  │
  └─ 成功 → 判断：是否包含encrypt_type=AES？
              ├─ 否 → 5. 处理业务数据（直接使用明文数据，更新订单状态等）
              │        ↓
              │       结束: 业务处理完成
              │
              └─ 是 → 3. AES解密（使用AES密钥，对密文进行解密）
                       ↓
                      4. 得到明文数据（JSON格式的业务数据）
                       ↓
                      5. 处理业务数据（直接使用明文数据，更新订单状态等）
                       ↓
                      结束: 业务处理完成
```

#### 4.3 密钥管理与数据流转图

**密钥初始化**
```
商户生成RSA密钥对 → 商户私钥、商户公钥
系统生成RSA密钥对 → 系统私钥、系统公钥
系统生成AES密钥 → AES密钥
```

**密钥交换**
```
商户私钥、商户公钥 → 商户保存: 商户私钥、系统公钥、AES密钥
系统私钥、系统公钥 → 系统保存: 系统私钥、商户公钥、AES密钥
AES密钥 → 商户保存: AES密钥
AES密钥 → 系统保存: AES密钥
```

**请求流程**
```
商户保存的密钥 → 商户使用商户私钥签名 → 发送请求
商户保存的密钥 → 商户使用AES密钥加密 → 发送请求
```

**响应流程**
```
发送请求 → 系统使用商户公钥验签 → 返回响应
发送请求 → 系统使用AES密钥解密 → 返回响应
系统保存的密钥 → 系统使用系统私钥签名 → 返回响应
系统保存的密钥 → 系统使用AES密钥加密 → 返回响应
```

**商户接收**
```
返回响应 → 商户使用系统公钥验签 → 处理业务数据
返回响应 → 商户使用AES密钥解密 → 处理业务数据
```

### 5. 关键点与注意事项

#### 5.1 加密与签名的核心原则

**核心原则**：**先加密业务数据（bizContent），再对所有参数进行签名**

1. **加密对象**：
   - AES只加密 `bizContent` 业务数据内容
   - 不加密公共参数（如merchantId、requestTime、encrypt_type等）

2. **签名对象**：
   - 签名针对的是**所有参数（除sign字段外）**
   - 包括：`merchantId`、`requestTime`、`bizContent`、`encrypt_type`（如有）
   - 如果开启了加密，签名时用的 `bizContent` 必须是**加密后的密文**
   - 目的：防止任何参数被篡改，保证数据完整性

3. **验签顺序**：
   - 接收数据时，一定要**先验签，再解密**
   - 如果不验签直接解密，可能会导致系统处理了恶意构造的数据，存在安全风险

#### 5.2 具体实现要点

1. **encrypt_type参数**：当使用AES加密时，必须在请求参数中显式添加 `encrypt_type=AES`，告知系统该请求使用了加密

2. **参数排序**：生成待签名字符串时，必须按照参数名的ASCII码从小到大排序

3. **签名算法**：使用SHA256WithRSA算法，签名结果需要进行Base64编码

4. **sign字段**：`sign` 字段本身不参与签名计算（签名不能自己签自己）

5. **bizContent格式**：
   - 不加密时：JSON字符串格式
   - 加密时：AES加密后的Base64编码字符串

### 6. 完整签名加密流程示例

以创建订单接口为例，演示完整的签名加密流程：

#### 6.1 不使用加密的流程

**步骤1：准备 bizContent 业务数据**
```json
{
  "buyerPhone": "18611111111",
  "goodsList": [
    {
      "goodsSkuId": 18212582980874649,
      "goodsNum": 2
    }
  ],
  "outOrderId": "ORDER_20250705_001",
  "payType": "MINI_APP",
  "alipayId": "13432154136"
}
```

**步骤2：组装请求参数**
```json
{
  "merchantId": "MERCHANT_001",
  "requestTime": "2025-07-05 10:10:10",
  "bizContent": "{\"buyerPhone\":\"18611111111\",\"goodsList\":[{\"goodsSkuId\":18212582980874649,\"goodsNum\":2}],\"outOrderId\":\"ORDER_20250705_001\",\"payType\":\"MINI_APP\",\"alipayId\":\"13432154136\"}"
}
```

**步骤3：生成待签名字符串（按ASCII排序）**
```
bizContent={"buyerPhone":"18611111111","goodsList":[{"goodsSkuId":18212582980874649,"goodsNum":2}],"outOrderId":"ORDER_20250705_001","payType":"MINI_APP","alipayId":"13432154136"}&merchantId=MERCHANT_001&requestTime=2025-07-05 10:10:10
```

**步骤4：RSA2签名**
使用商户RSA私钥对待签名字符串进行SHA256WithRSA签名，并Base64编码，得到：
```
sign=iJx8Z9K5m...（Base64编码的签名字符串）
```

**步骤5：最终请求**
```json
{
  "merchantId": "MERCHANT_001",
  "requestTime": "2025-07-05 10:10:10",
  "bizContent": "{\"buyerPhone\":\"18611111111\",\"goodsList\":[{\"goodsSkuId\":18212582980874649,\"goodsNum\":2}],\"outOrderId\":\"ORDER_20250705_001\",\"payType\":\"MINI_APP\",\"alipayId\":\"13432154136\"}",
  "sign": "iJx8Z9K5m..."
}
```

#### 6.2 使用AES加密的流程（推荐）

**步骤1：准备 bizContent 业务数据（同上）**
```json
{
  "buyerPhone": "18611111111",
  "goodsList": [
    {
      "goodsSkuId": 18212582980874649,
      "goodsNum": 2
    }
  ],
  "outOrderId": "ORDER_20250705_001",
  "payType": "MINI_APP",
  "alipayId": "13432154136"
}
```

**步骤2：AES加密 bizContent**
将上述JSON字符串使用AES密钥加密，得到Base64编码的密文：
```
AES_ENCRYPTED_BIZCONTENT_BASE64_STRING
```

**步骤3：组装请求参数（包含encrypt_type）**
```json
{
  "merchantId": "MERCHANT_001",
  "requestTime": "2025-07-05 10:10:10",
  "encrypt_type": "AES",
  "bizContent": "AES_ENCRYPTED_BIZCONTENT_BASE64_STRING"
}
```

**步骤4：生成待签名字符串（按ASCII排序，使用加密后的bizContent）**
```
bizContent=AES_ENCRYPTED_BIZCONTENT_BASE64_STRING&encrypt_type=AES&merchantId=MERCHANT_001&requestTime=2025-07-05 10:10:10
```

**步骤5：RSA2签名**
使用商户RSA私钥对待签名字符串进行SHA256WithRSA签名，并Base64编码，得到：
```
sign=pQr7T3N8w...（Base64编码的签名字符串）
```

**步骤6：最终请求**
```json
{
  "merchantId": "MERCHANT_001",
  "requestTime": "2025-07-05 10:10:10",
  "encrypt_type": "AES",
  "bizContent": "AES_ENCRYPTED_BIZCONTENT_BASE64_STRING",
  "sign": "pQr7T3N8w..."
}
```

#### 6.3 关键要点总结

1. **加密对象**：只加密 `bizContent` 业务数据
2. **签名对象**：所有参数（除sign外），包括 `merchantId`、`requestTime`、`bizContent`、`encrypt_type`
3. **签名内容**：如果使用加密，签名时的 `bizContent` 必须是**加密后的密文**
4. **执行顺序**：先加密 bizContent → 组装所有参数 → 对所有参数签名

## 接口地址

本章节给出测试环境和生产环境网关地址以及对应密钥信息，供接入方正确配置调用目标环境。

### 测试环境

| 配置项 | 值 |
|--------|-----|
| 域名 | https://gatewayuat.jrdaimao.com |
| RSA2私钥 | （商户私钥，用于签名） |
| RSA2公钥 | （平台公钥，用于验签） |
| AES密钥 | （AES加密密钥） |

### 生产环境

| 配置项 | 值 |
|--------|-----|
| 域名 | https://gateway.jrdaimao.com |
| RSA2私钥 | （商户私钥，用于签名） |
| RSA2公钥 | （平台公钥，用于验签） |
| AES密钥 | （AES加密密钥） |

## 创建订单接口

本接口用于创建支付订单，是所有支付流程的入口接口。

### 接口信息

| 项目 | 内容 |
|------|------|
| 接口URL | /easyhome-app-application/smartApi/external/createOrder |
| 请求方法 | POST |
| Content-Type | application/json |

### 请求参数说明

#### 公共参数（明文传输，不加密）

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| merchantId | String | 是 | 商户id | 是 | "YOUR_MERCHANT_ID" |
| requestTime | String | 是 | 请求时间，格式：YYYY-MM-DD HH:MM:SS | 是 | "2025-07-05 10:10:10" |
| encrypt_type | String | 否 | 加密类型，使用AES加密时必填 | 是 | "AES" |
| sign | String | 是 | RSA2签名，Base64编码字符串 | 否 | "Base64编码的签名字符串" |

#### 业务数据参数（敏感信息，需AES加密）

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| bizContent | String | 是 | 业务数据，AES加密后的Base64字符串 | 是 | "加密后的业务数据" |

**bizContent 原始数据结构（加密前）：**

| 参数名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| buyerPhone | String | 是 | 下单人手机号 | "18611111111" |
| goodsList | Array | 是 | 商品列表，支持多个商品组合，每个元素包含商品sku及数量 | `[{"goodsSkuId":18212582980874649,"goodsNum":2},{"goodsSkuId":18212582980874650,"goodsNum":1}]` |
| outOrderId | String | 是 | 商户侧订单号 | "OUT_ORDER_ID" |
| payType | String | 是 | 支付方式 | "MINI_APP" |
| alipayId | String | 否 | 用户支付宝id（支付宝小程序支付时必填） | "13432154136" |
| receiptAddress | String | 否 | 省市 | "北京市" |
| detailReceiptAddress | String | 否 | 详细地址 | "朝阳区xxx街道" |
| payNotifyUrl | String | 否 | 支付通知地址 | "https://your-domain.com/pay/notify" |
| refundNotifyUrl | String | 否 | 退款通知地址 | "https://your-domain.com/refund/notify" |

**goodsList 元素结构：**

| 字段名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| goodsSkuId | Integer | 是 | 商品skuId | 18212582980874649 |
| goodsNum | Integer | 是 | 商品数量 | 2 |

### 支付方式说明

| payType值 | 说明 |
|-----------|------|
| QR_CODE | 支付宝当当面付扫码支付 |
| MINI_APP | 支付宝小程序支付 |

### 请求示例

#### 示例1：不使用加密（明文传输）

**bizContent 原始数据（JSON格式）：**
```json
{
  "buyerPhone": "18611111111",
  "goodsList": [
    {
      "goodsSkuId": 18212582980874649,
      "goodsNum": 2
    },
    {
      "goodsSkuId": 18212582980874650,
      "goodsNum": 1
    }
  ],
  "outOrderId": "OUT_ORDER_ID",
  "payType": "MINI_APP",
  "alipayId": "13432154136",
  "receiptAddress": "北京市",
  "detailReceiptAddress": "朝阳区xxx街道",
  "payNotifyUrl": "https://your-domain.com/pay/notify",
  "refundNotifyUrl": "https://your-domain.com/refund/notify"
}
```

**最终请求体：**
```json
{
  "merchantId": "YOUR_MERCHANT_ID",
  "requestTime": "2025-07-05 10:10:10",
  "bizContent": "{\"buyerPhone\":\"18611111111\",\"goodsList\":[{\"goodsSkuId\":18212582980874649,\"goodsNum\":2},{\"goodsSkuId\":18212582980874650,\"goodsNum\":1}],\"outOrderId\":\"OUT_ORDER_ID\",\"payType\":\"MINI_APP\",\"alipayId\":\"13432154136\",\"receiptAddress\":\"北京市\",\"detailReceiptAddress\":\"朝阳区xxx街道\",\"payNotifyUrl\":\"https://your-domain.com/pay/notify\",\"refundNotifyUrl\":\"https://your-domain.com/refund/notify\"}",
  "sign": "Base64编码的签名字符串"
}
```

**签名字符串（按ASCII排序）：**
```
bizContent={"buyerPhone":"18611111111","goodsList":[{"goodsSkuId":18212582980874649,"goodsNum":2},{"goodsSkuId":18212582980874650,"goodsNum":1}],"outOrderId":"OUT_ORDER_ID","payType":"MINI_APP","alipayId":"13432154136","receiptAddress":"北京市","detailReceiptAddress":"朝阳区xxx街道","payNotifyUrl":"https://your-domain.com/pay/notify","refundNotifyUrl":"https://your-domain.com/refund/notify"}&merchantId=YOUR_MERCHANT_ID&requestTime=2025-07-05 10:10:10
```

#### 示例2：使用AES加密（推荐）

**步骤1：准备 bizContent 原始数据（JSON格式）：**
```json
{
  "buyerPhone": "18611111111",
  "goodsList": [
    {
      "goodsSkuId": 18212582980874649,
      "goodsNum": 2
    },
    {
      "goodsSkuId": 18212582980874650,
      "goodsNum": 1
    }
  ],
  "outOrderId": "OUT_ORDER_ID",
  "payType": "MINI_APP",
  "alipayId": "13432154136",
  "receiptAddress": "北京市",
  "detailReceiptAddress": "朝阳区xxx街道",
  "payNotifyUrl": "https://your-domain.com/pay/notify",
  "refundNotifyUrl": "https://your-domain.com/refund/notify"
}
```

**步骤2：AES加密 bizContent，得到密文：**
```
假设加密后得到：AES_ENCRYPTED_BASE64_STRING
```

**步骤3：组装最终请求体：**
```json
{
  "merchantId": "YOUR_MERCHANT_ID",
  "requestTime": "2025-07-05 10:10:10",
  "encrypt_type": "AES",
  "bizContent": "AES_ENCRYPTED_BASE64_STRING",
  "sign": "Base64编码的签名字符串"
}
```

**步骤4：生成签名字符串（按ASCII排序，使用加密后的bizContent）：**
```
bizContent=AES_ENCRYPTED_BASE64_STRING&encrypt_type=AES&merchantId=YOUR_MERCHANT_ID&requestTime=2025-07-05 10:10:10
```

**步骤5：使用商户RSA私钥对签名字符串进行SHA256WithRSA签名，Base64编码后得到sign值**

### 返回参数说明

系统返回的响应可能是明文或加密格式，商户需要先验签再解密。

#### 公共返回参数

| 参数名 | 类型 | 说明 | 是否参与验签 | 示例 |
|--------|------|------|------------|------|
| code | Integer | 状态码 | 是 | 200 |
| message | String | 状态描述 | 是 | "请求成功" |
| encrypt_type | String | 加密类型（如使用加密） | 是 | "AES" |
| data | String | 业务数据（可能是加密密文或明文） | 是 | 见下方说明 |
| sign | String | RSA2签名（系统使用系统私钥签名） | 否 | "Base64编码的签名字符串" |

#### data 数据说明

`data` 字段可能包含以下内容（根据是否加密）：

1. **不使用加密**：`data` 是 JSON 字符串格式的业务数据
2. **使用AES加密**：`data` 是 AES 加密后的 Base64 字符串

**data 解密后的数据结构：**

| 参数名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| merOrderId | String | 商户订单编号（系统生成的订单ID，用于后续订单查询、退款等操作） | "991751786491891847" |
| payData | Object/String | 支付相关数据（根据不同支付方式返回不同内容） | 见下方说明 |

### 返回示例

#### 示例1：不使用加密的返回

```json
{
  "code": 200,
  "message": "请求成功",
  "data": "{\"merOrderId\":\"991751786491891847\",\"payData\":\"OUT_ORDER_ID\"}",
  "sign": "Base64编码的签名字符串"
}
```

**验签字符串（按ASCII排序）：**
```
code=200&data={"merOrderId":"991751786491891847","payData":"OUT_ORDER_ID"}&message=请求成功
```

#### 示例2：使用AES加密的返回

```json
{
  "code": 200,
  "message": "请求成功",
  "encrypt_type": "AES",
  "data": "AES_ENCRYPTED_BASE64_STRING",
  "sign": "Base64编码的签名字符串"
}
```

**验签字符串（按ASCII排序，使用加密后的data）：**
```
code=200&data=AES_ENCRYPTED_BASE64_STRING&encrypt_type=AES&message=请求成功
```

**data 解密后的内容：**
```json
{
  "merOrderId": "991751786491891847",
  "payData": "OUT_ORDER_ID"
}
```

### payData 字段说明

根据不同的 `payType` 支付方式，`payData` 字段的返回类型和内容会有所不同：

##### 1. MINI_APP（小程序支付）

| 类型 | String |
|------|--------|
| 说明 | 返回 `OUT_ORDER_ID`（订单号），用于支付宝小程序 URL Scheme 静默授权拉起支付 |
| 格式 | 字符串格式的订单号 |
| 示例 | `"OUT_ORDER_ID"` 或 `"991751786491891847"` |

**返回示例：**
```json
{
  "code": 200,
  "message": "请求成功",
  "merOrderId": "991751786491891847",
  "data": "OUT_ORDER_ID"
}
```

** 支付宝小程序支付特殊要求：**

支付宝小程序支付**强制要求**提供支付宝用户的 `openid`（买家支付宝用户标识）。支付宝小程序客户端需要提供特殊的 **URL Scheme** 拉起功能，可以通过 `OUT_ORDER_ID` 订单号在**用户静默授权**的情况下获取用户 `openid` 并拉起支付宝小程序支付。

**关键特性：**
-  **静默授权**：无需用户手动确认授权登录
-  **OUT_ORDER_ID 专属**：只有通过此接口创建的订单才具备 `OUT_ORDER_ID`，才能通过 URL Scheme 拉起支付
-  **小程序内支付**：支付成功后在支付宝小程序内跳转支付成功页
-  **其他方式不可用**：其他方式创建的订单不具备 `OUT_ORDER_ID`，则不进行静默授权流程即可阻止拉起支付宝小程序支付
---

**支付宝小程序静默授权流程：**

```
1. 用户打开小程序（App.onLaunch）
   ↓
   小程序启动时，立即在后台静默调用：
   my.getAuthCode({ scopes: 'auth_base' })
   
2. 换取 OpenID
   ↓
   拿到 auth_code 后，发送给后端
   后端调用 alipay.system.oauth.token 接口换取 OpenID
   
3. 静默注册/登录
   ↓
   后端保存用户的 OpenID，完成用户身份识别
   
4. 发起支付
   ↓
   后端已经有了这个用户的 OpenID
   直接填入 alipay.trade.create 接口的 buyer_open_id 字段
   返回 OUT_ORDER_ID 给客户端
   
5. 拉起支付
   ↓
   客户端使用 URL Scheme + OUT_ORDER_ID 拉起支付宝小程序支付
   用户完成支付后，在小程序内跳转支付成功页
```

---

**需要 URL Scheme 使用场景：**

| 场景 | URL Scheme 功能 | 说明 |
|------|----------------|------|
| **场景1：普通支付** | 拉起支付宝小程序支付 | 使用 `OUT_ORDER_ID` 通过 URL Scheme 拉起支付，支付成功后在小程序内跳转支付成功页 |
| **场景2：扫码支付** | 拉起支付宝小程序扫一扫 | 需要支付宝小程序客户端提供特殊 URL Scheme 便捷拉起支付宝小程序内的扫一扫功能，应对支付宝当面付二维码的情况，支付成功同样在支付宝小程序页面跳转支付成功页 |

**使用说明：**
1. **普通支付流程**：商户获取到 `OUT_ORDER_ID` 后，通过支付宝小程序 URL Scheme 拉起支付
2. **扫码支付流程**：通过 URL Scheme 拉起支付宝小程序内的扫一扫功能，扫描支付宝当面付二维码完成支付
3. **支付结果**：支付成功后，都会在支付宝小程序内跳转到支付成功页面
4. **订单查询**：可以使用 `merOrderId` 或 `outOrderId` 通过支付查询接口查询订单状态

##### 2. QR_CODE（当面付-扫码支付）

| 类型 | String |
|------|--------|
| 说明 | 返回支付二维码URL或支付参数字符串，商户需要使用该URL生成二维码供用户扫码支付 |
| 格式 | 完整的支付二维码URL地址，或包含二维码信息的JSON字符串 |
| 示例 | `"https://qr.alipay.com/xxx"` 或 `"{\"qrCode\":\"https://qr.alipay.com/xxx\"}"` |

**返回示例：**
```json
{
  "code": 200,
  "message": "请求成功",
  "merOrderId": "991751786491891847",
  "data": "https://qr.alipay.com/baxxxxxx"
}
```

**使用说明：**
- 商户获取到二维码URL后，需要使用该URL生成二维码图片
- 用户使用支付宝App扫描该二维码完成支付
- 支付结果通过 `payNotifyUrl` 异步通知，也可通过支付查询接口主动查询

##### 通用说明

1. **merOrderId存储**：所有支付方式成功返回时都会包含 `merOrderId` 字段（系统生成的订单编号），商户**必须存储**该字段，用于后续的订单查询、退款等操作
2. **数据类型判断**：商户在接收到响应后，需要根据 `payType` 来判断 `data` 的类型和格式
3. **JSON解析**：如果 `data` 为JSON字符串，需要使用 `JSON.parse()` 等方法解析后再使用
4. **错误处理**：当 `code` 不为 200 时，`data` 可能为 `null` 或包含错误信息，`merOrderId` 也可能为空
5. **订单号说明**：
   - `merOrderId`：系统生成的订单编号，用于调用支付查询、退款等接口
   - `outOrderId`：商户传入的订单号，用于订单关联
   - 对于小程序支付和App支付，`data` 可能直接返回商户订单号或支付参数
6. **异步通知**：无论哪种支付方式，支付结果都会通过 `payNotifyUrl` 异步通知，建议同时处理异步通知和主动查询

## 支付查询接口

本接口用于根据平台订单号或商户订单号查询支付结果，适合在异步通知之外进行结果补偿或对账。

### 接口信息

| 项目 | 内容 |
|------|------|
| 接口URL | /easyhome-app-application/smartApi/external/paymentQuery |
| 请求方法 | POST |
| Content-Type | application/json |

### 请求参数说明

#### 公共参数（明文传输，不加密）

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| merchantId | String | 是 | 商户id | 是 | "YOUR_MERCHANT_ID" |
| requestTime | String | 是 | 请求时间，格式：YYYY-MM-DD HH:MM:SS | 是 | "2025-07-05 10:10:10" |
| encrypt_type | String | 否 | 加密类型，使用AES加密时必填 | 是 | "AES" |
| sign | String | 是 | RSA2签名，Base64编码字符串 | 否 | "Base64编码的签名字符串" |

#### 业务数据参数（敏感信息，需AES加密）

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| bizContent | String | 是 | 业务数据，AES加密后的Base64字符串 | 是 | "加密后的业务数据" |

**bizContent 原始数据结构（加密前）：**

| 参数名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| merOrderId | String | 否 | 商户订单编号（与outOrderId二选一） | "YOUR_MERCHANT_ORDER_ID" |
| outOrderId | String | 否 | 商户订单号（与merOrderId二选一） | "OUT_ORDER_ID" |

**注意**：merOrderId和outOrderId必须填写其中一个，不能同时为空。

### 请求示例

#### 示例1：不使用加密，使用merOrderId查询

**bizContent 原始数据：**
```json
{
  "merOrderId": "YOUR_MERCHANT_ORDER_ID"
}
```

**最终请求体：**
```json
{
  "merchantId": "YOUR_MERCHANT_ID",
  "requestTime": "2025-07-05 10:10:10",
  "bizContent": "{\"merOrderId\":\"YOUR_MERCHANT_ORDER_ID\"}",
  "sign": "Base64编码的签名字符串"
}
```

**签名字符串：**
```
bizContent={"merOrderId":"YOUR_MERCHANT_ORDER_ID"}&merchantId=YOUR_MERCHANT_ID&requestTime=2025-07-05 10:10:10
```

#### 示例2：使用AES加密，使用outOrderId查询

**bizContent 原始数据：**
```json
{
  "outOrderId": "OUT_ORDER_ID"
}
```

**AES加密后，最终请求体：**
```json
{
  "merchantId": "YOUR_MERCHANT_ID",
  "requestTime": "2025-07-05 10:10:10",
  "encrypt_type": "AES",
  "bizContent": "AES_ENCRYPTED_BASE64_STRING",
  "sign": "Base64编码的签名字符串"
}
```

**签名字符串（使用加密后的bizContent）：**
```
bizContent=AES_ENCRYPTED_BASE64_STRING&encrypt_type=AES&merchantId=YOUR_MERCHANT_ID&requestTime=2025-07-05 10:10:10
```

### 返回参数说明

#### 公共返回参数

| 参数名 | 类型 | 说明 | 是否参与验签 | 示例 |
|--------|------|------|------------|------|
| code | Integer | 状态码 | 是 | 200 |
| message | String | 状态描述 | 是 | "查询成功" |
| encrypt_type | String | 加密类型（如使用加密） | 是 | "AES" |
| data | String | 业务数据（可能是加密密文或明文JSON字符串） | 是 | 见下方说明 |
| sign | String | RSA2签名（系统使用系统私钥签名） | 否 | "Base64编码的签名字符串" |

#### data 解密后的数据结构

| 参数名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| message | String | 消息 | "" |
| targetOrderId | String | 目标订单号 | "2023052222001419051436220914" |
| totalAmount | String | 总金额 | "1950" |
| payTime | String | 支付时间 | "2023-05-22T10:18:22.000+08:00" |
| billNo | String | 账单号 | "1684721886666" |
| status | String | 支付状态 | "TRADE_SUCCESS" |

### 返回示例

#### 示例1：不使用加密

```json
{
  "code": 200,
  "message": "查询成功",
  "data": "{\"message\":\"\",\"targetOrderId\":\"2023052222001419051436220914\",\"totalAmount\":\"1950\",\"payTime\":\"2023-05-22T10:18:22.000+08:00\",\"billNo\":\"1684721886666\",\"status\":\"TRADE_SUCCESS\"}",
  "sign": "Base64编码的签名字符串"
}
```

**验签字符串：**
```
code=200&data={"message":"","targetOrderId":"2023052222001419051436220914","totalAmount":"1950","payTime":"2023-05-22T10:18:22.000+08:00","billNo":"1684721886666","status":"TRADE_SUCCESS"}&message=查询成功
```

#### 示例2：使用AES加密

```json
{
  "code": 200,
  "message": "查询成功",
  "encrypt_type": "AES",
  "data": "AES_ENCRYPTED_BASE64_STRING",
  "sign": "Base64编码的签名字符串"
}
```

**验签字符串（使用加密后的data）：**
```
code=200&data=AES_ENCRYPTED_BASE64_STRING&encrypt_type=AES&message=查询成功
```

## 退款接口

本接口用于对已支付成功的订单发起退款申请，仅支持通过订单号或商户订单号进行退款。

### 接口信息

| 项目 | 内容 |
|------|------|
| 接口URL | /easyhome-app-application/smartApi/external/refundApply |
| 请求方法 | POST |
| Content-Type | application/json |

### 请求参数说明

#### 公共参数（明文传输，不加密）

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| merchantId | String | 是 | 商户id | 是 | "YOUR_MERCHANT_ID" |
| requestTime | String | 是 | 请求时间，格式：YYYY-MM-DD HH:MM:SS | 是 | "2025-07-05 10:10:10" |
| encrypt_type | String | 否 | 加密类型，使用AES加密时必填 | 是 | "AES" |
| sign | String | 是 | RSA2签名，Base64编码字符串 | 否 | "Base64编码的签名字符串" |

#### 业务数据参数（敏感信息，需AES加密）

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| bizContent | String | 是 | 业务数据，AES加密后的Base64字符串 | 是 | "加密后的业务数据" |

**bizContent 原始数据结构（加密前）：**

| 参数名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| orderId | String | 否 | 订单号（与outOrderId二选一） | "991751786491891847" |
| outOrderId | String | 否 | 商户订单号（与orderId二选一） | "OUT_ORDER_ID" |

**注意**：orderId和outOrderId必须填写其中一个，不能同时为空。

### 请求示例

#### 示例1：不使用加密，使用orderId退款

**bizContent 原始数据：**
```json
{
  "orderId": "991751786491891847"
}
```

**最终请求体：**
```json
{
  "merchantId": "YOUR_MERCHANT_ID",
  "requestTime": "2025-07-05 10:10:10",
  "bizContent": "{\"orderId\":\"991751786491891847\"}",
  "sign": "Base64编码的签名字符串"
}
```

**签名字符串：**
```
bizContent={"orderId":"991751786491891847"}&merchantId=YOUR_MERCHANT_ID&requestTime=2025-07-05 10:10:10
```

#### 示例2：使用AES加密，使用outOrderId退款

**bizContent 原始数据：**
```json
{
  "outOrderId": "OUT_ORDER_ID"
}
```

**AES加密后，最终请求体：**
```json
{
  "merchantId": "YOUR_MERCHANT_ID",
  "requestTime": "2025-07-05 10:10:10",
  "encrypt_type": "AES",
  "bizContent": "AES_ENCRYPTED_BASE64_STRING",
  "sign": "Base64编码的签名字符串"
}
```

**签名字符串（使用加密后的bizContent）：**
```
bizContent=AES_ENCRYPTED_BASE64_STRING&encrypt_type=AES&merchantId=YOUR_MERCHANT_ID&requestTime=2025-07-05 10:10:10
```

### 返回参数说明

#### 公共返回参数

| 参数名 | 类型 | 说明 | 是否参与验签 | 示例 |
|--------|------|------|------------|------|
| code | Integer | 状态码 | 是 | 200 |
| message | String | 状态描述 | 是 | "退款成功" |
| encrypt_type | String | 加密类型（如使用加密） | 是 | "AES" |
| data | String | 业务数据（可能是加密密文或明文JSON字符串） | 是 | 见下方说明 |
| sign | String | RSA2签名（系统使用系统私钥签名） | 否 | "Base64编码的签名字符串" |

#### data 解密后的数据结构

| 参数名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| refundApplyId | String | 退款申请单号 | "YOUR_REFUND_APPLY_ID" |

### 返回示例

#### 示例1：不使用加密

```json
{
  "code": 200,
  "message": "退款成功",
  "data": "{\"refundApplyId\":\"REFUND_APPLY_20250705_001\"}",
  "sign": "Base64编码的签名字符串"
}
```

#### 示例2：使用AES加密

```json
{
  "code": 200,
  "message": "退款成功",
  "encrypt_type": "AES",
  "data": "AES_ENCRYPTED_BASE64_STRING",
  "sign": "Base64编码的签名字符串"
}
```

## 退款结果查询接口

本接口用于查询退款申请的处理结果，通常与退款申请接口配合使用。

### 接口信息

| 项目 | 内容 |
|------|------|
| 接口URL | /easyhome-app-application/smartApi/external/refundQuery |
| 请求方法 | POST |
| Content-Type | application/json |

### 请求参数说明

#### 公共参数（明文传输，不加密）

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| merchantId | String | 是 | 商户id | 是 | "YOUR_MERCHANT_ID" |
| requestTime | String | 是 | 请求时间，格式：YYYY-MM-DD HH:MM:SS | 是 | "2025-07-05 10:10:10" |
| encrypt_type | String | 否 | 加密类型，使用AES加密时必填 | 是 | "AES" |
| sign | String | 是 | RSA2签名，Base64编码字符串 | 否 | "Base64编码的签名字符串" |

#### 业务数据参数（敏感信息，需AES加密）

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| bizContent | String | 是 | 业务数据，AES加密后的Base64字符串 | 是 | "加密后的业务数据" |

**bizContent 原始数据结构（加密前）：**

| 参数名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| orderRefundApplyId | String | 否 | 退款申请单号（与outOrderId二选一） | "YOUR_REFUND_APPLY_ID" |
| outOrderId | String | 否 | 商户订单号（与orderRefundApplyId二选一） | "OUT_ORDER_ID" |

**注意**：orderRefundApplyId和outOrderId必须填写其中一个，不能同时为空。

### 请求示例

#### 示例1：不使用加密，使用orderRefundApplyId查询

**bizContent 原始数据：**
```json
{
  "orderRefundApplyId": "YOUR_REFUND_APPLY_ID"
}
```

**最终请求体：**
```json
{
  "merchantId": "YOUR_MERCHANT_ID",
  "requestTime": "2025-07-05 10:10:10",
  "bizContent": "{\"orderRefundApplyId\":\"YOUR_REFUND_APPLY_ID\"}",
  "sign": "Base64编码的签名字符串"
}
```

**签名字符串：**
```
bizContent={"orderRefundApplyId":"YOUR_REFUND_APPLY_ID"}&merchantId=YOUR_MERCHANT_ID&requestTime=2025-07-05 10:10:10
```

#### 示例2：使用AES加密，使用outOrderId查询

**bizContent 原始数据：**
```json
{
  "outOrderId": "OUT_ORDER_ID"
}
```

**AES加密后，最终请求体：**
```json
{
  "merchantId": "YOUR_MERCHANT_ID",
  "requestTime": "2025-07-05 10:10:10",
  "encrypt_type": "AES",
  "bizContent": "AES_ENCRYPTED_BASE64_STRING",
  "sign": "Base64编码的签名字符串"
}
```

**签名字符串（使用加密后的bizContent）：**
```
bizContent=AES_ENCRYPTED_BASE64_STRING&encrypt_type=AES&merchantId=YOUR_MERCHANT_ID&requestTime=2025-07-05 10:10:10
```

### 返回参数说明

#### 公共返回参数

| 参数名 | 类型 | 说明 | 是否参与验签 | 示例 |
|--------|------|------|------------|------|
| code | Integer | 状态码 | 是 | 200 |
| message | String | 状态描述 | 是 | "退款成功" |
| encrypt_type | String | 加密类型（如使用加密） | 是 | "AES" |
| data | String | 业务数据（可能是加密密文或明文JSON字符串） | 是 | 见下方说明 |
| sign | String | RSA2签名（系统使用系统私钥签名） | 否 | "Base64编码的签名字符串" |

#### data 解密后的数据结构

| 参数名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| refundPaymentStatus | String | 退款状态 | "REFUND_SUCCESS" |
| orderId | String | 订单号 | "991751786491891847" |

### 返回示例

#### 示例1：不使用加密

```json
{
  "code": 200,
  "message": "退款成功",
  "data": "{\"refundPaymentStatus\":\"REFUND_SUCCESS\",\"orderId\":\"991751786491891847\"}",
  "sign": "Base64编码的签名字符串"
}
```

#### 示例2：使用AES加密

```json
{
  "code": 200,
  "message": "退款成功",
  "encrypt_type": "AES",
  "data": "AES_ENCRYPTED_BASE64_STRING",
  "sign": "Base64编码的签名字符串"
}
```

## 支付结果通知接口

本接口为异步通知接口，系统在支付结果变更后回调商户的 `payNotifyUrl`，用于驱动商户侧订单状态更新。

### 接口信息

| 项目 | 内容 |
|------|------|
| 接口URL | （由商户提供，在创建订单时通过payNotifyUrl参数指定） |
| 请求方法 | POST |
| Content-Type | application/json |

### 请求参数说明

系统会向商户的 `payNotifyUrl` 发送支付结果通知。

#### 公共参数（明文传输）

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| encrypt_type | String | 否 | 加密类型，使用AES加密时包含此字段 | 是 | "AES" |
| sign | String | 是 | RSA2签名，Base64编码字符串（系统使用系统私钥签名） | 否 | "Base64编码的签名字符串" |

#### 业务数据参数

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| bizContent | String | 是 | 业务数据，可能是AES加密后的Base64字符串或明文JSON字符串 | 是 | "加密后的业务数据或明文" |

**bizContent 原始数据结构（解密前）：**

| 参数名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| merOrderId | String | 是 | 商户订单号（系统生成） | "YOUR_MERCHANT_ORDER_ID" |
| outOrderId | String | 是 | 商户订单号（商户传入） | "OUT_ORDER_ID" |
| paymentStatus | String | 是 | 支付状态 | "TRADE_SUCCESS" |

### 请求示例

#### 示例1：不使用加密

```json
{
  "bizContent": "{\"merOrderId\":\"991751786491891847\",\"outOrderId\":\"ORDER_20250705_001\",\"paymentStatus\":\"TRADE_SUCCESS\"}",
  "sign": "Base64编码的签名字符串"
}
```

#### 示例2：使用AES加密

```json
{
  "encrypt_type": "AES",
  "bizContent": "AES_ENCRYPTED_BASE64_STRING",
  "sign": "Base64编码的签名字符串"
}
```

### 商户处理流程

1. **验签**：使用系统RSA公钥验证 `sign` 签名
2. **解密**（如有encrypt_type=AES）：使用AES密钥解密 `bizContent`
3. **处理业务**：更新订单状态等
4. **返回响应**：返回成功或失败标识

**商户必须返回**：
```json
{
  "code": 200,
  "message": "success"
}
```
或其他约定的成功标识，否则系统会重试通知。

## 退款结果通知接口

本接口为异步通知接口，系统在退款结果产生后回调商户的 `refundNotifyUrl`，用于驱动商户侧退款单状态更新。

### 接口信息

| 项目 | 内容 |
|------|------|
| 接口URL | （由商户提供，在创建订单时通过refundNotifyUrl参数指定） |
| 请求方法 | POST |
| Content-Type | application/json |

### 请求参数说明

系统会向商户的 `refundNotifyUrl` 发送退款结果通知。

#### 公共参数（明文传输）

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| encrypt_type | String | 否 | 加密类型，使用AES加密时包含此字段 | 是 | "AES" |
| sign | String | 是 | RSA2签名，Base64编码字符串（系统使用系统私钥签名） | 否 | "Base64编码的签名字符串" |

#### 业务数据参数

| 参数名 | 类型 | 必填 | 说明 | 是否参与签名 | 示例 |
|--------|------|------|------|------------|------|
| bizContent | String | 是 | 业务数据，可能是AES加密后的Base64字符串或明文JSON字符串 | 是 | "加密后的业务数据或明文" |

**bizContent 原始数据结构（解密前）：**

| 参数名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| refundPaymentId | String | 是 | 退款单号 | "YOUR_REFUND_PAYMENT_ID" |
| outOrderId | String | 是 | 商户订单号 | "OUT_ORDER_ID" |
| refundStatus | String | 是 | 退款状态 | "REFUND_SUCCESS" |

### 请求示例

#### 示例1：不使用加密

```json
{
  "bizContent": "{\"refundPaymentId\":\"REFUND_20250705_001\",\"outOrderId\":\"ORDER_20250705_001\",\"refundStatus\":\"REFUND_SUCCESS\"}",
  "sign": "Base64编码的签名字符串"
}
```

#### 示例2：使用AES加密

```json
{
  "encrypt_type": "AES",
  "bizContent": "AES_ENCRYPTED_BASE64_STRING",
  "sign": "Base64编码的签名字符串"
}
```

### 商户处理流程

1. **验签**：使用系统RSA公钥验证 `sign` 签名
2. **解密**（如有encrypt_type=AES）：使用AES密钥解密 `bizContent`
3. **处理业务**：更新退款状态等
4. **返回响应**：返回成功或失败标识

**商户必须返回**：
```json
{
  "code": 200,
  "message": "success"
}
```
或其他约定的成功标识，否则系统会重试通知。

## 订单状态码解释

本章节对支付和退款相关的核心状态码进行统一说明，方便接入方对照处理业务逻辑。

| 状态码 | 说明 |
|--------|------|
| TRADE_CLOSED | 交易关闭 |
| TRADE_FINISHED | 交易完结 |
| TRADE_SUCCESS | 支付成功 |
| WAIT_BUYER_PAY | 交易创建 |
| REFUND_SUCCESS | 退款成功 |
| REFUND_FAILED | 退款失败 |

---

