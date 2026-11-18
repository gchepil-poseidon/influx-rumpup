# InfluxDB learning workspace

## Goal

This workspace is for learning to use, diagnose, and operate InfluxDB safely and confidently, with an immediate focus on the InfluxDB Cloud 1 environment used at work.

The practical outcome is to reduce incidents where schemas, writes, queries, retention settings, or operational decisions overload or destabilize the database.

## Product scope

- Treat **InfluxDB Cloud 1 (InfluxCloud 1.x)** as the primary target unless the user explicitly says otherwise.
- Cloud 1 is the legacy managed service based on **InfluxDB Enterprise v1**, using the TSM engine, databases, retention policies, and primarily InfluxQL.
- Prefer the InfluxDB Enterprise v1 / InfluxDB OSS v1 documentation when answering Cloud 1 technical questions.
- Do not silently apply InfluxDB 2, Flux, buckets, tasks, or InfluxDB 3 / SQL guidance to Cloud 1. Clearly label material about other versions and explain whether it transfers.
- If the exact product or version is unclear, help identify it before recommending production changes.

## How we learn

- Use short lessons built around real work scenarios, followed by a safe exercise and a small knowledge check.
- Explain the data model first: measurement, tag, field, point, series, shard, database, and retention policy.
- Prioritize schema design, series cardinality, write patterns, query cost, retention/downsampling, monitoring, and incident diagnosis.
- Connect every rule to its operational consequence: memory, CPU, disk, network, latency, or cardinality.
- Prefer reproducible examples and small synthetic datasets over abstract explanations.
- Maintain a running curriculum and record completed lessons, discoveries, and production-safe checklists in this workspace as they are created.

## Safety rules

- Never request or store production passwords, tokens, or connection strings in repository files. Use redacted examples and environment variables.
- Treat production access as read-only by default.
- Before any production query, estimate its time range, series scope, result size, and likely cost.
- Never run or recommend destructive or high-impact operations against production without explicit user approval and a rollback or recovery plan.
- Test uncertain queries and schema changes against local or synthetic data first.
- For an incident, separate symptoms from evidence and root-cause hypotheses. Do not guess when metrics or query plans can verify the cause.

## Documentation and research

- Prefer official InfluxData documentation and primary sources for technical claims.
- Use the configured `influxdb-docs` MCP server for documentation search when it is available.
- Always verify that a result applies to Cloud 1 / Enterprise v1 before using it.
- Cite the relevant documentation URL in lesson notes and recommendations.

## Key references

- Platform and version guide: https://docs.influxdata.com/platform/
- InfluxDB documentation MCP: https://docs.influxdata.com/platform/mcp/server/
- MCP endpoint: https://influxdb-docs.mcp.kapa.ai
- InfluxDB Enterprise v1 documentation: https://docs.influxdata.com/enterprise_influxdb/v1/
- InfluxDB OSS v1 documentation: https://docs.influxdata.com/influxdb/v1/
- Cloud 1 / Enterprise v1 write guide: https://docs.influxdata.com/enterprise_influxdb/v1/guides/write_data/
- Cloud 1 / Enterprise v1 query guide: https://docs.influxdata.com/enterprise_influxdb/v1/guides/query_data/
- Node.js v2 client guide (use only for compatible v2 endpoints and label version differences): https://docs.influxdata.com/influxdb/v2/api-guide/client-libraries/nodejs/

## Initial curriculum direction

1. Identify the exact Cloud 1 topology, limits, clients, and current failure modes.
2. Build a precise mental model of points, series, indexes, shards, and retention policies.
3. Learn line protocol and safe, efficient write batching.
4. Design schemas that control series cardinality.
5. Write bounded, explainable InfluxQL queries and inspect query cost.
6. Use retention policies and continuous queries for lifecycle management and downsampling.
7. Monitor Cloud 1 and create an incident triage checklist.
8. Review the current work schema and queries using redacted samples.

