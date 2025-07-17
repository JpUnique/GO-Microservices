# Go Microservices Demo

## Overview

This project is a demonstration of a microservices architecture built using Go (Golang). It showcases how to structure independent services that communicate with each other and how to expose a unified API to the outside world using an API Gateway.

The primary technologies used are:
*   **Go:** For building high-performance, concurrent backend services.
*   **gRPC & Protobuf:** For efficient, strongly-typed, and language-agnostic inter-service communication.
*   **GraphQL:** For a flexible and powerful client-facing API, implemented with `gqlgen`.
*   **PostgreSQL:** As the data store for the `account` service.
*   **Docker:** For containerization and easy setup of dependencies like the database.

## Why Go for Microservices?

Go is an excellent choice for microservices for several key reasons, which this project aims to illustrate:

-   **High Performance:** Go compiles to a single, native binary, offering C-like performance with faster startup times and lower resource consumption compared to interpreted languages.
-   **Built-in Concurrency:** Go's goroutines and channels make it incredibly simple to handle thousands of concurrent requests, a fundamental requirement for scalable backend services.
-   **Static Binaries:** Deployment is simplified to copying a single file. This leads to small, secure Docker images and faster deployment cycles.
-   **Strong Standard Library:** Go's standard library provides robust support for networking (HTTP, RPC) and other common tasks, reducing the need for heavy external frameworks.

## Architecture

The system is designed following the **API Gateway** pattern. Clients interact with a single GraphQL endpoint, which then delegates requests to the appropriate backend microservices.

```
+-----------+       +-------------------+       +---------------------+
|           |       |                   | ----> |  Account Service    |
|  Client   | ----> |  GraphQL Gateway  |       | (gRPC, PostgreSQL)  |
| (Browser) |       |  (gqlgen, HTTP)   |       +---------------------+
|           |       |                   |
+-----------+       +-------------------+       +---------------------+
                                        | ----> | (Planned) Catalog   |
                                        |       | Service (gRPC)      |
                                        |       +---------------------+
                                        |
                                        |       +---------------------+
                                        | ----> | (Planned) Order     |
                                                | Service (gRPC)      |
                                                +---------------------+
```

### Services

-   **GraphQL Gateway (`/graphql`)**: The public-facing entry point. It exposes a GraphQL API and is responsible for querying the various backend microservices to fulfill client requests.
-   **Account Service (`/account`)**: Manages user accounts. It has its own PostgreSQL database and exposes its functionality via a gRPC API defined in `account.proto`.

## Getting Started

Follow these instructions to get the project running on your local machine.

### Prerequisites

-   Go (version 1.23 or newer)
-   Docker
-   protoc (the Protocol Buffers compiler)
-   `protoc-gen-go` and `protoc-gen-go-grpc` Go plugins:
    ```sh
    go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
    go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
    ```
    Ensure your `$GOPATH/bin` directory (usually `~/go/bin`) is in your system's `PATH`.

### 1. Setup Database

You can easily start a PostgreSQL instance using Docker.

```sh
docker run --name go-microservices-db -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=mysecretpassword -p 5432:5432 -d postgres:14-alpine
```

Next, connect to the database (using a tool like `psql` or a GUI client) and create the `accounts` table.

```sql
CREATE TABLE accounts (
    id VARCHAR(255) PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);
```

### 2. Generate gRPC Code

Before running the services, you need to generate the Go code from the `.proto` definition. From the root of the project, run:

```sh
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    account/account.proto
```
This will generate/update the `account.pb.go` and `account_grpc.pb.go` files inside the `account/pb` directory path.

### 3. Run the Services

You'll need two separate terminal windows to run the services.

**Terminal 1: Run the Account Service**

```sh
# Navigate to the account service directory
cd account/cmd/account

# Set the environment variable for the database URL
export DATABASE_URL="postgres://postgres:mysecretpassword@localhost:5432/postgres?sslmode=disable"

# Run the service
go run .
```
You should see the output `Listening to port 50051....`.

**Terminal 2: Run the GraphQL Gateway**

```sh
# Navigate to the graphql gateway directory
cd graphql

# Set environment variables for the service URLs
export ACCOUNT_SERVICE_URL="localhost:50051"
export CATALOG_SERVICE_URL="localhost:8081" # dummy
export ORDER_SERVICE_URL="localhost:8082"   # dummy

# Run the server
go run .
```
The server will start on port `8080`.

### 4. Use the API

Once the GraphQL server is running, you can access the GraphQL Playground in your browser at:

**http://localhost:8080/playground**

You can test the API with queries and mutations. Note that the resolvers are not yet implemented, so these will not return data until that step is complete.

**Create an Account (Mutation):**
```graphql
mutation {
  createAccount(account: {
    name: "John Doe"
  }) {
    id
    name
  }
}
```

**Get Accounts (Query):**
```graphql
query {
  accounts {
    id
    name
  }
}
```

