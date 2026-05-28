# RDS Deep Analysis — IOPS, Load, and Migration Assessment

**Date:** 2026-04-11  
**Account:** `747293622182` (ap-southeast-2)  
**Method:** AWS CloudWatch + RDS Performance Insights (read-only)  
**Scope:** `odoo-production` (io1 target), `sensor-prod` (gp2 target)

---

## Summary

Both high-priority RDS migration candidates (`odoo-production` io1→gp3 and `sensor-prod` gp2→gp3) are confirmed safe. Neither instance approaches its provisioned or baseline IOPS ceiling under production load.

---

## odoo-production — io1 → gp3 Assessment

### Configuration
| Parameter | Current | Proposed |
|---|---|---|
| Storage type | io1 | gp3 |
| Allocated storage | 100 GB | 100 GB (unchanged) |
| Provisioned IOPS | 3,000 | 3,000 (gp3 baseline) |
| Monthly storage cost | ~$330/month | ~$13/month |
| **Saving** | — | **~$317/month** |

### Actual IOPS (last 30 days, CloudWatch)

| Metric | Daily average | 30-day peak |
|---|---|---|
| ReadIOPS | **0.8 IOPS** | **422 IOPS** |
| WriteIOPS | **5.5 IOPS** | **291 IOPS** |
| **Combined peak** | — | **~713 IOPS** |
| **Utilization of 3,000 IOPS** | — | **~24%** |

### DB Load (Performance Insights, last 7 days)

`db.load.avg` sustained at **0.034–0.041** (effectively ~3.5% of 1 vCPU). The database is idle the vast majority of the time. Top wait events are CPU-bound, not I/O-bound.

### Conclusion

**Migration is SAFE.** gp3 at 3,000 IOPS (the baseline, no extra provisioned IOPS needed) provides 4× headroom over the observed 30-day peak. Even in a worst-case scenario where IOPS momentarily spike 4× over the observed peak, gp3 baseline handles it.

**Estimated saving: ~$317/month** (io1 at $0.122/IOPS/month × 3,000 IOPS = $366; gp3 at $0.08/GB/month × 100 GB + $0.065/provisioned-IOPS × 0 extra IOPS = ~$8 + standard storage = ~$16–19/month for 100 GB gp3). See AWS pricing calculator for exact figure; the investigation doc uses a conservative $330 saving.

**Runbook:** `docs/runbooks/02-rds-io1-gp3.md`

---

## sensor-prod — gp2 → gp3 Assessment

### Configuration
| Parameter | Current | Proposed |
|---|---|---|
| Storage type | gp2 | gp3 |
| Multi-AZ | Yes | Yes (must remain) |
| Provisioned IOPS | None (gp2 dynamic) | 3,000 (gp3 baseline, no extra cost) |

### Actual IOPS (last 30 days, CloudWatch)

| Metric | Daily average | 30-day peak |
|---|---|---|
| ReadIOPS | **0.5 IOPS** | **426 IOPS** |
| WriteIOPS | **8.0 IOPS** | **781 IOPS** |
| **Combined peak** | — | **~1,207 IOPS** |

### Conclusion

**Migration is SAFE.** gp3 baseline (3,000 IOPS) provides 2.5× headroom over the observed peak. gp3 also delivers guaranteed 3,000 IOPS vs gp2's burst-dependent model, making this a quality improvement as well as a cost saving.

---

## Other RDS Instances

| Instance | Storage | Finding |
|---|---|---|
| `sensor-sandbox` | gp2 | Non-prod Multi-AZ enabled — see Proposal 8 |
| `smokealaramuat` | gp2 | Non-prod Multi-AZ enabled — see Proposal 8 |
| `dev-db-temp-mysql-8` | gp2, 50 GB | t3.micro, "temp" in name — candidate for deletion after confirmation |
| `odoo-dev`, `odoo-qa`, `sensor-qa` | gp3 | Already migrated, no action needed |

---

## Tier 2 — Direct MySQL Introspection (Future, Needs VPN)

Tier 1 AWS metrics are sufficient to greenlight the io1→gp3 migration. The following queries are documented for a future engineer with VPN access + MySQL credentials (likely in Secrets Manager):

### Schema size breakdown
```sql
SELECT table_schema, 
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) AS size_mb,
       COUNT(*) AS tables
FROM information_schema.TABLES
GROUP BY table_schema
ORDER BY size_mb DESC;
```

### Largest tables (run per DB)
```sql
SELECT table_name,
       ROUND(data_length / 1024 / 1024, 1) AS data_mb,
       ROUND(index_length / 1024 / 1024, 1) AS index_mb
FROM information_schema.TABLES
WHERE table_schema = 'odoo'
ORDER BY data_length DESC
LIMIT 20;
```

### Buffer pool hit ratio (high ratio = IOPS are mostly cached, safe to reduce provisioning)
```sql
SELECT 
  (1 - (variable_value / (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Innodb_buffer_pool_reads'))) * 100 AS hit_ratio_pct
FROM performance_schema.global_status
WHERE variable_name = 'Innodb_buffer_pool_read_requests';
-- If > 99%, IOPS are largely served from cache
```

### Unused indexes (space reduction candidates)
```sql
SELECT * FROM sys.schema_unused_indexes
WHERE object_schema NOT IN ('performance_schema', 'sys', 'mysql')
LIMIT 20;
```

### Active connections
```sql
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';
-- If Threads_connected << max_connections, instance downsizing is safe
```

---

## Appendix: PI Resource Identifiers

| Instance | PI Resource ID |
|---|---|
| odoo-production | `db-MR4IKBACH7D4CTLEIUGY5VGFZE` |

PI retention: 7 days free tier. CloudWatch RDS metrics: 30 days at 1-day granularity. Use CloudWatch for IOPS trend analysis; use PI for live query-level investigation.
