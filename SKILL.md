---
name: Phoenix Reborn
description: Automated reliability and recovery system for OpenClaw services - handles failures, restarts, health checks, and rollback procedures
category: reliability
purpose: Reliability and recovery automation
tags:
  - reliability
  - recovery
  - healthcheck
  - failover
  - restart
  - monitoring
version: 1.0.0
requires:
  - docker
  - docker-compose
  - curl
  - systemd
---

# Phoenix Reborn - Reliability & Recovery Automation

Phoenix Reborn is OpenClaw's reliability automation skill that handles service failures, graceful restarts, health monitoring, and recovery procedures. Named for the mythical bird that rises from its ashes, this skill ensures OpenClaw services recover automatically from failures.

## Activation

```
/phoenix [service] [action]
/phoenix status
/phoenix check [service]
/phoenix restart [service]
/phoenix rollback [service]
/phoenix heal
```

## Actions

### status
Shows health status of all monitored services.

### check [service]
Performs health check on specified service or all services.

### restart [service]
Gracefully restarts a service with health validation.

### rollback [service]
Rolls back a service to previous stable version.

### heal
Auto-heals all detected failures across the system.

## Real Use Cases

### Use Case 1: Service Crash Recovery
**Scenario:** RPGCLAW backend crashes due to unhandled exception
```
/phoenix restart rpgclaw
```
- Detects service is down
- Stops gracefully (SIGTERM, 30s timeout)
- Waits for port release
- Restarts container
- Runs health check (3 retries, 5s interval)
- Confirms `200 OK` on `/api/health`
- Logs recovery event with timestamp

### Use Case 2: Database Connection Pool Exhaustion
**Scenario:** FlickClaw database connections spike, service becomes unresponsive
```
/phoenix check flickclaw
```
- Checks container status
- Tests DB connectivity
- Measures response time (threshold: 2000ms)
- If unhealthy: triggers restart with `FORCE_RECREATE` flag
- Clears connection pool on restart

### Use Case 3: Memory Leak Detection
**Scenario:** Long-running service shows increasing memory usage
```
/phoenix check rpgclaw
```
- Monitors container memory (via `docker stats`)
- Compares against baseline (configurable, default: 80%)
- If exceeded: logs warning, suggests restart
- Can auto-heal if `AUTO_HEAL_MEMORY=true` in config

### Use Case 4: Port Conflict After Unexpected Shutdown
**Scenario:** Container stopped abruptly, port 3000 not released
```
/phoenix restart rpgclaw
```
- Detects port bind failure
- Identifies zombie process on port
- Executes `fuser -k 3000/tcp` to free port
- Retries container start
- Validates binding succeeded

### Use Case 5: Full System Recovery (Multiple Services Down)
**Scenario:** Server reboot, multiple services failed to start
```
/phoenix heal
```
- Scans all defined services in `docker-compose.yml`
- Checks each container status
- Prioritizes dependency order (DB → Backend → Frontend)
- Starts services sequentially with health validation
- Reports final status for each service

### Use Case 6: Rollback After Bad Deploy
**Scenario:** New version of FlickClaw introduced breaking changes
```
/phoenix rollback flickclaw
```
- Reads `docker-compose.yml` for previous image tag
- Or uses `docker images` to find previous SHA
- Pulls previous image
- Recreates container with old image
- Validates health
- Notifies via logs

## Configuration

Phoenix Reborn reads from `~/.openclaw/config/phoenix.conf`:

```bash
# Health check settings
HEALTH_CHECK_PATH=/api/health
HEALTH_CHECK_TIMEOUT=5000
HEALTH_CHECK_RETRIES=3
HEALTH_CHECK_INTERVAL=5

# Recovery settings
MAX_RESTART_ATTEMPTS=3
RESTART_BACKOFF=10
AUTO_HEAL_MEMORY=true
MEMORY_THRESHOLD=80

# Timeouts
GRACEFUL_STOP_TIMEOUT=30
PORT_RELEASE_WAIT=5
```

## Specific Commands Reference

| Command | Description |
|---------|-------------|
| `docker ps --filter "name=rpgclaw" --format "{{.Status}}"` | Check container status |
| `docker exec rpgclaw curl -sf localhost:3000/api/health` | Internal health check |
| `docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}" rpgclaw` | Memory usage |
| `fuser -k 3000/tcp` | Kill process on port 3000 |
| `docker-compose -f /path/to/compose.yml up -d --force-recreate` | Force recreate service |

