# throughput-lab

**Pushing HTTP servers to their throughput limits — Node.js vs. C++, benchmarked head-to-head.**

This is the umbrella project tying together the code from the *Handling 1 Million Requests per Second* video. It contains three companion repositories: two server implementations (Node.js and C++) built to serve the same route under extreme load, and a custom load-testing tool used to measure and compare them.

> 🎥 Full walkthrough: [Handling 1 Million Requests per Second](https://youtu.be/W4EwfEU8CGA)

---

## Repositories

| Repo | Description | Link |
|---|---|---|
| **node-1m-rps** | Node.js implementation of the benchmark server — Express, Fastify, and a custom "Cpeak" framework, backed by Postgres + Redis (standalone or cluster mode). | [github.com/agile8118/node-1m-rps](https://github.com/agile8118/node-1m-rps) |
| **cpp-1m-rps** | C++ rewrite of the same route using Drogon + RapidJSON, built after the Node.js version hit a ceiling on a 192-core AWS instance. | [github.com/agile8118/cpp-1m-rps](https://github.com/agile8118/cpp-1m-rps) |
| **1m-rps-tester** | The load-testing tool used to hammer both servers and record throughput, latency, and error-rate numbers. | [github.com/agile8118/1m-rps-tester](https://github.com/agile8118/1m-rps-tester) |

---

## Why this exists

The goal was simple to state and hard to achieve: serve **1,000,000 requests per second** from a single instance, on a route that returns a 30KB JSON payload.

The Node.js implementation got close but hit a wall — even with cluster mode and PM2 spreading work across cores, the single-threaded dispatcher model became the bottleneck. Rewriting the hot path in C++ with Drogon's native multi-threaded event loop (and RapidJSON instead of Drogon's default parser, which was actually *slower* than Node's JSON.stringify) is what pushed it over the line — reaching 1M RPS while moving ~40GB/s (320 Gb/s) across the network.

The reasoning is grounded in Amdahl's Law: the more of your program you can genuinely parallelize, the more a high core count actually helps. At 95% parallelizable (roughly where the Node.js cluster model lands), 192 cores buys you an ~18x speedup. At 99% parallelizable (what a native multi-threaded event loop enables), that same core count buys you ~66x.

$$\text{Speed up} = \frac{1}{(1 - p) + \frac{p}{\text{core count}}}$$

## Architecture

```
                     ┌────────────────────┐
                     │   1m-rps-tester     │
                     │  (load generator)   │
                     └─────────┬───────────┘
                               │ HTTP requests
                 ┌─────────────┴─────────────┐
                 ▼                           ▼
       ┌───────────────────┐       ┌───────────────────┐
       │   node-1m-rps      │       │   cpp-1m-rps       │
       │ Express / Fastify  │       │  Drogon + RapidJSON │
       │ / Cpeak (Node.js)  │       │       (C++)         │
       └─────────┬──────────┘       └─────────┬───────────┘
                 │                             │
        ┌────────┴────────┐                    │
        ▼                 ▼                    ▼
   ┌─────────┐      ┌──────────┐        (in-memory / same
   │ Postgres │      │  Redis   │         data path as Node)
   └─────────┘      └──────────┘
```

## Results at a glance

| Implementation | Peak throughput | Notes |
|---|---|---|
| Node.js (cluster, PM2) | Below 1M RPS | Bottlenecked by the single-threaded dispatcher handing off connections via IPC |
| C++ (Drogon + RapidJSON) | **1,000,000+ RPS** | Native multi-threaded event loop, ~40GB/s (320 Gb/s) sustained |

Benchmarked on an AWS `c8gn.48xlarge` instance (192 vCPUs).

## Getting started

Each repo is self-contained and has its own setup instructions — clone whichever one you want to run:

```bash
git clone https://github.com/agile8118/node-1m-rps.git
git clone https://github.com/agile8118/cpp-1m-rps.git
git clone https://github.com/agile8118/1m-rps-tester.git
```

**Quick summary:**

1. **node-1m-rps** — requires Node.js, Redis, and Postgres. `npm install`, seed the DB with `npm run seed`, then run `node cpeak.js` / `express.js` / `fastify.js`. Cluster mode via PM2: `pm2 start ecosystem.config.cjs`.
2. **cpp-1m-rps** — CMake-based build using Drogon and RapidJSON (bundled under `include/rapidjson`). See `CMakeLists.txt` and `do.sh` for the build/run flow.
3. **1m-rps-tester** — point it at either server and run a load test; see that repo's own README for exact usage and flags.

Full setup details, environment variables, and troubleshooting live in each repo's own README — this file is just the map.

## Tech stack

- **Languages:** JavaScript/Node.js, C++
- **Node.js frameworks:** Express, Fastify, Cpeak
- **C++ libraries:** Drogon, RapidJSON
- **Data layer:** PostgreSQL, Redis (standalone + cluster mode)
- **Process management:** PM2 (Node.js cluster mode)

## License

Check each individual repository for its license terms.

## Credits

Built by [agile8118](https://github.com/agile8118) as the companion code for the *Handling 1 Million Requests per Second* video.