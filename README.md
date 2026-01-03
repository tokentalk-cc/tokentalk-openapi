# TokenTalk 第三方充提 API 文档

TokenTalk 平台为第三方开发者提供安全、高效的充提业务 API，支持平台内部资金流转。

## 📋 目录

- [快速开始](#快速开始)
- [用户授权机制](#用户授权机制)
- [API 概览](#api-概览)
- [鉴权说明](#鉴权说明)
- [接口文档](#接口文档)
- [错误码说明](#错误码说明)
- [SDK 示例](#sdk-示例)
- [常见问题](#常见问题)

## 🚀 快速开始

### 1. 申请接入

1. 在 TokenTalk 平台注册账户
2. 提交第三方应用申请（提供公司信息、联系方式等）
3. 等待平台审核
4. 审核通过后获得 API 凭证（`app_key` 和 `app_secret`）

### 2. 配置环境

- **Base URL**: `https://api.tokentalk.cc`
- **协议**: HTTPS
- **数据格式**: JSON
- **字符编码**: UTF-8

### 3. 业务流程概述

```
┌────────────────────────────────────────────────────────────────────────────┐
│                          第三方充提业务流程                                   │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐              │
│  │  1.用户      │  →   │  2.跳转平台    │  →   │  3.用户授权   │              │
│  │  发起授权     │      │  授权页面      │      │  确认        │              │
│  └──────────────┘      └──────────────┘      └──────────────┘              │
│                                                                            │
│                                                      ↓                     │
│                                                                            │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐              │
│  │  4.返回Token │  →   │  5.第三方      │  →   │  6.验证Token  │             │
│  │  给第三方     │      │  获取Token    │      │  获取user_id  │             │
│  └──────────────┘      └──────────────┘      └──────────────┘              │
│                                                                            │
│                                                      ↓                     │
│                                                                            │
│                              ┌──────────────────────────┐                  │
│                              │  7.调用充值/提现接口        │                  │
│                              │  完成资金操作              │                  │
│                              └──────────────────────────┘                  │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

## 🔐 用户授权机制

### 为什么需要授权？

为了保护用户隐私和资金安全，第三方应用**不能直接获取用户ID**。用户必须在 TokenTalk 平台上主动授权，生成专属的 `authorization_token`，第三方通过此 Token 获取用户ID并进行充提操作。

### 完整授权流程图

```
┌──────────────┐          ┌──────────────┐          ┌──────────────┐
│    用户       │          │  TokenTalk   │          │  第三方应用   │
│   (浏览器)    │          │    平台       │          │    服务器     │
└──────┬───────┘          └──────┬───────┘          └──────┬───────┘
       │                         │                         │
       │  1. 用户在第三方网站点击"授权充值"                     │
       │<──────────────────────────────────────────────────│
       │                         │                         │
       │  2. 第三方引导用户跳转到平台授权页面                   │
       │     URL: https://tokentalk.cc/authorize           │
       │     ?app_id=xxx&redirect_uri=xxx&state=xxx        │
       │────────────────────────>│                         │
       │                         │                         │
       │  3. 用户登录平台（如未登录）                          │
       │<───────────────────────>│                         │
       │                         │                         │
       │  4. 平台展示授权确认页面                             │
       │     "XXX应用请求访问您的账户"                        │
       │     [授权范围: 充值/提现]                            │
       │<────────────────────────│                         │
       │                         │                         │
       │  5. 用户确认授权                                    │
       │────────────────────────>│                         │
       │                         │                         │
       │  6. 平台生成 authorization_token                   │
       │                         │──┐                      │
       │                         │  │ 保存授权记录           │
       │                         │<─┘                      │
       │                         │                         │
       │  7. 平台重定向回第三方，携带 token                    │
       │     redirect_uri?token=auth_xxx&state=xxx         │
       │<────────────────────────│                         │
       │                         │                         │
       │  8. 浏览器跳转到第三方回调地址                        │
       │──────────────────────────────────────────────────>│
       │                         │                         │
       │                         │  9. 第三方验证Token       │
       │                         │     POST /auth/token    │
       │                         │<────────────────────────│
       │                         │                         │
       │                         │  10. 返回 user_id 等信息  │
       │                         │────────────────────────>│
       │                         │                         │
       │                         │  11. 第三方调用充提接口    │
       │                         │     POST /deposit/create│
       │                         │     (with auth_token)   │
       │                         │<────────────────────────│
       │                         │                         │
       │                         │  12. 返回充提结果         │
       │                         │────────────────────────>│
       │                         │                         │
       │  13. 第三方展示操作结果给用户                         │
       │<──────────────────────────────────────────────────│
       │                         │                         │
```

### 授权流程说明

| 步骤 | 说明 |
|------|------|
| **1-2** | 用户在第三方网站发起授权，第三方构建授权链接引导用户跳转到 TokenTalk 平台 |
| **3-5** | 用户在 TokenTalk 平台登录并确认授权范围（充值/提现） |
| **6-8** | 平台生成 Token 并重定向回第三方的回调地址 |
| **9-10** | 第三方调用 API 验证 Token，获取用户的 `user_id` |
| **11-13** | 第三方使用 Token 调用充值/提现接口完成业务操作 |

### 授权链接参数

第三方需要引导用户跳转到以下授权页面：

```
https://tokentalk.cc/authorize?app_id={app_id}&redirect_uri={redirect_uri}&state={state}&scopes={scopes}
```

| 参数 | 必需 | 说明 |
|------|------|------|
| `app_id` | 是 | 第三方应用ID |
| `redirect_uri` | 是 | 授权完成后的回调地址（需要URL编码） |
| `state` | 是 | 随机字符串，用于防止CSRF攻击，回调时原样返回 |
| `scopes` | 否 | 授权范围，逗号分隔：`deposit`,`withdraw`,`all`，默认 `all` |

### 授权回调

用户授权成功后，平台会重定向到 `redirect_uri`，携带以下参数：

```
{redirect_uri}?authorization_token={token}&state={state}
```

| 参数 | 说明 |
|------|------|
| `authorization_token` | 授权Token，用于后续API调用 |
| `state` | 第三方传入的state参数，原样返回 |

### 授权 Token 特性

| 特性 | 说明 |
|------|------|
| **防枚举攻击** | Token 为随机生成的64+字符，无法被猜测 |
| **用户可控** | 用户主动授权，可随时撤销 |
| **可撤销** | 用户可在平台内随时撤销授权，立即生效 |
| **有效期控制** | 支持设置过期时间 |
| **使用次数限制** | 支持设置最大使用次数 |
| **范围控制** | 可限制仅充值或仅提现 |

### 授权范围（Scopes）

| 范围 | 说明 |
|------|------|
| `deposit` | 允许充值（用户向第三方账户转账） |
| `withdraw` | 允许提现（第三方账户向用户转账） |
| `all` | 允许所有操作 |

## 📡 API 概览

### 授权相关（第三方调用）
- `POST /api/third-party/auth/token` - 验证授权Token，获取用户信息
- `GET /api/third-party/auth/token/status` - 查询授权Token状态

### 账户查询
- `GET /api/third-party/account/balance` - 查询第三方账户余额
- `GET /api/third-party/account/ledgers` - 查询第三方账户流水

### 充值业务
- `POST /api/third-party/deposit/create` - 创建充值订单（平台用户 → 第三方账户）
- `GET /api/third-party/deposit/query` - 查询充值订单状态

### 提现业务
- `POST /api/third-party/withdraw/create` - 创建提现订单（第三方账户 → 平台用户）
- `GET /api/third-party/withdraw/query` - 查询提现订单状态

## 🔐 鉴权说明

### 请求头

每个请求必须携带以下请求头：

| 请求头 | 说明 | 示例 |
|--------|------|------|
| `X-App-Key` | API Key | `ak_abcdef1234567890` |
| `X-Signature` | 签名值 | `a1b2c3d4e5f6...` |
| `X-Timestamp` | 时间戳（毫秒） | `1704067200000` |
| `X-Nonce` | 随机字符串 | `random123456` |
| `X-Request-Id` | 请求唯一ID（可选） | `req_1234567890` |

### 签名算法

```go
func GenerateSignature(params map[string]string, appKey, appSecret string) string {
    // 1. 添加 key 和 secret
    params["key"] = appKey
    params["secret"] = appSecret
    
    // 2. 按 key 排序
    keys := make([]string, 0, len(params))
    for k := range params {
        keys = append(keys, k)
    }
    sort.Strings(keys)
    
    // 3. 拼接字符串
    var buf bytes.Buffer
    for i, k := range keys {
        if i > 0 {
            buf.WriteString("&")
        }
        buf.WriteString(k)
        buf.WriteString("=")
        buf.WriteString(params[k])
    }
    
    // 4. SHA256
    h := sha256.New()
    h.Write(buf.Bytes())
    return hex.EncodeToString(h.Sum(nil))
}
```

### 安全要求

- 时间戳与服务器时间差不超过 5 分钟
- Nonce 在 5 分钟内只能使用一次（防重放）
- IP 白名单：仅允许配置的 IP 访问
- 所有请求必须使用 HTTPS

## 📚 接口文档

### 1. 验证授权Token

**接口**: `POST /api/third-party/auth/token`

**说明**: 第三方验证用户提供的授权Token，获取用户ID等信息

**请求体**:
```json
{
  "authorization_token": "auth_app_1234_abc123def456..."
}
```

**响应示例**:
```json
{
  "code": 1,
  "message": "success",
  "data": {
    "authorization_token": "auth_app_1234_abc123def456...",
    "user_id": 10001,
    "status": "active",
    "scopes": ["deposit", "withdraw"],
    "expires_at": "2026-02-03T10:00:00Z",
    "max_uses": null,
    "used_count": 5,
    "last_used_at": "2026-01-03T09:00:00Z",
    "create_time": "2026-01-03T10:00:00Z"
  }
}
```

**说明**:
- 此接口用于第三方验证用户授权的Token是否有效
- 验证成功后返回 `user_id`，第三方可以用于后续充提操作
- `status` 可能的值：`active`（有效）、`revoked`（已撤销）、`expired`（已过期）、`exhausted`（使用次数已达上限）

### 2. 查询授权Token状态

**接口**: `GET /api/third-party/auth/token/status`

**参数**:
- `authorization_token` (必需): 授权Token

**响应示例**: 同验证授权Token

### 3. 查询账户余额

**接口**: `GET /api/third-party/account/balance`

**说明**: 查询当前第三方账户的所有资产余额

**响应示例**:
```json
{
  "code": 1,
  "message": "success",
  "data": {
    "user_id": 20001,
    "account_type": 2,
    "balances": [
      {
        "asset_symbol": "USDT",
        "available_amount": "5000.500000",
        "frozen_amount": "100.000000",
        "total_amount": "5100.500000"
      }
    ],
    "total_value_usd": "5100.50",
    "update_time": "2026-01-03T10:00:00Z"
  }
}
```

### 4. 查询账户流水

**接口**: `GET /api/third-party/account/ledgers`

**参数**:
- `asset_symbol` (可选): 资产符号，如 `USDT`
- `page` (可选): 页码，默认 1
- `page_size` (可选): 每页条数，默认 20，最大 100

**响应示例**:
```json
{
  "code": 1,
  "message": "success",
  "data": {
    "list": [
      {
        "id": 12345,
        "asset_symbol": "USDT",
        "direction": "in",
        "change_type": "third_party_deposit_in",
        "amount": "100.000000",
        "balance_after": "5100.500000",
        "ref_type": "third_party_deposit",
        "ref_id": "TPD2026010312345678",
        "memo": "接收用户充值",
        "create_time": "2026-01-03T10:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 100
    }
  }
}
```

### 5. 创建充值订单

**接口**: `POST /api/third-party/deposit/create`

**说明**: 平台用户向第三方账户充值（内部转账）

**请求体**:
```json
{
  "third_party_order_no": "D2026010312345678",
  "authorization_token": "auth_app_1234_abc123def456...",
  "asset_symbol": "USDT",
  "amount": "100.000000",
  "memo": "用户向第三方充值"
}
```

**参数说明**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `third_party_order_no` | string | 是 | 第三方订单号（用于幂等性控制，必须唯一） |
| `authorization_token` | string | 是 | 用户授权Token |
| `asset_symbol` | string | 是 | 资产符号，如 `USDT`、`USDC` |
| `amount` | string | 是 | 充值金额，精度最多18位小数 |
| `memo` | string | 否 | 备注信息 |

**响应示例**:
```json
{
  "code": 1,
  "message": "success",
  "data": {
    "order_no": "TPD2026010312345678",
    "third_party_order_no": "D2026010312345678",
    "status": "success",
    "from_user_id": 10001,
    "third_party_user_id": 20001,
    "asset_symbol": "USDT",
    "amount": "100.000000",
    "create_time": "2026-01-03T10:00:00Z",
    "completed_at": "2026-01-03T10:00:01Z"
  }
}
```

**注意事项**:
- `authorization_token` 必须是用户授权的有效Token，且授权范围包含 `deposit`
- `third_party_order_no` 必须唯一，用于幂等性控制
- 充值会立即到账（内部转账）
- 会同时记录两条流水（来源用户出账 + 第三方账户入账）

### 6. 查询充值订单

**接口**: `GET /api/third-party/deposit/query`

**参数**:
- `third_party_order_no` (必需): 第三方订单号

**响应示例**: 同创建充值订单

### 7. 创建提现订单

**接口**: `POST /api/third-party/withdraw/create`

**说明**: 第三方账户向平台用户提现（内部转账）

**请求体**:
```json
{
  "third_party_order_no": "W2026010312345678",
  "authorization_token": "auth_app_1234_xyz789abc123...",
  "asset_symbol": "USDT",
  "amount": "100.000000",
  "fee": "5.000000",
  "memo": "第三方向用户提现"
}
```

**参数说明**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `third_party_order_no` | string | 是 | 第三方订单号（用于幂等性控制，必须唯一） |
| `authorization_token` | string | 是 | 用户授权Token |
| `asset_symbol` | string | 是 | 资产符号，如 `USDT`、`USDC` |
| `amount` | string | 是 | 提现总金额（包含手续费） |
| `fee` | string | 否 | 手续费，默认 0 |
| `memo` | string | 否 | 备注信息 |

**响应示例**:
```json
{
  "code": 1,
  "message": "success",
  "data": {
    "order_no": "TPW2026010312345678",
    "third_party_order_no": "W2026010312345678",
    "status": "success",
    "third_party_user_id": 20001,
    "to_user_id": 10002,
    "asset_symbol": "USDT",
    "amount": "100.000000",
    "fee": "5.000000",
    "actual_amount": "95.000000",
    "create_time": "2026-01-03T10:00:00Z",
    "completed_at": "2026-01-03T10:00:01Z"
  }
}
```

**注意事项**:
- `authorization_token` 必须是用户授权的有效Token，且授权范围包含 `withdraw`
- `third_party_order_no` 必须唯一，用于幂等性控制
- 提现前会检查第三方账户余额，余额不足会返回错误
- `amount` 是总金额，`actual_amount = amount - fee` 是用户实际到账金额
- 提现会立即到账（内部转账）

### 8. 查询提现订单

**接口**: `GET /api/third-party/withdraw/query`

**参数**:
- `third_party_order_no` (必需): 第三方订单号

**响应示例**: 同创建提现订单

## ❌ 错误码说明

### 通用错误码

| 错误码 | 说明 | 解决方案 |
|--------|------|----------|
| 40100 | 缺少必需请求头 | 检查请求头是否完整 |
| 40101 | 时间戳格式错误 | 使用毫秒级时间戳 |
| 40102 | 应用不存在 | 检查 app_key 是否正确 |
| 40103 | 应用已禁用 | 联系平台管理员 |
| 40104 | IP 不在白名单 | 联系平台管理员添加 IP |
| 40105 | 签名验证失败 | 检查签名算法和参数 |
| 40106 | 时间戳过期 | 确保时间戳在 5 分钟内 |
| 40107 | Nonce 已使用 | 每次请求使用新的 nonce |
| 40000 | 请求参数错误 | 检查请求参数格式 |
| 40400 | 资源不存在 | 检查订单号 |
| 50000 | 服务器内部错误 | 联系技术支持 |

### 授权相关错误码

| 错误码 | 说明 | 解决方案 |
|--------|------|----------|
| 40201 | 授权Token不存在 | 检查Token是否正确 |
| 40202 | 授权Token无效 | Token与应用不匹配，用户需要重新授权 |
| 40203 | 授权已被撤销 | 用户已撤销授权，需要重新授权 |
| 40204 | 授权已过期 | Token已过期，用户需要重新授权 |
| 40205 | 授权使用次数已达上限 | 用户需要重新授权 |
| 40206 | 授权范围不足 | 用户需要授权相应的操作权限 |

### 业务相关错误码

| 错误码 | 说明 | 解决方案 |
|--------|------|----------|
| 40301 | 用户不存在 | 检查用户授权Token是否有效 |
| 40302 | 余额不足 | 检查账户余额 |
| 40303 | 资产不支持 | 检查允许的资产列表 |
| 40304 | 超过单笔限额 | 减少单笔金额 |
| 40305 | 超过每日限额 | 等待次日重试 |
| 40306 | 订单已存在 | 使用不同的订单号 |
| 40307 | 金额无效 | 检查金额格式和范围 |

## 💻 SDK 示例

### Go SDK

```go
package main

import (
    "bytes"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "sort"
    "strings"
    "time"
)

type Client struct {
    AppID     string
    AppKey    string
    AppSecret string
    BaseURL   string
    Client    *http.Client
}

func NewClient(appID, appKey, appSecret, baseURL string) *Client {
    return &Client{
        AppID:     appID,
        AppKey:    appKey,
        AppSecret: appSecret,
        BaseURL:   baseURL,
        Client:    &http.Client{Timeout: 30 * time.Second},
    }
}

// GenerateAuthorizeURL 生成授权链接
func (c *Client) GenerateAuthorizeURL(redirectURI, state string, scopes []string) string {
    params := url.Values{}
    params.Set("app_id", c.AppID)
    params.Set("redirect_uri", redirectURI)
    params.Set("state", state)
    if len(scopes) > 0 {
        params.Set("scopes", strings.Join(scopes, ","))
    }
    return fmt.Sprintf("%s/authorize?%s", c.BaseURL, params.Encode())
}

func (c *Client) generateSignature(params map[string]string) string {
    params["key"] = c.AppKey
    params["secret"] = c.AppSecret
    
    keys := make([]string, 0, len(params))
    for k := range params {
        keys = append(keys, k)
    }
    sort.Strings(keys)
    
    var buf strings.Builder
    for i, k := range keys {
        if i > 0 {
            buf.WriteString("&")
        }
        buf.WriteString(k)
        buf.WriteString("=")
        buf.WriteString(params[k])
    }
    
    h := sha256.New()
    h.Write([]byte(buf.String()))
    return hex.EncodeToString(h.Sum(nil))
}

func (c *Client) Request(method, path string, body interface{}) (*http.Response, error) {
    url := c.BaseURL + path
    
    var bodyReader io.Reader
    params := make(map[string]string)
    
    if body != nil {
        bodyBytes, _ := json.Marshal(body)
        bodyReader = bytes.NewReader(bodyBytes)
        params["body"] = string(bodyBytes)
    }
    
    timestamp := fmt.Sprintf("%d", time.Now().UnixMilli())
    nonce := fmt.Sprintf("%d", time.Now().UnixNano())
    
    params["timestamp"] = timestamp
    params["nonce"] = nonce
    
    signature := c.generateSignature(params)
    
    req, _ := http.NewRequest(method, url, bodyReader)
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-App-Key", c.AppKey)
    req.Header.Set("X-Signature", signature)
    req.Header.Set("X-Timestamp", timestamp)
    req.Header.Set("X-Nonce", nonce)
    
    return c.Client.Do(req)
}

// AuthTokenInfo 授权Token信息
type AuthTokenInfo struct {
    AuthorizationToken string  `json:"authorization_token"`
    UserID             uint64  `json:"user_id"`
    Status             string  `json:"status"`
    Scopes             []string `json:"scopes"`
}

// VerifyAuthToken 验证授权Token，获取用户信息
func (c *Client) VerifyAuthToken(token string) (*AuthTokenInfo, error) {
    req := map[string]string{
        "authorization_token": token,
    }
    
    resp, err := c.Request("POST", "/api/third-party/auth/token", req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var result struct {
        Code    int           `json:"code"`
        Message string        `json:"message"`
        Data    AuthTokenInfo `json:"data"`
    }
    
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, err
    }
    
    if result.Code != 1 {
        return nil, fmt.Errorf("error: %s", result.Message)
    }
    
    return &result.Data, nil
}

// CreateDeposit 创建充值订单（使用授权Token）
func (c *Client) CreateDeposit(authToken, orderNo, assetSymbol, amount, memo string) (*DepositResponse, error) {
    req := DepositRequest{
        ThirdPartyOrderNo:  orderNo,
        AuthorizationToken: authToken,
        AssetSymbol:        assetSymbol,
        Amount:             amount,
        Memo:               memo,
    }
    
    resp, err := c.Request("POST", "/api/third-party/deposit/create", req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var result struct {
        Code    int              `json:"code"`
        Message string           `json:"message"`
        Data    DepositResponse  `json:"data"`
    }
    
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, err
    }
    
    if result.Code != 1 {
        return nil, fmt.Errorf("error: %s", result.Message)
    }
    
    return &result.Data, nil
}

type DepositRequest struct {
    ThirdPartyOrderNo  string `json:"third_party_order_no"`
    AuthorizationToken string `json:"authorization_token"`
    AssetSymbol        string `json:"asset_symbol"`
    Amount             string `json:"amount"`
    Memo               string `json:"memo"`
}

type DepositResponse struct {
    OrderNo           string `json:"order_no"`
    ThirdPartyOrderNo string `json:"third_party_order_no"`
    Status            string `json:"status"`
    FromUserID        uint64 `json:"from_user_id"`
    ThirdPartyUserID  uint64 `json:"third_party_user_id"`
    AssetSymbol       string `json:"asset_symbol"`
    Amount            string `json:"amount"`
    CreateTime        string `json:"create_time"`
    CompletedAt       string `json:"completed_at"`
}

func main() {
    client := NewClient(
        "app_1234567890",
        "ak_your_app_key",
        "sk_your_app_secret",
        "https://api.tokentalk.cc",
    )
    
    // 第一步：生成授权链接，引导用户跳转
    authURL := client.GenerateAuthorizeURL(
        "https://your-website.com/callback",
        "random_state_123",
        []string{"deposit", "withdraw"},
    )
    fmt.Printf("请引导用户访问: %s\n", authURL)
    
    // 第二步：用户授权后，从回调地址获取 authorization_token
    // 假设从回调获取到 token
    authToken := "auth_app_1234_abc123def456..."
    
    // 第三步：验证Token，获取用户ID
    tokenInfo, err := client.VerifyAuthToken(authToken)
    if err != nil {
        fmt.Printf("验证Token失败: %v\n", err)
        return
    }
    fmt.Printf("用户ID: %d, 状态: %s\n", tokenInfo.UserID, tokenInfo.Status)
    
    // 第四步：使用Token进行充值
    resp, err := client.CreateDeposit(
        authToken,
        "D2026010312345678",
        "USDT",
        "100.000000",
        "用户充值",
    )
    
    if err != nil {
        fmt.Printf("充值失败: %v\n", err)
        return
    }
    
    fmt.Printf("订单号: %s\n", resp.OrderNo)
    fmt.Printf("状态: %s\n", resp.Status)
}
```

### Python SDK

```python
import hashlib
import json
import time
import requests
from typing import Dict, List, Optional
from urllib.parse import urlencode

class TokenTalkClient:
    def __init__(self, app_id: str, app_key: str, app_secret: str, 
                 base_url: str = "https://api.tokentalk.cc"):
        self.app_id = app_id
        self.app_key = app_key
        self.app_secret = app_secret
        self.base_url = base_url
    
    def generate_authorize_url(self, redirect_uri: str, state: str, 
                               scopes: List[str] = None) -> str:
        """生成授权链接"""
        params = {
            "app_id": self.app_id,
            "redirect_uri": redirect_uri,
            "state": state,
        }
        if scopes:
            params["scopes"] = ",".join(scopes)
        return f"{self.base_url}/authorize?{urlencode(params)}"
    
    def _generate_signature(self, params: Dict[str, str]) -> str:
        params["key"] = self.app_key
        params["secret"] = self.app_secret
        
        sorted_keys = sorted(params.keys())
        query_string = "&".join([f"{k}={params[k]}" for k in sorted_keys])
        
        return hashlib.sha256(query_string.encode()).hexdigest()
    
    def _request(self, method: str, path: str, body: Optional[Dict] = None) -> Dict:
        url = f"{self.base_url}{path}"
        params = {}
        
        if body:
            params["body"] = json.dumps(body)
        
        timestamp = str(int(time.time() * 1000))
        nonce = str(int(time.time() * 1000000))
        
        params["timestamp"] = timestamp
        params["nonce"] = nonce
        
        signature = self._generate_signature(params)
        
        headers = {
            "Content-Type": "application/json",
            "X-App-Key": self.app_key,
            "X-Signature": signature,
            "X-Timestamp": timestamp,
            "X-Nonce": nonce,
        }
        
        if method == "GET":
            resp = requests.get(url, headers=headers, params=body)
        else:
            resp = requests.post(url, headers=headers, json=body)
        
        resp.raise_for_status()
        return resp.json()
    
    def verify_auth_token(self, token: str) -> Dict:
        """验证授权Token，获取用户信息"""
        return self._request("POST", "/api/third-party/auth/token", {
            "authorization_token": token,
        })
    
    def create_deposit(self, authorization_token: str, third_party_order_no: str,
                      asset_symbol: str, amount: str, memo: str = "") -> Dict:
        """创建充值订单（使用授权Token）"""
        return self._request("POST", "/api/third-party/deposit/create", {
            "third_party_order_no": third_party_order_no,
            "authorization_token": authorization_token,
            "asset_symbol": asset_symbol,
            "amount": amount,
            "memo": memo,
        })
    
    def create_withdraw(self, authorization_token: str, third_party_order_no: str,
                       asset_symbol: str, amount: str, fee: str = "0", memo: str = "") -> Dict:
        """创建提现订单（使用授权Token）"""
        return self._request("POST", "/api/third-party/withdraw/create", {
            "third_party_order_no": third_party_order_no,
            "authorization_token": authorization_token,
            "asset_symbol": asset_symbol,
            "amount": amount,
            "fee": fee,
            "memo": memo,
        })
    
    def get_balance(self) -> Dict:
        """查询第三方账户余额"""
        return self._request("GET", "/api/third-party/account/balance")

# 使用示例
client = TokenTalkClient(
    app_id="app_1234567890",
    app_key="ak_your_app_key",
    app_secret="sk_your_app_secret"
)

# 第一步：生成授权链接，引导用户跳转
auth_url = client.generate_authorize_url(
    redirect_uri="https://your-website.com/callback",
    state="random_state_123",
    scopes=["deposit", "withdraw"]
)
print(f"请引导用户访问: {auth_url}")

# 第二步：用户授权后，从回调地址获取 authorization_token
# 假设从回调获取到 token
auth_token = "auth_app_1234_abc123def456..."

# 第三步：验证Token，获取用户ID
token_info = client.verify_auth_token(auth_token)
print(f"用户ID: {token_info['data']['user_id']}")
print(f"状态: {token_info['data']['status']}")

# 第四步：使用Token进行充值
result = client.create_deposit(
    authorization_token=auth_token,
    third_party_order_no="D2026010312345678",
    asset_symbol="USDT",
    amount="100.000000",
    memo="用户充值"
)

print(f"订单号: {result['data']['order_no']}")
print(f"状态: {result['data']['status']}")
```

## ❓ 常见问题

### Q1: 如何获取用户的授权Token？

**A**: 按照以下步骤：
1. 生成授权链接（包含 app_id、redirect_uri、state）
2. 引导用户跳转到 TokenTalk 授权页面
3. 用户在 TokenTalk 登录并确认授权
4. TokenTalk 重定向回您的 redirect_uri，URL 参数中包含 `authorization_token`
5. 调用 `POST /api/third-party/auth/token` 验证 Token 并获取 user_id

### Q2: 为什么验证Token后才能获取user_id？

**A**: 这是为了安全考虑：
- 用户自己输入 Token 到第三方网站，Token 本身不包含 user_id
- 第三方需要调用 API 验证 Token 的真实性
- 验证成功后，平台才返回 user_id，确保 Token 未被伪造

### Q3: 授权Token会过期吗？

**A**: 取决于用户授权时的设置：
- 用户可以设置过期时间（如30天）
- 用户可以设置最大使用次数
- 用户可以随时撤销授权
- 如果未设置，Token 永不过期

### Q4: 如何保证充提不会重复？

**A**: 通过 `third_party_order_no` 做幂等性控制，同一个第三方订单号只会创建一次订单。

### Q5: 充值和提现是链上交易吗？

**A**: 不是。充值和提现都是平台内部转账，即时到账，无 Gas 费用。

### Q6: 第三方账户余额不足时能提现吗？

**A**: 不能。提现前会严格校验第三方账户的可用余额，余额不足会直接返回错误。

### Q7: 手续费如何处理？

**A**: 手续费由第三方在调用接口时指定。提现时：从第三方账户扣除 `amount`，用户实际到账 `actual_amount = amount - fee`。

### Q8: 如何成为第三方应用？

**A**: 
1. 先在平台注册普通账户
2. 提交第三方应用申请（提供公司信息等）
3. 运营后台审核通过后，账户升级为第三方账户
4. 获得 API 凭证（app_id、app_key 和 app_secret）
5. 可以调用授权和充提接口

## 📞 技术支持

- **文档**: https://github.com/tokentalk-cc/tokentalk-openapi
- **问题反馈**: https://github.com/tokentalk-cc/tokentalk-openapi/issues

## 📄 更新日志

### v1.2.0 (2026-01-03)
- 优化授权流程，采用 OAuth 风格的授权机制
- 新增 `POST /api/third-party/auth/token` 接口，用于验证Token获取user_id
- 新增 `GET /api/third-party/auth/token/status` 接口
- 更新流程图和文档说明

### v1.1.0 (2026-01-02)
- 新增用户授权机制，使用 `authorization_token` 代替直接使用 `user_id`
- 更新 SDK 示例

### v1.0.0 (2026-01-01)
- 初始版本发布
- 支持充值和提现接口
- 支持余额和流水查询

## 📄 许可证

Copyright © 2026 TokenTalk. All rights reserved.
