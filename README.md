# Fincore

> Financial calculation engine — anchor side project to demonstrate engineering maturity.

Fincore is a financial calculation system built in layers over 5 months.
The domain (finance), the technologies (Go, gRPC, Kafka, Rust), and the architecture
(event-driven, observability) are directly relevant to backend engineering at fintechs.

---

## Code organization

```
fincore/
├── go.mod                        # module github.com/dellatsantos/fincore
├── go.sum
├── README.md
│
├── cmd/
│   └── fincore/
│       └── main.go               # entry point — wires pieces together, no business logic
│
├── internal/
│   ├── calc/                     # pure financial logic, zero external dependencies
│   │   ├── simple.go
│   │   ├── simple_test.go
│   │   ├── compound.go
│   │   ├── compound_test.go
│   │   ├── sac.go
│   │   ├── sac_test.go
│   │   ├── price.go
│   │   └── price_test.go
│   │
│   ├── cli/                      # argument parsing and output formatting (Month 1)
│   │   └── commands.go
│   │
│   ├── server/                   # gRPC server (Month 2)
│   │   └── server.go
│   │
│   ├── producer/                 # Kafka event publishing (Month 3)
│   │   └── producer.go
│   │
│   └── consumer/                 # event consumption and persistence (Month 3)
│       └── consumer.go
│
├── proto/                        # Protobuf definitions (Month 2)
│   └── fincore.proto
│
├── migrations/                   # table creation SQL (Month 3)
│   └── 001_create_calculations.sql
│
└── docker-compose.yml            # Kafka + Zookeeper + PostgreSQL (Month 3)
```

### Core principles

`internal/calc/` is the stable core of the system. All future layers
(gRPC, Kafka, Rust FFI) import this package — it is never rewritten, only extended.

Each `.go` file in `internal/calc/` has its own `_test.go` counterpart in the same directory.
Test files use `package calc_test` (black-box testing) — they only access exported identifiers,
just like any external consumer of the package would.

`internal/` prevents any external code from importing the project's packages.
This is enforced by the Go compiler — it is not a convention, it is a language rule.

`cmd/` contains only `main.go`, whose sole responsibility is to instantiate
dependencies and call `cli.Run()`. No business logic here.

---

## Roadmap

### Month 1 — Go: CLI and foundations (May)

**Goal:** learn Go in practice by building the Fincore base.

- [ ] `go mod init github.com/dellatsantos/fincore`
- [ ] Implement calculation functions in `internal/calc/`:
  - [ ] Simple interest: `M = C * (1 + i*t)`
  - [ ] Compound interest: `M = C * (1 + i)^t`
  - [ ] SAC amortization: constant amortization, decreasing payments
  - [ ] Price amortization: constant payments (French table)
- [ ] Unit tests with the native `testing` package from the very first file
- [ ] CLI with subcommands in `internal/cli/`:
  ```
  fincore compound --principal 10000 --rate 0.01 --periods 12
  fincore sac --principal 12000 --rate 0.01 --periods 12
  ```
- [ ] Learn: structs, interfaces, idiomatic error handling (`errors.New`, `fmt.Errorf`), `go fmt`, `go vet`

**Done when:** `go test ./...` passes, CLI works for all 4 subcommands.

---

### Month 2 — gRPC + Protobuf (June)

**Goal:** expose the calculations as a gRPC API.

- [ ] Define schema in `proto/fincore.proto`:
  - [ ] `CalculateCompoundInterest(principal, rate, periods) → result`
  - [ ] `SimulateLoan(principal, rate, periods, type) → schedule []Installment`
  - [ ] `ProjectBalance(initial, monthly_rate, months) → []BalancePoint`
- [ ] Generate Go code with `protoc`
- [ ] Implement gRPC server in `internal/server/`
- [ ] Python client consuming the API (`grpcio` + `protobuf`)
- [ ] Integration tests
- [ ] Learn: Protobuf schema design, unary vs server-streaming, gRPC interceptors for logging

**Done when:** Python client calls all 3 endpoints and receives correct responses.

---

### Month 3 — Kafka: event pipeline (July)

**Goal:** decouple the server using a message broker.

- [ ] Docker Compose with Kafka + Zookeeper + PostgreSQL
- [ ] Each gRPC call publishes an event to the corresponding topic:
  - `loan.simulated`, `interest.calculated`, `balance.projected`
- [ ] Separate Go consumer in `internal/consumer/`:
  - [ ] Reads events and persists to the database (`pgx` for PostgreSQL or `modernc/sqlite`)
  - [ ] Idempotency: the same event must never be processed twice
- [ ] Learn: producer/consumer in Go, partitions, offsets, consumer groups, at-least-once delivery

**Done when:** full pipeline running via `docker-compose up`. An event published by gRPC appears persisted in the database.

---

### Month 4 — Rust: rewrite the core (August)

**Goal:** reimplement `internal/calc/` as a standalone Rust crate.

- [ ] `cargo new fincore-core --lib`
- [ ] Reimplement simple interest, compound interest, SAC, Price in Rust
- [ ] Focus on: ownership, borrowing, traits (`Display`, `From`), `Result`, `Option`
- [ ] Choose integration (decide at the time):
  - **Option A:** compile as `cdylib` and call from Go via FFI
  - **Option B:** expose via HTTP REST with Axum and benchmark Go vs Rust on the same calculation
- [ ] Learn: `cargo`, `rustfmt`, `clippy`, error handling with `thiserror` or `anyhow`

**Done when:** the same calculations from Month 1 run via Rust, called from Go or via HTTP.

---

### Month 5 — Observability + portfolio (September)

**Goal:** make the project presentable and observable.

- [ ] Prometheus metrics on the gRPC server (latency, throughput, errors per endpoint)
- [ ] Grafana dashboard connected to Prometheus
- [ ] OpenTelemetry traces (spans for each calculation and Kafka publish)
- [ ] Final documentation:
  - [ ] README with architecture, how to run, and usage examples
  - [ ] Architecture diagram in Mermaid
  - [ ] Technical post on Medium/Dev.to about a design decision

**Done when:** `docker-compose up` brings everything up; Grafana shows real metrics; README lets any engineer run the project in under 5 minutes.

---

## How to run (Month 1)

```bash
# Clone and enter the project
git clone https://github.com/dellatsantos/fincore
cd fincore

# Run all tests
go test ./...

# Use the CLI
go run ./cmd/fincore compound --principal 10000 --rate 0.01 --periods 12
go run ./cmd/fincore sac --principal 12000 --rate 0.01 --periods 12
go run ./cmd/fincore price --principal 12000 --rate 0.01 --periods 12
```

---

## Project principles

- **Tests from day one.** Without tests, the code does not exist.
- **Small, descriptive commits.** `git log` should tell a story.
- **Never leave commented-out code.** Delete it or document it.
- **README always updated** at the end of each completed phase.
- **Leave a note** in the README or a `TODO` comment at the end of each session — never come back without knowing where to resume.