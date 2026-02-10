# Operational Runbooks

Common operational procedures for Discordination.

## Table of Contents

- [Service Recovery](#service-recovery)
- [Database Operations](#database-operations)
- [Cache Operations](#cache-operations)
- [Voice Service](#voice-service)
- [File Storage](#file-storage)
- [Security Incidents](#security-incidents)

---

## Service Recovery

### API Service Not Responding

**Symptoms:** 502/503 from ALB, health check failures.

1. Check ECS task status:
   ```bash
   aws ecs describe-services --cluster discordination-prod --services api
   ```

2. Check recent logs:
   ```bash
   aws logs tail /ecs/discordination-api --since 10m
   ```

3. Check database connectivity:
   ```bash
   aws rds describe-db-instances --db-instance-identifier discordination-prod
   ```

4. Force new deployment (rolling restart):
   ```bash
   aws ecs update-service --cluster discordination-prod --service api --force-new-deployment
   ```

### Gateway WebSocket Connections Dropping

**Symptoms:** Users disconnected, presence not updating.

1. Check Gateway task health:
   ```bash
   aws ecs describe-tasks --cluster discordination-prod \
     --tasks $(aws ecs list-tasks --cluster discordination-prod --service-name gateway --query 'taskArns' --output text)
   ```

2. Check Redis pub/sub:
   ```bash
   redis-cli -u $REDIS_URL info clients
   redis-cli -u $REDIS_URL pubsub numsub
   ```

3. Check memory (Elixir BEAM memory pressure):
   ```bash
   aws cloudwatch get-metric-data --metric-data-queries '[{"Id":"mem","MetricStat":{"Metric":{"Namespace":"AWS/ECS","MetricName":"MemoryUtilization","Dimensions":[{"Name":"ServiceName","Value":"gateway"}]},"Period":300,"Stat":"Average"}}]' --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S)
   ```

4. Restart gateway with rolling deploy:
   ```bash
   aws ecs update-service --cluster discordination-prod --service gateway --force-new-deployment
   ```

---

## Database Operations

### Run Migrations

```bash
# Via ECS task
aws ecs run-task \
  --cluster discordination-prod \
  --task-definition discordination-migrate \
  --network-configuration "awsvpcConfiguration={subnets=[$PRIVATE_SUBNETS],securityGroups=[$DB_SG]}"

# Check migration task logs
aws logs tail /ecs/discordination-migrate --follow
```

### Create Database Backup

```bash
# Manual RDS snapshot
aws rds create-db-snapshot \
  --db-instance-identifier discordination-prod \
  --db-snapshot-identifier manual-$(date +%Y%m%d-%H%M%S)
```

### Restore from Backup

```bash
# List available snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier discordination-prod \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime]' \
  --output table

# Restore to new instance
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier discordination-restore \
  --db-snapshot-identifier <snapshot-id> \
  --db-instance-class db.r6g.large
```

### Connection Pool Exhaustion

**Symptoms:** "too many connections" errors.

1. Check active connections:
   ```sql
   SELECT count(*) FROM pg_stat_activity WHERE datname = 'discordination';
   SELECT state, count(*) FROM pg_stat_activity WHERE datname = 'discordination' GROUP BY state;
   ```

2. Kill idle connections:
   ```sql
   SELECT pg_terminate_backend(pid) FROM pg_stat_activity
   WHERE datname = 'discordination' AND state = 'idle' AND state_change < now() - interval '10 minutes';
   ```

3. Scale up: increase `db_max_connections` in Terraform vars.

---

## Cache Operations

### Flush Redis Cache

```bash
# Flush only the API response cache (safe)
redis-cli -u $REDIS_URL --scan --pattern "api:cache:*" | xargs redis-cli -u $REDIS_URL DEL

# Flush rate limit counters (resets all limits)
redis-cli -u $REDIS_URL --scan --pattern "rl:*" | xargs redis-cli -u $REDIS_URL DEL

# Flush everything (CAUTION: drops sessions, rate limits, etc.)
redis-cli -u $REDIS_URL FLUSHDB
```

### Redis Memory Pressure

**Symptoms:** Eviction warnings, slow responses.

1. Check memory:
   ```bash
   redis-cli -u $REDIS_URL info memory
   ```

2. Check key distribution:
   ```bash
   redis-cli -u $REDIS_URL --bigkeys
   ```

3. Scale up ElastiCache node type in Terraform.

---

## Voice Service

### LiveKit Cluster Health

```bash
# Check LiveKit service status
curl -s https://livekit.example.com/healthz

# List active rooms
livekit-cli list-rooms --url https://livekit.example.com --api-key $LK_KEY --api-secret $LK_SECRET

# List participants in a room
livekit-cli list-participants --url https://livekit.example.com --api-key $LK_KEY --api-secret $LK_SECRET --room voice:<channel_id>
```

### Kick User from Voice

```bash
livekit-cli remove-participant \
  --url https://livekit.example.com \
  --api-key $LK_KEY --api-secret $LK_SECRET \
  --room voice:<channel_id> \
  --identity <user_id>
```

---

## File Storage

### Check S3 Usage

```bash
aws s3 ls s3://discordination-files --recursive --summarize | tail -2
```

### Manual Cleanup Run

```bash
# Run cleanup daemon as one-shot
docker run --rm discordination/cleanup:latest --once --dry-run

# If dry run looks good, run for real
docker run --rm discordination/cleanup:latest --once
```

### Restore Deleted File

S3 versioning is enabled by default. To restore:

```bash
# List versions
aws s3api list-object-versions --bucket discordination-files --prefix "attachments/<file_id>"

# Restore specific version
aws s3api copy-object \
  --bucket discordination-files \
  --copy-source "discordination-files/attachments/<file_id>?versionId=<version_id>" \
  --key "attachments/<file_id>"
```

---

## Security Incidents

### Compromised User Account

1. Invalidate all sessions:
   ```sql
   DELETE FROM sessions WHERE user_id = '<user_id>';
   ```

2. Notify via Redis pub/sub to disconnect WebSocket:
   ```bash
   redis-cli -u $REDIS_URL PUBLISH "user:<user_id>" '{"type":"ForceDisconnect"}'
   ```

3. Disable account:
   ```sql
   UPDATE users SET disabled = true WHERE id = '<user_id>';
   ```

### Rate Limit Bypass

1. Check for suspicious patterns:
   ```bash
   aws logs filter-log-events --log-group-name /ecs/discordination-api \
     --filter-pattern "429" --start-time $(date -u -d '1 hour ago' +%s)000
   ```

2. Block IP at ALB:
   ```bash
   # Add IP to WAF blocklist
   aws wafv2 update-ip-set --name blocked-ips --scope REGIONAL \
     --addresses "<ip>/32" --id $IP_SET_ID --lock-token $LOCK_TOKEN
   ```

### Rotate Secrets

```bash
# Rotate JWT secret (causes all sessions to invalidate)
aws secretsmanager rotate-secret --secret-id discordination/jwt-secret

# Rotate database password
aws secretsmanager rotate-secret --secret-id discordination/db-password

# After rotating, force ECS redeployment to pick up new secrets
aws ecs update-service --cluster discordination-prod --service api --force-new-deployment
aws ecs update-service --cluster discordination-prod --service gateway --force-new-deployment
```