## Health Check Endpoints

Services must expose health endpoints for Phoenix Reborn to monitor:

| Service | Endpoint | Expected Response |
|---------|----------|-------------------|
| RPGCLAW | `http://localhost:3000/api/health` | `{"status":"ok"}` |
| FlickClaw | `http://localhost:3010/api/health` | `{"status":"ok"}` |
| PostgreSQL | `docker exec postgres pg_isready` | exit code 0 |
| Redis | `docker exec redis redis-cli ping` | PONG |

## Troubleshooting

### Issue: Health check returns 502 Bad Gateway
**Cause:** Service started but not listening on expected port
**Fix:** Check service logs: `docker logs <service>`
**Command:** `/phoenix check <service>`

### Issue: Container keeps restarting in loop
**Cause:** Application crash on startup, bad config, missing env vars
**Fix:** 
```bash
docker logs <service>
docker inspect <service> --format='{{.State.ExitCode}}'
```
Check `~/.openclaw/config/ext/` for missing environment variables

### Issue: Port already bound after restart attempt
**Cause:** Zombie process holding port
**Fix:**
```bash
fuser -k <port>/tcp
sleep 2
/phoenix restart <service>
```

### Issue: Rollback fails - image not found
**Cause:** Previous image was pruned
**Fix:** 
```bash
docker images --format "{{.Repository}}:{{.Tag}}" | grep <service>
docker pull <old-image-sha>
```

### Issue: Health check timeout on internal network
**Cause:** Container networking misconfigured
**Fix:**
```bash
docker network inspect openclaw_default
docker exec <service> curl -v localhost:<port>/health
```

## Recovery Playbook

### P1 - Complete System Outage
1. Run `/phoenix heal`
2. If heal fails: `docker-compose -f /path/to/compose.yml down`
3. Remove volumes: `docker volume rm $(docker volume ls -q)`
4. Run full stack: `docker-compose up -d`
5. Verify all: `/phoenix status`

### P2 - Single Service Degraded
1. Check logs: `docker logs <service> --tail 50`
2. Restart: `/phoenix restart <service>`
3. If recurring: rollback `/phoenix rollback <service>`
4. Escalate if 3 restarts fail in 10 minutes

### P3 - Resource Warning
1. Check metrics: `docker stats`
2. If memory > 80%: plan maintenance window
3. For immediate relief: `/phoenix restart <service>`

## Examples

### Example 1: Check All Services
```
/phoenix status
```
Output:
```
Phoenix Reborn - Service Health Status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ RPGCLAW     - healthy   (port 3000)
✓ FlickClaw   - healthy   (port 3010)
✓ PostgreSQL  - healthy   (port 5432)
✓ Redis       - healthy   (port 6379)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Last check: 2026-03-01T14:32:00Z
```

### Example 2: Restart Failing Service
```
/phoenix restart rpgclaw
```
Output:
```
⚠ RPGCLAW detected unhealthy
→ Stopping container (SIGTERM)... done
→ Waiting for port release... done
→ Starting container... done
→ Health check: ✓ passed (2/3)
✓ RPGCLAW recovered successfully
```

### Example 3: Full System Heal
```
/phoenix heal
```
Output:
```
Phoenix Reborn - Auto Heal
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
→ Scanning services... found 4
→ Priority order: postgres, redis, rpgclaw, flickclaw

1/4 Starting PostgreSQL... ✓ healthy
2/4 Starting Redis... ✓ healthy
3/4 Starting RPGCLAW... ✓ healthy
4/4 Starting FlickClaw... ✓ healthy

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ All services recovered (23s)
```

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success - service healthy/recovered |
| 1 | Service unhealthy after max retries |
| 2 | Configuration error |
| 3 | Port conflict - manual intervention required |
| 4 | Dependency service down |
| 5 | Rollback failed - image not found |

## Integration

Phoenix Reborn can be called from other skills:

```bash
# From VPS-OPS after deployment
/phoenix check rpgclaw

# From RPGCLAW-OPS after config change
/phoenix restart rpgclaw

# From FLICKCLAW-OPS after database migration
/phoenix restart flickclaw
```

---

**Remember:** Phoenix Reborn自动化恢复，但不要忽视根本原因。Always investigate why a service failed after recovery.
```