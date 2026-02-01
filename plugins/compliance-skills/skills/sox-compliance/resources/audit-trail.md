# Audit Trail Implementation Patterns

## Append-Only Audit Log Table Schema

```sql
-- SOX Section 802: Immutable audit log with tamper detection
-- This table must NEVER have UPDATE or DELETE permissions granted
CREATE TABLE financial_audit_log (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    event_id        CHAR(36) NOT NULL,          -- UUID, globally unique
    timestamp_utc   DATETIME(6) NOT NULL DEFAULT NOW(6),
    user_id         VARCHAR(100) NOT NULL,       -- Unique user, never shared accounts
    user_ip         VARCHAR(45),                 -- Source IP for forensics
    session_id      VARCHAR(100),                -- Session correlation

    action          VARCHAR(100) NOT NULL,       -- e.g., 'journal_entry_created'
    entity_type     VARCHAR(100) NOT NULL,       -- e.g., 'journal_entry'
    entity_id       VARCHAR(100) NOT NULL,       -- Record identifier
    transaction_id  VARCHAR(100),                -- Groups related changes

    before_state    JSON,                        -- Full state before change
    after_state     JSON,                        -- Full state after change
    changed_fields  JSON,                        -- List of specific fields changed

    change_reason   TEXT,                        -- Business justification
    ticket_id       VARCHAR(100),                -- Link to change request

    -- Tamper detection: hash chain
    previous_hash   CHAR(64),                    -- SHA-256 of previous record
    record_hash     CHAR(64) NOT NULL,           -- SHA-256 of this record

    INDEX idx_entity (entity_type, entity_id),
    INDEX idx_user (user_id, timestamp_utc),
    INDEX idx_timestamp (timestamp_utc),
    INDEX idx_transaction (transaction_id)
) ENGINE=InnoDB;

-- CRITICAL: Revoke UPDATE and DELETE on this table for all application users
-- GRANT INSERT, SELECT ON financial_audit_log TO 'app_user'@'%';
-- No UPDATE, no DELETE
```

## Python AuditLogger with Hash Chain

```python
import hashlib
import json
from datetime import datetime, timezone

class AuditLogger:
    """
    SOX Section 802: Append-only audit log with cryptographic hash chain.
    Each record includes the hash of the previous record, creating a
    tamper-evident chain. If any record is modified or deleted, the
    chain breaks and the tampering is detectable.
    """

    def __init__(self, db_connection):
        self.db = db_connection

    def _compute_hash(self, record: dict) -> str:
        """Compute SHA-256 hash of audit record fields."""
        hashable = json.dumps({
            "event_id": record["event_id"],
            "timestamp_utc": record["timestamp_utc"],
            "user_id": record["user_id"],
            "action": record["action"],
            "entity_type": record["entity_type"],
            "entity_id": record["entity_id"],
            "before_state": record["before_state"],
            "after_state": record["after_state"],
            "previous_hash": record["previous_hash"],
        }, sort_keys=True, default=str)
        return hashlib.sha256(hashable.encode()).hexdigest()

    def _get_last_hash(self) -> str:
        """Retrieve hash of the most recent audit record."""
        result = self.db.execute(
            "SELECT record_hash FROM financial_audit_log "
            "ORDER BY id DESC LIMIT 1"
        ).fetchone()
        return result.record_hash if result else "0" * 64  # Genesis hash

    def log(self, user_id: str, action: str, entity_type: str,
            entity_id: str, before_state: dict = None,
            after_state: dict = None, change_reason: str = None,
            ticket_id: str = None, transaction_id: str = None):
        """Write an immutable, hash-chained audit record."""
        import uuid

        previous_hash = self._get_last_hash()

        record = {
            "event_id": str(uuid.uuid4()),
            "timestamp_utc": datetime.now(timezone.utc).isoformat(),
            "user_id": user_id,
            "action": action,
            "entity_type": entity_type,
            "entity_id": entity_id,
            "before_state": before_state,
            "after_state": after_state,
            "changed_fields": self._diff_fields(before_state, after_state),
            "change_reason": change_reason,
            "ticket_id": ticket_id,
            "transaction_id": transaction_id,
            "previous_hash": previous_hash,
        }
        record["record_hash"] = self._compute_hash(record)

        self.db.execute(
            "INSERT INTO financial_audit_log "
            "(event_id, timestamp_utc, user_id, action, entity_type, entity_id, "
            "transaction_id, before_state, after_state, changed_fields, "
            "change_reason, ticket_id, previous_hash, record_hash) "
            "VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)",
            (record["event_id"], record["timestamp_utc"], record["user_id"],
             record["action"], record["entity_type"], record["entity_id"],
             record["transaction_id"],
             json.dumps(record["before_state"]),
             json.dumps(record["after_state"]),
             json.dumps(record["changed_fields"]),
             record["change_reason"], record["ticket_id"],
             record["previous_hash"], record["record_hash"])
        )
        self.db.commit()

    def _diff_fields(self, before: dict, after: dict) -> list:
        """Identify which fields changed between before and after states."""
        if before is None or after is None:
            return []
        changed = []
        all_keys = set(list(before.keys()) + list(after.keys()))
        for key in all_keys:
            if before.get(key) != after.get(key):
                changed.append(key)
        return changed

    def verify_chain(self, start_id: int = None, end_id: int = None) -> dict:
        """
        SOX Section 802: Verify hash chain integrity.
        Run periodically to detect tampering.
        """
        query = "SELECT * FROM financial_audit_log ORDER BY id"
        if start_id and end_id:
            query += f" WHERE id BETWEEN {start_id} AND {end_id}"

        records = self.db.execute(query).fetchall()
        broken_links = []
        previous_hash = "0" * 64

        for record in records:
            if record.previous_hash != previous_hash:
                broken_links.append({
                    "record_id": record.id,
                    "expected_previous_hash": previous_hash,
                    "actual_previous_hash": record.previous_hash,
                })
            recomputed = self._compute_hash(record.__dict__)
            if recomputed != record.record_hash:
                broken_links.append({
                    "record_id": record.id,
                    "expected_hash": recomputed,
                    "actual_hash": record.record_hash,
                    "issue": "record_modified",
                })
            previous_hash = record.record_hash

        return {
            "records_checked": len(records),
            "chain_intact": len(broken_links) == 0,
            "broken_links": broken_links,
        }
```

