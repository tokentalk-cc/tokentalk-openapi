# TokenTalk 第三方充提 API 文档

TokenTalk 平台为第三方开发者提供安全、高效的充提业务 API，支持平台内部资金流转。

## 📋 目录

- [快速开始](#快速开始)
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

### 3. 生成签名

每个 API 请求都需要携带签名，签名算法如下：

```go
// 1. 将所有请求参数（包括 query、body、timestamp、nonce）按 key 字典序排序
// 2. 拼接成 key1=value1&key2=value2&...&key=app_key&secret=app_secret
// 3. 计算 SHA256 哈希值作为签名
```

详细实现请参考 [SDK 示例](#sdk-示例)。

## 📡 API 概览

### 账户查询
- `GET /api/third-party/account/balance` - 查询账户余额
- `GET /api/third-party/account/ledgers` - 查询账户流水

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

### 1. 查询账户余额

**接口**: `GET /api/third-party/account/balance`

**说明**: 查询当前第三方账户的所有资产余额

**响应示例**:
```json
{
  "code": 0,
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

### 2. 查询账户流水

**接口**: `GET /api/third-party/account/ledgers`

**参数**:
- `asset_symbol` (可选): 资产符号，如 `USDT`
- `page` (可选): 页码，默认 1
- `page_size` (可选): 每页条数，默认 20，最大 100

**响应示例**:
```json
{
  "code": 0,
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

### 3. 创建充值订单

**接口**: `POST /api/third-party/deposit/create`

**说明**: 平台用户向第三方账户充值（内部转账）

**请求体**:
```json
{
  "third_party_order_no": "D2026010312345678",
  "from_user_id": 10001,
  "asset_symbol": "USDT",
  "amount": "100.000000",
  "memo": "用户向第三方充值"
}
```

**响应示例**:
```json
{
  "code": 0,
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
- `third_party_order_no` 必须唯一，用于幂等性控制
- 充值会立即到账（内部转账）
- 会同时记录两条流水（来源用户出账 + 第三方账户入账）

### 4. 查询充值订单

**接口**: `GET /api/third-party/deposit/query`

**参数**:
- `third_party_order_no` (必需): 第三方订单号

**响应示例**: 同创建充值订单

### 5. 创建提现订单

**接口**: `POST /api/third-party/withdraw/create`

**说明**: 第三方账户向平台用户提现（内部转账）

**请求体**:
```json
{
  "third_party_order_no": "W2026010312345678",
  "to_user_id": 10002,
  "asset_symbol": "USDT",
  "amount": "100.000000",
  "fee": "5.000000",
  "memo": "第三方向用户提现"
}
```

**响应示例**:
```json
{
  "code": 0,
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
- `third_party_order_no` 必须唯一，用于幂等性控制
- 提现前会检查账户余额，余额不足会返回错误
- `amount` 是总金额，`actual_amount = amount - fee` 是用户实际到账金额
- 提现会立即到账（内部转账）

### 6. 查询提现订单

**接口**: `GET /api/third-party/withdraw/query`

**参数**:
- `third_party_order_no` (必需): 第三方订单号

**响应示例**: 同创建提现订单

## ❌ 错误码说明

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
| 40400 | 资源不存在 | 检查订单号或用户ID |
| 50000 | 服务器内部错误 | 联系技术支持 |

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
    "sort"
    "strings"
    "time"
)

type Client struct {
    AppKey    string
    AppSecret string
    BaseURL   string
    Client    *http.Client
}

func NewClient(appKey, appSecret, baseURL string) *Client {
    return &Client{
        AppKey:    appKey,
        AppSecret: appSecret,
        BaseURL:   baseURL,
        Client:    &http.Client{Timeout: 30 * time.Second},
    }
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

// 创建充值订单示例
func (c *Client) CreateDeposit(req DepositRequest) (*DepositResponse, error) {
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
    
    if result.Code != 0 {
        return nil, fmt.Errorf("error: %s", result.Message)
    }
    
    return &result.Data, nil
}

type DepositRequest struct {
    ThirdPartyOrderNo string `json:"third_party_order_no"`
    FromUserID        uint64 `json:"from_user_id"`
    AssetSymbol       string `json:"asset_symbol"`
    Amount            string `json:"amount"`
    Memo              string `json:"memo"`
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
        "ak_your_app_key",
        "sk_your_app_secret",
        "https://api.tokentalk.cc",
    )
    
    resp, err := client.CreateDeposit(DepositRequest{
        ThirdPartyOrderNo: "D2026010312345678",
        FromUserID:        10001,
        AssetSymbol:       "USDT",
        Amount:            "100.000000",
        Memo:              "用户充值",
    })
    
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("Order No: %s\n", resp.OrderNo)
    fmt.Printf("Status: %s\n", resp.Status)
}
```

### Python SDK

```python
import hashlib
import hmac
import json
import time
import requests
from typing import Dict, Optional

class TokenTalkClient:
    def __init__(self, app_key: str, app_secret: str, base_url: str = "https://api.tokentalk.cc"):
        self.app_key = app_key
        self.app_secret = app_secret
        self.base_url = base_url
    
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
    
    def create_deposit(self, third_party_order_no: str, from_user_id: int, 
                      asset_symbol: str, amount: str, memo: str = "") -> Dict:
        return self._request("POST", "/api/third-party/deposit/create", {
            "third_party_order_no": third_party_order_no,
            "from_user_id": from_user_id,
            "asset_symbol": asset_symbol,
            "amount": amount,
            "memo": memo,
        })

# 使用示例
client = TokenTalkClient(
    app_key="ak_your_app_key",
    app_secret="sk_your_app_secret"
)

result = client.create_deposit(
    third_party_order_no="D2026010312345678",
    from_user_id=10001,
    asset_symbol="USDT",
    amount="100.000000",
    memo="用户充值"
)

print(f"Order No: {result['data']['order_no']}")
print(f"Status: {result['data']['status']}")
```

## ❓ 常见问题

### Q1: 如何保证充提不会重复？
**A**: 通过 `third_party_order_no` 做幂等性控制，同一个第三方订单号只会创建一次订单。

### Q2: 充值和提现是链上交易吗？
**A**: 不是。充值和提现都是平台内部转账，即时到账，无 Gas 费用。

### Q3: 第三方账户余额不足时能提现吗？
**A**: 不能。提现前会严格校验第三方账户的可用余额，余额不足会直接返回错误。

### Q4: 手续费如何处理？
**A**: 手续费由第三方在调用接口时指定。提现时：从第三方账户扣除 `amount`，用户实际到账 `actual_amount = amount - fee`。

### Q5: 如何成为第三方应用？
**A**: 
1. 先在平台注册普通账户
2. 提交第三方应用申请（提供公司信息等）
3. 运营后台审核通过后，账户升级为第三方账户
4. 获得 API 凭证（app_key 和 app_secret）
5. 可以调用充提接口

## 📞 技术支持

- **邮箱**: api-support@tokentalk.cc
- **文档**: https://github.com/tokentalk/tokentalk-openapi
- **问题反馈**: https://github.com/tokentalk/tokentalk-openapi/issues

## 📄 许可证

Copyright © 2026 TokenTalk. All rights reserved.

