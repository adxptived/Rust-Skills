# gRPC with Tonic

High-performance RPC with Protocol Buffers, HTTP/2 streaming, and generated clients/servers.

## Dependencies

```toml
[dependencies]
tonic = "0.12"
prost = "0.13"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }

[build-dependencies]
tonic-build = "0.12"
```

## Proto Definition

```protobuf
// proto/orders.proto
syntax = "proto3";

package orders;

service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (OrderResponse);
    rpc GetOrder(GetOrderRequest) returns (OrderResponse);
    rpc ListOrders(ListOrdersRequest) returns (stream OrderResponse);
    rpc StreamOrders(stream OrderRequest) returns (stream OrderStatus);
}

message CreateOrderRequest {
    string customer_id = 1;
    repeated LineItem items = 2;
}

message LineItem {
    string product_id = 1;
    int32 quantity = 2;
    double price = 3;
}

message OrderResponse {
    string order_id = 1;
    string status = 2;
    double total = 3;
    int64 created_at = 4;
}

message GetOrderRequest {
    string order_id = 1;
}

message ListOrdersRequest {
    string customer_id = 1;
    int32 page_size = 2;
    string page_token = 3;
}

message OrderRequest {
    string action = 1;
}

message OrderStatus {
    string order_id = 1;
    string status = 2;
}
```

## Build Script

```rust
// build.rs
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::compile_protos("proto/orders.proto")?;
    Ok(())
}
```

## Server Implementation

```rust
use tonic::{Request, Response, Status};
use orders::order_service_server::{OrderService, OrderServiceServer};
use orders::*;

pub mod orders {
    tonic::include_proto!("orders");
}

#[derive(Debug, Default)]
pub struct MyOrderService {
    // Store orders, connect to DB, etc.
}

#[tonic::async_trait]
impl OrderService for MyOrderService {
    async fn create_order(
        &self,
        request: Request<CreateOrderRequest>,
    ) -> Result<Response<OrderResponse>, Status> {
        let req = request.into_inner();
        tracing::info!("creating order for customer {}", req.customer_id);

        let response = OrderResponse {
            order_id: uuid::Uuid::new_v4().to_string(),
            status: "created".into(),
            total: req.items.iter().map(|i| i.price * i.quantity as f64).sum(),
            created_at: chrono::Utc::now().timestamp(),
        };

        Ok(Response::new(response))
    }

    async fn get_order(
        &self,
        request: Request<GetOrderRequest>,
    ) -> Result<Response<OrderResponse>, Status> {
        let req = request.into_inner();
        // Fetch from DB
        todo!()
    }

    /// Server streaming
    async fn list_orders(
        &self,
        request: Request<ListOrdersRequest>,
    ) -> Result<Response<Self::ListOrdersStream>, Status> {
        let req = request.into_inner();
        let orders = vec![OrderResponse::default()]; // fetch from DB
        let stream = tokio_stream::iter(orders);
        Ok(Response::new(Box::pin(stream)))
    }

    /// Bidirectional streaming
    async fn stream_orders(
        &self,
        request: Request<tonic::Streaming<OrderRequest>>,
    ) -> Result<Response<Self::StreamOrdersStream>, Status> {
        let mut stream = request.into_inner();
        let (tx, rx) = tokio::sync::mpsc::channel(128);
        let tx2 = tx.clone();

        tokio::spawn(async move {
            while let Some(Ok(req)) = stream.next().await {
                let status = OrderStatus {
                    order_id: "order_123".into(),
                    status: format!("received action: {}", req.action),
                };
                if tx.send(Ok(status)).await.is_err() {
                    break;
                }
            }
        });

        Ok(Response::new(Box::pin(tokio_stream::wrappers::ReceiverStream::new(rx))))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse()?;
    let service = MyOrderService::default();

    Server::builder()
        .add_service(OrderServiceServer::new(service))
        .serve(addr)
        .await?;

    Ok(())
}
```

## Client

```rust
use orders::order_service_client::OrderServiceClient;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = OrderServiceClient::connect("http://[::1]:50051").await?;

    // Unary call
    let request = CreateOrderRequest {
        customer_id: "cust_123".into(),
        items: vec![LineItem {
            product_id: "prod_1".into(),
            quantity: 2,
            price: 19.99,
        }],
    };

    let response = client.create_order(request).await?;
    println!("Order created: {:?}", response.into_inner());

    // Server streaming
    let mut stream = client
        .list_orders(ListOrdersRequest {
            customer_id: "cust_123".into(),
            page_size: 10,
            page_token: "".into(),
        })
        .await?
        .into_inner();

    while let Some(order) = stream.next().await {
        println!("Order: {:?}", order?);
    }

    Ok(())
}
```

