# Binance Async Rust 库文档

## 简介

Binance Async 是一个非官方的 Rust 库，用于与 [Binance API](https://github.com/binance-exchange/binance-official-api-docs) 进行交互。该库采用 Rust 的异步/等待（Async/Await）特性，并注重人体工程学设计，使 API 调用更加直观和类型安全。

该库支持币安的多种产品线：
- 现货交易 (Spot)
- U本位合约 (USDM Futures)
- 币本位合约 (COINM Futures)

## 设计理念

该库采用类型驱动设计，主要特点如下：

1. **结构体驱动的请求/响应模式**：每个 API 请求都被封装为一个结构体，该结构体包含了请求所需的所有参数。
2. **类型安全**：利用 Rust 的类型系统确保 API 调用的正确性。
3. **默认值支持**：所有请求结构体都实现了 `Default` 特性，允许用户只填写必要的字段，其余字段使用默认值。
4. **模块化组织**：代码按功能模块组织，使结构清晰易懂。

## 安装

在 `Cargo.toml` 文件中添加以下依赖：

```toml
[dependencies]
binance-async = "0.3"
```

## 基本用法

### 初始化客户端

```rust
use binance_async::Binance;

// 不带 API 密钥的客户端（只能访问公共 API）
let binance = Binance::new();

// 带 API 密钥的客户端（可以访问私有 API）
let binance = Binance::with_key("your_api_key");

// 带 API 密钥和密钥的客户端（可以访问需要签名的 API）
let binance = Binance::with_key_and_secret("your_api_key", "your_api_secret");
```

### 配置客户端

```rust
use binance_async::{Binance, Config};

let mut binance = Binance::new();
let config = Config {
    recv_window: 5000, // 设置接收窗口（毫秒）
    ..Default::default()
};
binance.config(config);
```

## REST API 使用示例

### 现货交易 (Spot)

#### 测试服务器连接

```rust
use binance_async::{Binance, rest::spot::Ping};

async fn ping_server() -> Result<(), Box<dyn std::error::Error>> {
    let binance = Binance::new();
    binance.request(Ping {}).await?;
    println!("服务器连接正常");
    Ok(())
}
```

#### 获取账户信息

```rust
use binance_async::{Binance, rest::spot::GetAccount};

async fn get_account_info() -> Result<(), Box<dyn std::error::Error>> {
    let binance = Binance::with_key_and_secret("your_api_key", "your_api_secret");
    let account = binance.request(GetAccount {}).await?;
    println!("账户信息: {:?}", account);
    Ok(())
}
```

### U本位合约 (USDM Futures)

#### 下单

```rust
use binance_async::{
    Binance, 
    rest::usdm::NewOrderRequest,
    models::{OrderType, Side, TimeInForce}
};
use rust_decimal::Decimal;

async fn place_order() -> Result<(), Box<dyn std::error::Error>> {
    let binance = Binance::with_key_and_secret("your_api_key", "your_api_secret");
    
    let resp = binance
        .request(NewOrderRequest {
            symbol: "ethusdt".into(),
            r#type: OrderType::Limit,
            side: Side::Buy,
            price: Decimal::from_f64(1500.0).unwrap(),
            quantity: Decimal::from_f64(0.004).unwrap(),
            time_in_force: Some(TimeInForce::GTC),
            ..Default::default()
        })
        .await?;
        
    println!("订单已创建: {:?}", resp);
    Ok(())
}
```

#### 撤单

```rust
use binance_async::{Binance, rest::usdm::CancelOrderRequest};

async fn cancel_order(order_id: u64) -> Result<(), Box<dyn std::error::Error>> {
    let binance = Binance::with_key_and_secret("your_api_key", "your_api_secret");
    
    let resp = binance
        .request(CancelOrderRequest {
            symbol: "ethusdt".into(),
            order_id: Some(order_id),
            ..Default::default()
        })
        .await?;
        
    println!("订单已取消: {:?}", resp);
    Ok(())
}
```

#### 获取账户信息

```rust
use binance_async::{Binance, rest::usdm::AccountInformationV2};

async fn get_futures_account() -> Result<(), Box<dyn std::error::Error>> {
    let binance = Binance::with_key_and_secret("your_api_key", "your_api_secret");
    let account = binance.request(AccountInformationV2 {}).await?;
    
    println!("合约账户余额: {}", account.total_wallet_balance);
    println!("可用余额: {}", account.available_balance);
    
    // 打印所有资产
    for asset in account.assets {
        println!("资产: {}, 余额: {}", asset.asset, asset.wallet_balance);
    }
    
    // 打印所有持仓
    for position in account.positions {
        println!("持仓: {}, 数量: {}, 方向: {}", 
            position.symbol, position.position_amount, position.position_side);
    }
    
    Ok(())
}
```

#### 获取资金费率

```rust
use binance_async::{Binance, rest::usdm::FundingRate};

async fn get_funding_rate() -> Result<(), Box<dyn std::error::Error>> {
    let binance = Binance::new();
    
    let rates = binance
        .request(FundingRate {
            symbol: Some("ethusdt".into()),
            limit: Some(10),
            ..Default::default()
        })
        .await?;
        
    for rate in rates {
        println!("资金费率: {}, 时间: {}", rate.funding_rate, rate.funding_time);
    }
    
    Ok(())
}
```

## WebSocket API 使用示例

### 订阅市场数据

```rust
use binance_async::{BinanceWebsocket, websocket::usdm::WebsocketMessage};

async fn subscribe_market_data() -> Result<(), Box<dyn std::error::Error>> {
    // 创建 WebSocket 连接，订阅 ETH/USDT 的聚合交易和 SOL/USDT 的订单簿行情
    let mut ws: BinanceWebsocket<WebsocketMessage> = BinanceWebsocket::new(&[
        "ethusdt@aggTrade",
        "solusdt@bookTicker",
    ]).await?;
    
    // 接收消息
    for _ in 0..100 {
        let msg = ws.next().await.expect("ws exited")?;
        match msg {
            WebsocketMessage::AggregateTrade(trade) => {
                println!("聚合交易: {:?}", trade);
            }
            WebsocketMessage::BookTicker(ticker) => {
                println!("订单簿行情: {:?}", ticker);
            }
            _ => {}
        }
    }
    
    Ok(())
}
```

### 订阅用户数据流

```rust
use binance_async::{
    Binance, 
    BinanceWebsocket, 
    rest::usdm::StartUserDataStreamRequest,
    websocket::usdm::WebsocketMessage
};
use std::env::var;

async fn subscribe_user_data() -> Result<(), Box<dyn std::error::Error>> {
    // 创建 REST 客户端
    let binance = Binance::with_key(&var("BINANCE_KEY")?);
    
    // 获取 listen_key
    let listen_key = binance.request(StartUserDataStreamRequest {}).await?;
    
    // 创建 WebSocket 连接，订阅用户数据流
    let mut ws: BinanceWebsocket<WebsocketMessage> = BinanceWebsocket::new(&[
        listen_key.listen_key.as_str(),
    ]).await?;
    
    // 接收消息
    for _ in 0..100 {
        let msg = ws.next().await.expect("ws exited")?;
        match msg {
            WebsocketMessage::UserOrderUpdate(order) => {
                println!("订单更新: {:?}", order);
            }
            WebsocketMessage::UserAccountUpdate(account) => {
                println!("账户更新: {:?}", account);
            }
            _ => {}
        }
    }
    
    Ok(())
}
```

## 错误处理

该库使用 `anyhow` 进行错误处理和传递。所有 API 调用都会返回 `Result<T, BinanceError>`，其中 `BinanceError` 包含了详细的错误信息。

```rust
use binance_async::{Binance, rest::usdm::NewOrderRequest, models::{OrderType, Side}};
use rust_decimal::Decimal;

async fn handle_error() {
    let binance = Binance::with_key_and_secret("your_api_key", "your_api_secret");
    
    let result = binance
        .request(NewOrderRequest {
            symbol: "ethusdt".into(),
            r#type: OrderType::Limit,
            side: Side::Buy,
            price: Decimal::from_f64(1500.0).unwrap(),
            quantity: Decimal::from_f64(0.004).unwrap(),
            ..Default::default() // 缺少必要的 time_in_force 字段
        })
        .await;
        
    match result {
        Ok(resp) => {
            println!("订单已创建: {:?}", resp);
        }
        Err(e) => {
            println!("错误: {}", e);
            
            // 可以进一步处理特定类型的错误
            if let Some(binance_err) = e.downcast_ref::<binance_async::BinanceResponseError>() {
                println!("币安 API 错误码: {}, 消息: {}", binance_err.code, binance_err.msg);
            }
        }
    }
}
```

## 日志记录

该库使用 `tracing` 框架记录详细日志。您可以通过配置 `tracing` 来控制日志输出级别和格式。

```rust
use tracing_subscriber::{fmt, EnvFilter};

fn setup_logging() {
    // 设置日志级别为 debug
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env().add_directive("binance_async=debug".parse().unwrap()))
        .init();
}
```

## 扩展库功能

如果您需要使用库中未实现的 API 端点，可以轻松添加新的请求类型。使用 `define_request!` 宏可以简化这个过程：

```rust
use binance_async::models::Product;
use reqwest::Method;

// 定义新的请求和响应
crate::define_request! {
    Name => MyCustomRequest;
    Product => Product::UsdMFutures;
    Method => Method::GET;
    Endpoint => "/fapi/v1/myCustomEndpoint";
    Signed => true;
    Request => {
        pub param1: String,
        pub param2: Option<u64>,
    };
    Response => {
        pub result: String,
        pub value: u64,
    };
}

// 使用新定义的请求
async fn use_custom_request() -> Result<(), Box<dyn std::error::Error>> {
    let binance = Binance::with_key_and_secret("your_api_key", "your_api_secret");
    
    let resp = binance
        .request(MyCustomRequest {
            param1: "value".into(),
            param2: Some(123),
        })
        .await?;
        
    println!("结果: {}, 值: {}", resp.result, resp.value);
    Ok(())
}
```

## 注意事项

1. **风险警告**：这是一个个人项目，使用风险自负。加密货币投资具有高市场风险。
2. **API 限制**：请注意币安 API 的速率限制，过度请求可能导致 IP 被临时封禁。
3. **安全性**：请妥善保管您的 API 密钥和密钥，不要将其硬编码在代码中或提交到版本控制系统。

## 贡献

欢迎贡献代码！如果您想添加新功能或修复 bug，请提交 Pull Request。

## 许可证

该库采用 MIT 和 Apache-2.0 双重许可。