## PHP Audit Logging Implementation

```php
class FinancialAuditLogger
{
    private PDO $db;

    public function __construct(PDO $db)
    {
        $this->db = $db;
    }

    public function log(
        string $userId,
        string $action,
        string $entityType,
        string $entityId,
        ?array $beforeState = null,
        ?array $afterState = null,
        ?string $changeReason = null,
        ?string $ticketId = null,
        ?string $transactionId = null
    ): void {
        $previousHash = $this->getLastHash();
        $eventId = $this->generateUuid();
        $timestamp = gmdate('Y-m-d\TH:i:s.u\Z');

        $record = [
            'event_id'       => $eventId,
            'timestamp_utc'  => $timestamp,
            'user_id'        => $userId,
            'action'         => $action,
            'entity_type'    => $entityType,
            'entity_id'      => $entityId,
            'before_state'   => $beforeState,
            'after_state'    => $afterState,
            'previous_hash'  => $previousHash,
        ];
        $recordHash = $this->computeHash($record);
        $changedFields = $this->diffFields($beforeState, $afterState);

        $stmt = $this->db->prepare(
            'INSERT INTO financial_audit_log
            (event_id, timestamp_utc, user_id, action, entity_type, entity_id,
             transaction_id, before_state, after_state, changed_fields,
             change_reason, ticket_id, previous_hash, record_hash)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)'
        );

        $stmt->execute([
            $eventId, $timestamp, $userId, $action, $entityType, $entityId,
            $transactionId,
            json_encode($beforeState),
            json_encode($afterState),
            json_encode($changedFields),
            $changeReason, $ticketId, $previousHash, $recordHash,
        ]);
    }

    private function computeHash(array $record): string
    {
        $hashable = json_encode([
            'event_id'       => $record['event_id'],
            'timestamp_utc'  => $record['timestamp_utc'],
            'user_id'        => $record['user_id'],
            'action'         => $record['action'],
            'entity_type'    => $record['entity_type'],
            'entity_id'      => $record['entity_id'],
            'before_state'   => $record['before_state'],
            'after_state'    => $record['after_state'],
            'previous_hash'  => $record['previous_hash'],
        ], JSON_THROW_ON_ERROR);

        return hash('sha256', $hashable);
    }

    private function getLastHash(): string
    {
        $stmt = $this->db->query(
            'SELECT record_hash FROM financial_audit_log ORDER BY id DESC LIMIT 1'
        );
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        return $row ? $row['record_hash'] : str_repeat('0', 64);
    }

    private function diffFields(?array $before, ?array $after): array
    {
        if ($before === null || $after === null) {
            return [];
        }
        $changed = [];
        $allKeys = array_unique(array_merge(array_keys($before), array_keys($after)));
        foreach ($allKeys as $key) {
            if (($before[$key] ?? null) !== ($after[$key] ?? null)) {
                $changed[] = $key;
            }
        }
        return $changed;
    }
}
```

## 7-Year Retention Policy Configuration

```sql
-- SOX Section 802: Retention policy
-- Financial audit logs must be retained for a minimum of 7 years
-- Partition by year for efficient archival without deletion

ALTER TABLE financial_audit_log
    PARTITION BY RANGE (YEAR(timestamp_utc)) (
        PARTITION p2024 VALUES LESS THAN (2025),
        PARTITION p2025 VALUES LESS THAN (2026),
        PARTITION p2026 VALUES LESS THAN (2027),
        PARTITION p2027 VALUES LESS THAN (2028),
        PARTITION p2028 VALUES LESS THAN (2029),
        PARTITION p2029 VALUES LESS THAN (2030),
        PARTITION p2030 VALUES LESS THAN (2031),
        PARTITION p2031 VALUES LESS THAN (2032),
        PARTITION pmax  VALUES LESS THAN MAXVALUE
    );

-- Archive old partitions to cold storage before dropping
-- NEVER drop a partition less than 7 years old
```