## TLS for gRPC

```rust
use tonic::transport::{Server, ClientTlsConfig, Certificate};

// Server with TLS
async fn tls_server() -> Result<(), Box<dyn std::error::Error>> {
    let cert = tokio::fs::read("server.pem").await?;
    let key = tokio::fs::read("server.key").await?;

    let identity = tonic::transport::Identity::from_pem(cert, key);

    Server::builder()
        .tls_config(tonic::transport::ServerTlsConfig::new().identity(identity))?
        .add_service(OrderServiceServer::new(MyOrderService::default()))
        .serve("[::1]:50051".parse()?)
        .await?;

    Ok(())
}

// Client with TLS
async fn tls_client() -> Result<(), Box<dyn std::error::Error>> {
    let ca_cert = tokio::fs::read("ca.pem").await?;
    let tls = ClientTlsConfig::new()
        .ca_certificate(Certificate::from_pem(ca_cert))
        .domain_name("localhost");

    let channel = tonic::transport::Channel::from_static("https://[::1]:50051")
        .tls_config(tls)?
        .connect()
        .await?;

    let mut client = OrderServiceClient::new(channel);
    Ok(())
}
```

## Interceptors (Middleware for gRPC)

```rust
use tonic::service::Interceptor;
use tonic::{Request, Status};

#[derive(Clone)]
struct AuthInterceptor {
    token: String,
}

impl Interceptor for AuthInterceptor {
    fn call(&mut self, mut request: Request<()>) -> Result<Request<()>, Status> {
        request.metadata_mut().insert(
            "authorization",
            format!("Bearer {}", self.token).parse().unwrap(),
        );
        Ok(request)
    }
}

// Server-side interceptor
async fn server_with_interceptor() -> Result<(), Box<dyn std::error::Error>> {
    let interceptor = |req: Request<()>| {
        let token = req.metadata().get("authorization")
            .and_then(|v| v.to_str().ok())
            .ok_or(Status::unauthenticated("missing token"))?;
        if token != "Bearer valid-token" {
            return Err(Status::permission_denied("invalid token"));
        }
        Ok(req)
    };

    Server::builder()
        .add_service(OrderServiceServer::with_interceptor(MyOrderService::default(), interceptor))
        .serve("[::1]:50051".parse()?)
        .await?;

    Ok(())
}
```

## Error Handling

```rust
use tonic::Status;

fn map_error(e: impl std::fmt::Display) -> Status {
    Status::internal(format!("internal error: {e}"))
}

impl From<DbError> for Status {
    fn from(e: DbError) -> Self {
        match e {
            DbError::NotFound => Status::not_found("resource not found"),
            DbError::Conflict => Status::already_exists("resource already exists"),
            DbError::Timeout => Status::deadline_exceeded("database timeout"),
            DbError::Other(msg) => Status::internal(msg),
        }
    }
}

// In service:
async fn get_order(&self, request: Request<GetOrderRequest>) -> Result<Response<OrderResponse>, Status> {
    let order = self.db.find_order(&req.order_id).await.map_err(Status::from)?;
    Ok(Response::new(order))
}
```

## Deadlines and Timeouts

```rust
use tokio::time::Duration;

// Client-side timeout
let mut client = OrderServiceClient::connect("http://[::1]:50051").await?;
let request = tonic::Request::new(CreateOrderRequest {
    // ...
});
request.set_timeout(Duration::from_secs(5));

let response = client.create_order(request).await?;

// Or set on channel
let channel = tonic::transport::Channel::from_static("http://[::1]:50051")
    .timeout(Duration::from_secs(10))
    .connect()
    .await?;
```

## Anti-Patterns

- No timeout on gRPC calls (default is no timeout — hangs forever).
- Using tonic::Status::Internal for all errors instead of proper gRPC status codes.
- Blocking the event loop in `#[tonic::async_trait]` implementations.
- Not closing the server's receiving side of bidirectional streams on disconnect.
- Forgetting to pin streams with `Box::pin` for streaming responses.
- Large messages without configuring `max_decoding_message_size`.
