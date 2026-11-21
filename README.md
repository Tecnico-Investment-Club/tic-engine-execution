# TIC QUANT - EXECUTION ENGINE

## Abstract
This repository houses the Execution Engine for the Técnico Investment Club's Strategy Application team. The ultimate goal of this system is to act as the central, broker-agnostic execution layer that translates strategy target allocations into real market orders.

The architecture is designed to:
* Act as a concurrent, fire-and-forget gRPC server, receiving target portfolio weights from multiple strategies simultaneously and returning an immediate acknowledgment.
* Securely load per-strategy broker credentials and execution thresholds from environment variables with strict fail-fast validation.
* Translate asset identifiers between internal names and broker-specific tickers via a modular `AssetMapper`.
* Fetch ephemeral account snapshots, compute exact rebalancing deltas using `Decimal` precision, and generate ordered trade instructions (`SELL` before `BUY`).
* Route orders to external brokers through a modular `BaseBroker` interface (starting with Alpaca Paper Trading, extensible to Binance, Hyperliquid, etc.).
* Persist run-level execution history in PostgreSQL and expose a recovery RPC (`GetLastOrderId`) so strategies can safely resume their `order_id` sequence after container restarts.

---

## Core Execution Flow
Unlike a stateful trading monolith, the Execution Engine is built as a **stateless, one-way service** with respect to strategies:

* **Fire-and-Forget gRPC:** Strategies call `ExecuteWeights` with their target `Weights` and a sequential run-level `order_id`. The server validates the request, dispatches it to a background `ThreadPoolExecutor`, and immediately returns an `ExecutionResponse` acknowledgment.
* **Stateless Concurrency:** Only read-only configuration (`ExecutionSettings` and `AssetMapper`) is shared across threads. Broker instances, portfolio snapshots, and order lists are created locally per request so concurrent runs for different accounts execute safely in parallel.
* **Deterministic Rebalancing:** For each run, the orchestrator cancels open orders, fetches the current `Portfolio` normalized in USD, and runs the pure `Rebalancer`. Assets omitted from target weights generate a `SELL` order with `close_position=True` to avoid dust; deltas below `min_order_usd` are skipped.
* **Recovery & Idempotency:** Global execution runs (`strategy_id`, `order_id`, `created_at`) are stored in PostgreSQL. On startup, strategies query `GetLastOrderId` to recover their sequence, and any duplicate or out-of-sequence `order_id` is rejected before reaching the broker.

---

## Architecture & Directory Structure
The codebase strictly rejects loose scripts in favor of Object-Oriented Programming (OOP), immutability, and Dependency Injection.

* **`core/`**: Pure domain logic with zero external I/O. Contains `models.py` (`Weights`, `Position`, `Portfolio`, `OrderSide`, `Order` using `Decimal`) and `rebalancer.py` (portfolio delta calculation).
* **`protos/`**: The gRPC contract (`execution.proto`) defining `ExecutionService` and its compiled Python stubs (`execution_pb2.py`, `execution_pb2_grpc.py`).
* **`config/`**: Environment and symbol configuration. Contains `settings.py` (`StrategyConfig`, `ExecutionSettings`), `assets.json`, and `asset_mapper.py`.
* **`brokers/`**: Isolates external broker APIs. Contains the abstract base class (`base.py` defining `BaseBroker`) and concrete adapters (e.g., `alpaca.py`).
* **`server/`**: Networking and request lifecycle. Contains `grpc_server.py` (`ExecutionServer` with `ThreadPoolExecutor`) and `orchestrator.py` (stateless execution pipeline).
* **`storage/`**: The only place in the repository allowed to contain SQL code. Contains the PostgreSQL `OrderRepository` for tracking `order_history`.
* **`scripts/`**: Standalone utility scripts, including the end-to-end gRPC client used to validate paper trading runs.
* **`tests/`**: Comprehensive unit and integration test suite written with `pytest`.

---

## Team Engineering Standards
* **Branch Protection & PRs:** The `main` branch is protected. Direct pushes are blocked. All code must be developed in a separate branch (e.g., `feature/rebalancer`) and submitted via Pull Request.
* **Codeowners:** Merging is blocked until explicit review and approval from department leads, enforced by the `CODEOWNERS` file.
* **Strong Typing & Precision:** Use native Python Type Hints in all functions (e.g., `-> list[Order]`) and strictly use `Decimal`—never `float`—for all weights, prices, quantities, and USD amounts.
* **Testing:** Every module must be accompanied by `pytest` unit or integration tests (using mocked broker/gRPC layers where appropriate) before a PR can be approved.

---

## Local Setup
1. Ensure Docker and Docker Compose are installed on your machine.
2. Copy `.env.example` to `.env` at the root (never commit `.env` to GitHub) and fill in your Alpaca Paper Trading credentials and PostgreSQL settings.
3. Run `docker compose up --build` to spin up both the PostgreSQL database and the Python Execution Engine container simultaneously.
4. Run `pytest -v` locally or inside the container to verify all unit and integration tests pass.
5. Whenever `protos/execution.proto` is modified, recompile the stubs from the repository root:
   ```bash
   python -m grpc_tools.protoc -I=. --python_out=. --grpc_python_out=. protos/execution.proto