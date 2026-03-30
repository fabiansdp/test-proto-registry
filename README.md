# Protobuf Repository

Central protobuf schema repository for me. Proto files live in
`proto/`, and CI auto-generates Rust (prost + tonic) stubs into `gen/rust/`
on every push to `main`.

## Adding or editing a proto

1. Create a branch
2. Add/edit files under `proto/<service>/v1/`
3. Open a PR — CI will lint and check for breaking changes
4. Merge — CI regenerates stubs and tags a new version

## Consuming in a Rust service

### Track latest main

```toml
[dependencies]
my-org-proto = { git = "https://github.com/<your-org>/proto-repo", branch = "main" }
```

### Pin to a specific version

```toml
[dependencies]
my-org-proto = { git = "https://github.com/<your-org>/proto-repo", tag = "rust/v0.1.0-20260330120000" }
```

### Pull latest after a proto change

```bash
cargo update -p my-org-proto
```

### Example client usage

```rust
use my_org_proto::user::v1::{
    user_service_client::UserServiceClient,
    GetUserRequest,
};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = UserServiceClient::connect("http://[::1]:50051").await?;

    let resp = client
        .get_user(GetUserRequest { id: "user-123".into() })
        .await?;

    println!("{:?}", resp.into_inner().user);
    Ok(())
}
```

### Example server usage

```rust
use tonic::{Request, Response, Status};
use my_org_proto::user::v1::{
    user_service_server::{UserService, UserServiceServer},
    GetUserRequest, GetUserResponse, User,
    CreateUserRequest, CreateUserResponse,
    ListUsersRequest, ListUsersResponse,
    DeleteUserRequest, DeleteUserResponse,
};

#[derive(Debug, Default)]
pub struct MyUserService;

#[tonic::async_trait]
impl UserService for MyUserService {
    async fn get_user(
        &self,
        request: Request<GetUserRequest>,
    ) -> Result<Response<GetUserResponse>, Status> {
        let user_id = request.into_inner().id;
        let user = User {
            id: user_id,
            name: "Alice".into(),
            email: "alice@example.com".into(),
            role: 2, // USER_ROLE_MEMBER
            created_at: None,
            updated_at: None,
        };
        Ok(Response::new(GetUserResponse { user: Some(user) }))
    }

    async fn create_user(
        &self,
        _req: Request<CreateUserRequest>,
    ) -> Result<Response<CreateUserResponse>, Status> {
        todo!()
    }

    async fn list_users(
        &self,
        _req: Request<ListUsersRequest>,
    ) -> Result<Response<ListUsersResponse>, Status> {
        todo!()
    }

    async fn delete_user(
        &self,
        _req: Request<DeleteUserRequest>,
    ) -> Result<Response<DeleteUserResponse>, Status> {
        todo!()
    }
}
```

## Local development

### Prerequisites

```bash
# Install buf CLI
brew install bufbuild/buf/buf    # macOS
# or: https://buf.build/docs/installation

# Install Rust protoc plugins
cargo install protoc-gen-prost
cargo install protoc-gen-tonic
cargo install protoc-gen-prost-crate
```

### Generate locally

```bash
rm -rf gen/rust/src gen/rust/Cargo.toml
buf generate
cd gen/rust && cargo check
```

### Lint

```bash
buf lint
```

### Check for breaking changes against main

```bash
buf breaking --against "https://github.com/<your-org>/proto-repo.git#branch=main"
```