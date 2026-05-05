# Fincore

> Motor de cálculo financeiro — projeto âncora para demonstrar maturidade de engenharia.

Fincore é um sistema de cálculo financeiro construído em camadas ao longo de 5 meses.
O domínio (finanças), as tecnologias (Go, gRPC, Kafka, Rust) e a arquitetura (eventos, observabilidade)
são diretamente relevantes para engenharia de backend em fintechs.

---

## Organização do código

```
fincore/
├── go.mod                        # module github.com/dellatsantos/fincore
├── go.sum
├── README.md
│
├── cmd/
│   └── fincore/
│       └── main.go               # entry point — só monta as peças, sem lógica
│
├── internal/
│   ├── calc/                     # lógica financeira pura, sem dependências externas
│   │   ├── simple.go
│   │   ├── compound.go
│   │   ├── sac.go
│   │   ├── price.go
│   │   └── calc_test.go
│   │
│   ├── cli/                      # parsing de argumentos e formatação de saída (Mês 1)
│   │   └── commands.go
│   │
│   ├── server/                   # servidor gRPC (Mês 2)
│   │   └── server.go
│   │
│   ├── producer/                 # publicação de eventos Kafka (Mês 3)
│   │   └── producer.go
│   │
│   └── consumer/                 # consumo e persistência dos eventos (Mês 3)
│       └── consumer.go
│
├── proto/                        # definições Protobuf (Mês 2)
│   └── fincore.proto
│
├── migrations/                   # SQL de criação de tabelas (Mês 3)
│   └── 001_create_calculations.sql
│
└── docker-compose.yml            # Kafka + Zookeeper + PostgreSQL (Mês 3)
```

### Princípio central

`internal/calc/` é o núcleo estável do sistema. Todas as camadas futuras
(gRPC, Kafka, Rust FFI) importam este pacote — ele nunca é reescrito, apenas estendido.

`internal/` impede que qualquer código externo importe os pacotes do projeto.
Isso é garantido pelo compilador Go — não é convenção, é regra da linguagem.

`cmd/` contém apenas o `main.go`, que tem uma única responsabilidade:
instanciar dependências e chamar `cli.Run()`. Sem lógica de negócio aqui.

---

## Roadmap

### Mês 1 — Go: CLI e fundações (Maio)

**Objetivo:** aprender Go construindo a base do Fincore.

- [ ] `go mod init github.com/dellatsantos/fincore`
- [ ] Implementar funções de cálculo em `internal/calc/`:
  - [ ] Juros simples: `M = C * (1 + i*t)`
  - [ ] Juros compostos: `M = C * (1 + i)^t`
  - [ ] Amortização SAC: amortização constante, prestação decrescente
  - [ ] Amortização Price: prestação constante (tabela francesa)
- [ ] Testes unitários com `testing` nativo desde o primeiro arquivo
- [ ] CLI com subcomandos em `internal/cli/`:
  ```
  fincore compound --principal 10000 --rate 0.01 --periods 12
  fincore sac --principal 12000 --rate 0.01 --periods 12
  ```
- [ ] Aprender: structs, interfaces, error handling (`errors.New`, `fmt.Errorf`), `go fmt`, `go vet`

**Critério de conclusão:** `go test ./...` passa, CLI funciona para os 4 subcomandos.

---

### Mês 2 — gRPC + Protobuf (Junho)

**Objetivo:** expor os cálculos como API gRPC.

- [ ] Definir schema em `proto/fincore.proto`:
  - [ ] `CalculateCompoundInterest(principal, rate, periods) → result`
  - [ ] `SimulateLoan(principal, rate, periods, type) → schedule []Installment`
  - [ ] `ProjectBalance(initial, monthly_rate, months) → []BalancePoint`
- [ ] Gerar código Go com `protoc`
- [ ] Implementar servidor gRPC em `internal/server/`
- [ ] Cliente Python consumindo a API (`grpcio` + `protobuf`)
- [ ] Testes de integração
- [ ] Aprender: Protobuf schema design, streaming unário vs server-streaming, interceptors para logging

**Critério de conclusão:** cliente Python chama os 3 endpoints e recebe respostas corretas.

---

### Mês 3 — Kafka: pipeline de eventos (Julho)

**Objetivo:** desacoplar o servidor com um broker de mensagens.

- [ ] Docker Compose com Kafka + Zookeeper + PostgreSQL
- [ ] Cada chamada gRPC publica um evento no tópico correspondente:
  - `loan.simulated`, `interest.calculated`, `balance.projected`
- [ ] Consumer Go separado em `internal/consumer/`:
  - [ ] Lê eventos e persiste no banco (`pgx` para PostgreSQL ou `modernc/sqlite`)
  - [ ] Idempotência: o mesmo evento não é processado duas vezes
- [ ] Aprender: producer/consumer em Go, partições, offsets, consumer groups, at-least-once delivery

**Critério de conclusão:** pipeline completo rodando via `docker-compose up`. Evento publicado pelo gRPC aparece persistido no banco.

---

### Mês 4 — Rust: reescrever o core (Agosto)

**Objetivo:** reimplementar `internal/calc/` como crate Rust independente.

- [ ] `cargo new fincore-core --lib`
- [ ] Reimplementar juros simples, compostos, SAC, Price em Rust
- [ ] Foco em: ownership, borrowing, traits (`Display`, `From`), `Result`, `Option`
- [ ] Escolher integração (decidir no momento):
  - **Opção A:** compilar como `cdylib` e chamar do Go via FFI
  - **Opção B:** expor via HTTP REST com Axum e comparar performance Go vs Rust
- [ ] Aprender: `cargo`, `rustfmt`, `clippy`, error handling com `thiserror` ou `anyhow`

**Critério de conclusão:** os mesmos cálculos do Mês 1 rodam via Rust, chamados pelo Go ou via HTTP.

---

### Mês 5 — Observabilidade + portfólio (Setembro)

**Objetivo:** tornar o projeto apresentável e observável.

- [ ] Métricas Prometheus no servidor gRPC (latência, throughput, erros por endpoint)
- [ ] Dashboard Grafana conectado ao Prometheus
- [ ] Traces com OpenTelemetry (spans para cada cálculo e publicação Kafka)
- [ ] Documentação final:
  - [ ] README com arquitetura, como rodar e exemplos de uso
  - [ ] Diagrama de arquitetura em Mermaid
  - [ ] Post técnico no Medium/Dev.to sobre uma decisão de design

**Critério de conclusão:** `docker-compose up` sobe tudo; Grafana mostra métricas reais; README permite que qualquer engenheiro rode o projeto em 5 minutos.

---

## Como rodar (Mês 1)

```bash
# Clonar e entrar no projeto
git clone https://github.com/dellatsantos/fincore
cd fincore

# Rodar os testes
go test ./...

# Usar a CLI
go run ./cmd/fincore compound --principal 10000 --rate 0.01 --periods 12
go run ./cmd/fincore sac --principal 12000 --rate 0.01 --periods 12
go run ./cmd/fincore price --principal 12000 --rate 0.01 --periods 12
```

---

## Princípios do projeto

- **Testes desde o dia 1.** Sem testes, o código não existe.
- **Commits pequenos e descritivos.** `git log` deve contar uma história.
- **Nunca deixar código comentado.** Delete ou documente.
- **README sempre atualizado** ao final de cada fase concluída.
- **Deixe uma nota** no README ou num `TODO` ao final de cada sessão — nunca chegue na próxima sem saber de onde retomar.