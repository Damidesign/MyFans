# ADR-001: Monolithic vs. Distributed Backend Architecture

**Status:** ✅ Accepted  
**Date:** 2026-04-22  
**Deciders:** Technical Team  
**Affected Components:** `backend/`, CI/CD, deployment, infrastructure  

---

## Context

The MyFans platform requires a backend service to handle multiple cross-cutting concerns:
- Authentication and session management
- Creator and fan API endpoints
- Content metadata and IPFS references
- Webhooks and event subscriptions
- On-chain event indexing and analytics
- Real-time notifications
- Subscription state caching and reconciliation

**Options Considered:**

1. **Monolithic Nest.js backend** (current: `backend/`)
   - Single deployable unit, shared database, unified codebase
   - All concerns in one container/process

2. **Distributed microservices** (separate API, Indexer, Notification servers)
   - Independent scaling per service
   - Fault isolation and independent deployment cycles
   - Requires eventual consistency handling and inter-service communication

3. **Hybrid approach** (API tier + async event processors with shared DB)
   - API handles synchronous requests; async workers process events
   - Shared PostgreSQL for coordination and state

---

## Decision

**We choose Option 1: Monolithic Nest.js backend** deployed as a single service.

### Rationale

1. **Startup velocity**: Single codebase, shared domain models, unified type system (TypeScript). Easier to reason about state transitions and data consistency.

2. **Stellar/Soroban constraints**: 
   - Subscription lifecycle and payment settlement have tight coupling.
   - Low-latency contract state queries and event processing benefit from co-located services.
   - Authorization checks (e.g., "is subscriber?") must be consistent with on-chain state; easier with single DB.

3. **Operational simplicity**: 
   - One deployable unit, one database, one monitoring footprint.
   - Backup/restore, schema migrations, and secret management are straightforward.
   - Reduced complexity in CI/CD, logging aggregation, and debugging.

4. **Subscription model fit**: 
   - Predictable, subscription-based workloads do not currently justify service splitting.
   - Event volume (new subscriptions, renewals, cancellations) is manageable in a single process with async queue (Bull/Redis).

5. **Development experience**: 
   - Developers pull one repo, run `npm install && npm run start:dev`, and have full platform running.
   - No RPC overhead during development or testing.

### Trade-offs Accepted

- **Scaling limitation**: If per-endpoint throughput (e.g., authentication, API endpoints) saturates single-instance CPU/memory, refactoring is required. Trigger: sustained p95 latency >500ms or error rate >0.1%.
- **Deployment coupling**: All services (auth, API, indexer) deploy together. Cannot hotfix a single endpoint without redeploying everything.
- **Monitoring discipline**: Single logs/metrics stream requires discipline to tag by subdomain (auth, indexer, api) for easy filtering.
- **Indexer resilience**: Event indexer and API are co-located. If indexer crashes, it can cascade if not properly isolated via NestJS module boundaries and error handling.

---

## Consequences

### Positive
✅ **Reduced operational complexity**: One deployment, one DB, one monitoring stack.  
✅ **Faster iteration**: Shared types, schemas, and business logic across auth, API, and indexer.  
✅ **Easier local development**: `npm install && npm run start:dev` in one folder; full platform runs locally.  
✅ **Data consistency**: ACID guarantees via single database; no distributed transaction complexity.  
✅ **Cost-effective**: One container, one database instance, lower infrastructure overhead initially.  

### Negative
❌ **Scaling limitation**: Single instance cannot scale auth independently from indexer or API.  
❌ **Deployment coupling**: Risk of cascading failures if one module fails (e.g., indexer crash impacts API availability).  
❌ **Monitoring burden**: Requires discipline to separate concerns in logs/metrics; harder to drill down by service.  
❌ **Memory footprint**: All concerns loaded in memory; no ability to shed unnecessary code in a lean service.  

### Mitigation Strategies

1. **Code organization**: Enforce strict NestJS module boundaries:
   ```
   src/
     auth/           – authentication only (guards, strategies, services)
     api/            – REST endpoints (users, creators, subscriptions)
     indexer/        – event polling and persistence (services, pollers)
     common/         – shared utilities, config, error handling
   ```

2. **Monitoring & logging**: Tag all logs by module/context:
   ```typescript
   // Example: logger.log('Payment indexed', 'indexer');
   logger.setContext('auth');      // in auth module
   logger.setContext('indexer');   // in indexer module
   ```

3. **Error handling**: Indexer failures (e.g., Soroban RPC failure) must not crash the API server:
   ```typescript
   // In event poller, catch and log; do NOT rethrow
   try {
     await this.pollContractEvents();
   } catch (err) {
     logger.warn(`Event polling failed: ${err.message}`, 'indexer');
     // Metrics: increment indexer_poll_failures
   }
   ```

4. **Load testing**: Before launch, verify single instance can handle expected load:
   - Auth requests: 100 req/s
   - API queries: 200 req/s
   - Indexer polling: <1% CPU
   - **Target**: p95 latency <200ms, p99 <500ms

---

## Future Considerations

### ADR-002 (Proposed): Event Indexer as Async Sidecar
**Trigger**: If event indexing latency exceeds 30 seconds or if memory usage consistently exceeds 1GB.  
**Decision**: Move indexer to separate service using Bull/Redis job queue and webhook callbacks from Soroban RPC.  
**Benefit**: Decouples event processing from API availability.  

### Scaling Strategy
If throughput demands grow:
1. Add instance 2, 3, etc. with round-robin load balancer (all running same monolith).
2. If **single endpoint** (e.g., auth) becomes bottleneck, refactor into separate service (ADR-003).
3. Keep database as single source of truth; use read replicas if needed.

---

## Related Decisions

- **ARCHITECTURE.md**: High-level diagram and service interaction model.
- **backend/src/**: Module structure comments and boundaries.
- **SETUP_STATUS.md**: Deployment checklist and scaling indicators.

---

## References

- [NestJS Modules Documentation](https://docs.nestjs.com/modules)
- [Stellar RPC Rate Limiting](https://developers.stellar.org/docs/reference/rate-limiting)
- [PostgreSQL ACID Properties](https://en.wikipedia.org/wiki/ACID)
- Related GitHub discussions: (to be filled in as issues emerge)
