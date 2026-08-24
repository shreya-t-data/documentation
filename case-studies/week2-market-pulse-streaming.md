# AI/Tech Market Pulse — Real-Time Streaming Pipeline

## Problem

Batch ETL pipelines answer "what happened yesterday." Trading desks, risk platforms, and fraud systems need to know "what's happening right now" — and correlate it against multiple live signals simultaneously. This project builds that capability: a real-time pipeline ingesting live stock trades and financial news for AI/tech bellwether stocks, processing both with Apache Spark Structured Streaming, and flagging price moves that may be news-driven or simply statistically unusual.

## Architecture

Two independent Kafka producers (live trades via WebSocket, news via REST polling) feed two topics. Three Spark Structured Streaming jobs consume them: one computes 1-minute OHLC aggregates with watermarking, one scores news sentiment with a vectorized Pandas UDF, and one performs a dual-watermarked stream-stream join correlating the two streams. Results land in PostgreSQL via an idempotent `foreachBatch` upsert pattern. A separate statistical anomaly detector and alert watcher round out the alerting layer. The full stack starts with one command via a Makefile.

## Tech Stack

- **Kafka (KRaft mode)** — no Zookeeper dependency, simpler local ops
- **Spark Structured Streaming** — declarative windowing/joins over unbounded data
- **VADER** — free, local, fast sentiment scoring appropriate for a streaming loop
- **PostgreSQL** — reused existing infrastructure; sufficient for single-instance scale
- **Docker Compose + Makefile** — one-command reproducibility

## Key Decisions

1. **Separate Kafka topics per data shape** — trades and news have different volumes and consumers; joining downstream beats merging upstream.
2. **foreachBatch + staging table + upsert** — Spark's JDBC sink has no native upsert; this pattern gives idempotent writes without adopting a lakehouse format at a scale that doesn't need one.
3. **Deliberately scoped down the stream-stream join** — rather than a complex self-join for "price change vs. 5 minutes prior," correlated on temporal proximity to keep the core skill (dual-watermarked join) demonstrable without over-engineering under time constraints.
4. **Two independent alert types** — news-correlated and statistical-anomaly alerts are kept separate rather than merged, since not every real price move has a discoverable cause.

## Results

A fully working, one-command streaming pipeline: live trades and news flowing through Kafka, three concurrent Spark Structured Streaming jobs, idempotent writes to Postgres, and a working alerting layer with both news-correlated and statistical detection paths.

## What I Learned

Debugging this pipeline surfaced real production-relevant lessons: Kafka's `advertised.listeners` needs to resolve correctly for every network a client sits on (host vs. Docker-internal clients often need separate listener definitions); `startingOffsets` semantics materially affect whether a stream-stream join can ever produce matches; streaming checkpoints can mask logic changes between development runs; and code with import-time side effects (like a live Kafka connection) breaks unit testing in CI environments without that infrastructure.

## GitHub

https://github.com/shreya-t-data/ai-market-pulse-streaming
