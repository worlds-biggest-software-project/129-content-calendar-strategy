# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Content Calendar & Strategy · Created: 2026-05-19

## Philosophy

This model treats every state change as an immutable event appended to a central event store. The event store is the single source of truth; all queryable state is derived by replaying events into materialised read models (projections). This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from optimised projections.

For a content calendar platform, this architecture is compelling because content goes through a rich lifecycle (idea, brief, draft, review, revision, approval, scheduled, published, analysed) and stakeholders frequently need to answer temporal questions: "Who changed the publish date?", "What did the brief look like before the last edit?", "Show me every piece of content that was approved and then un-approved in Q3." Event sourcing answers these naturally, because the full history is preserved by design.

The pattern is used by financial trading platforms (every trade is an event), healthcare record systems (every clinical encounter is an event), and content-heavy platforms like Contentful (which uses event-driven architecture internally). It produces a complete audit trail as a side effect of normal operation, which satisfies compliance requirements without bolt-on audit tables.

**Best for:** Organisations that need full audit trails, temporal queries, AI-driven analytics on content lifecycle patterns, and the ability to replay history to train prediction models.

**Trade-offs:**
- Pro: Complete, immutable audit trail as a natural byproduct
- Pro: Temporal queries ("what was true on date X?") are first-class operations
- Pro: Event replay enables training AI models on historical content lifecycle patterns
- Pro: Read models can be optimised independently for different query patterns
- Pro: New projections can be created retroactively from the event history
- Con: Higher storage requirements (events are never deleted)
- Con: More complex to implement than direct CRUD
- Con: Eventually consistent read models require careful UX handling
- Con: Event schema evolution needs versioning discipline
- Con: Debugging requires understanding event replay, not just current state

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 (CNCF) | Event envelope format: every event in the store follows CloudEvents structure (type, source, subject, time, data) |
| W3C Activity Streams 2.0 | Event `type` vocabulary draws from Activity Streams verbs (Create, Update, Approve, Publish) |
| RFC 5545 (iCalendar) | Calendar projection maps events to iCalendar VEVENT properties for export |
| schema.org CreativeWork | Content projection fields align with CreativeWork vocabulary |
| ISO 8601 | All event timestamps in ISO 8601 TIMESTAMPTZ; durations in ISO 8601 format |
| AsyncAPI 3.0 | Event types documented as AsyncAPI channel definitions for external consumers |
| PESO Model | Channel categorisation embedded in channel-related events and projections |
| OWASP API Security | Event store access controlled via API gateway; projections expose read-only endpoints |

---

## Event Store (Source of Truth)

```sql
-- The central event store: append-only, immutable
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- CloudEvents 1.0 envelope fields
    event_type      TEXT NOT NULL,           -- e.g., 'content.created', 'content.brief_generated', 'post.scheduled'
    event_source    TEXT NOT NULL,           -- e.g., '/workspaces/{id}/content/{id}'
    subject         TEXT,                    -- The entity this event relates to
    -- Aggregate identification
    aggregate_type  TEXT NOT NULL,           -- 'content_piece', 'campaign', 'channel', 'workspace'
    aggregate_id    UUID NOT NULL,           -- The entity being modified
    workspace_id    UUID NOT NULL,           -- Tenant scoping
    -- Event metadata
    actor_id        UUID NOT NULL,           -- User or system that caused the event
    actor_type      TEXT NOT NULL DEFAULT 'user' CHECK (actor_type IN ('user', 'system', 'ai', 'webhook')),
    sequence_number BIGINT NOT NULL,         -- Monotonically increasing per aggregate
    -- Event payload
    data            JSONB NOT NULL,          -- The event-specific payload
    metadata        JSONB NOT NULL DEFAULT '{}', -- Correlation IDs, request context, etc.
    -- Timestamp
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Primary query pattern: replay events for an aggregate in order
CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence_number);

-- Workspace-scoped queries (all events in a tenant)
CREATE INDEX idx_events_workspace ON events(workspace_id, occurred_at DESC);

-- Event type queries (e.g., "all approval events across workspace")
CREATE INDEX idx_events_type ON events(workspace_id, event_type, occurred_at DESC);

-- Uniqueness constraint: one sequence number per aggregate
CREATE UNIQUE INDEX idx_events_aggregate_seq ON events(aggregate_id, sequence_number);

-- Partition by month for performance at scale
-- CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Event Type Catalogue

```
-- Content lifecycle events
content.created              -- New content piece created
content.updated              -- Title, body, type, or metadata changed
content.brief_generated      -- AI brief attached to content
content.brief_updated        -- Brief revised
content.assigned             -- Assignee changed
content.status_changed       -- Status transition (idea → briefed → in_progress → ...)
content.tagged               -- Tag added
content.untagged             -- Tag removed
content.archived             -- Moved to archive

-- Scheduling events
post.scheduled               -- Post scheduled for a channel
post.rescheduled             -- Scheduled time changed
post.publishing              -- Publish attempt started
post.published               -- Successfully published to platform
post.failed                  -- Publish attempt failed
post.cancelled               -- Scheduled post cancelled

-- Approval events
approval.submitted           -- Content submitted for approval
approval.step_approved       -- Approval step passed
approval.step_rejected       -- Approval step rejected
approval.revision_requested  -- Reviewer requested changes
approval.completed           -- All steps approved
approval.cancelled           -- Approval request cancelled

-- Analytics events
metrics.collected            -- Engagement metrics snapshot captured
prediction.generated         -- AI performance prediction created

-- Campaign events
campaign.created
campaign.updated
campaign.activated
campaign.completed

-- Channel events
channel.connected            -- Social platform OAuth connected
channel.disconnected         -- Connection revoked
channel.token_refreshed      -- OAuth token refreshed

-- Workspace events
workspace.created
workspace.member_added
workspace.member_removed
workspace.member_role_changed
workspace.settings_updated
```

### Example Event Payloads

```sql
-- content.created event
-- {
--   "title": "Q3 Product Launch Blog Post",
--   "content_type": "blog_post",
--   "status": "idea",
--   "campaign_id": "a1b2c3d4-..."
-- }

-- content.brief_generated event
-- {
--   "brief_id": "e5f6g7h8-...",
--   "topic": "Q3 Product Launch",
--   "target_persona": "B2B SaaS Decision Maker",
--   "target_keywords": ["product launch", "enterprise features"],
--   "keyword_intent": "commercial",
--   "recommended_structure": "Problem → Solution → Features → CTA",
--   "tone_guidelines": "Professional but approachable",
--   "model_version": "brief-gen-v2.1"
-- }

-- post.scheduled event
-- {
--   "channel_id": "c9d0e1f2-...",
--   "scheduled_at": "2026-06-15T14:00:00Z",
--   "caption": "Exciting news! Our Q3 launch brings...",
--   "hashtags": ["#productlaunch", "#saas"],
--   "media_ids": ["m3n4o5p6-..."]
-- }

-- approval.step_approved event
-- {
--   "request_id": "r7s8t9u0-...",
--   "step_order": 2,
--   "step_name": "Legal Review",
--   "decision": "approved",
--   "comment": "Copy is compliant. Approved."
-- }
```

---

## Materialised Read Models (Projections)

These tables are derived from the event store and can be rebuilt at any time by replaying events.

```sql
-- Projection: Current state of content pieces
CREATE TABLE proj_content_pieces (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    title           TEXT NOT NULL,
    content_type    TEXT NOT NULL,
    status          TEXT NOT NULL,
    body            TEXT,
    excerpt         TEXT,
    author_id       UUID,
    assignee_id     UUID,
    campaign_id     UUID,
    brief_id        UUID,
    publish_date    TIMESTAMPTZ,
    tags            TEXT[] NOT NULL DEFAULT '{}',
    last_event_seq  BIGINT NOT NULL,        -- Tracks which event this projection is current to
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_content_ws ON proj_content_pieces(workspace_id);
CREATE INDEX idx_proj_content_status ON proj_content_pieces(workspace_id, status);
CREATE INDEX idx_proj_content_publish ON proj_content_pieces(workspace_id, publish_date);

-- Projection: Content calendar view (optimised for calendar rendering)
CREATE TABLE proj_calendar_entries (
    id              UUID PRIMARY KEY,       -- scheduled_post aggregate ID
    workspace_id    UUID NOT NULL,
    content_piece_id UUID NOT NULL,
    channel_id      UUID NOT NULL,
    channel_name    TEXT NOT NULL,           -- Denormalised for read performance
    platform        TEXT NOT NULL,
    title           TEXT NOT NULL,           -- Denormalised from content piece
    content_type    TEXT NOT NULL,
    scheduled_at    TIMESTAMPTZ NOT NULL,
    published_at    TIMESTAMPTZ,
    status          TEXT NOT NULL,
    peso_category   TEXT NOT NULL,
    campaign_name   TEXT,
    author_name     TEXT,
    thumbnail_url   TEXT,
    last_event_seq  BIGINT NOT NULL
);

CREATE INDEX idx_proj_calendar_ws ON proj_calendar_entries(workspace_id, scheduled_at);
CREATE INDEX idx_proj_calendar_channel ON proj_calendar_entries(channel_id, scheduled_at);

-- Projection: Approval dashboard
CREATE TABLE proj_approval_queue (
    id              UUID PRIMARY KEY,       -- approval_request aggregate ID
    workspace_id    UUID NOT NULL,
    content_piece_id UUID NOT NULL,
    content_title   TEXT NOT NULL,
    workflow_name   TEXT NOT NULL,
    current_step    SMALLINT NOT NULL,
    current_step_name TEXT NOT NULL,
    total_steps     SMALLINT NOT NULL,
    status          TEXT NOT NULL,
    submitted_by_name TEXT NOT NULL,
    submitted_at    TIMESTAMPTZ NOT NULL,
    current_reviewer_ids UUID[],
    last_event_seq  BIGINT NOT NULL
);

CREATE INDEX idx_proj_approval_ws ON proj_approval_queue(workspace_id, status);

-- Projection: Engagement analytics (latest snapshot per post)
CREATE TABLE proj_engagement_latest (
    scheduled_post_id UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    content_piece_id UUID NOT NULL,
    channel_id      UUID NOT NULL,
    platform        TEXT NOT NULL,
    published_at    TIMESTAMPTZ,
    impressions     INTEGER NOT NULL DEFAULT 0,
    reach           INTEGER NOT NULL DEFAULT 0,
    likes           INTEGER NOT NULL DEFAULT 0,
    comments        INTEGER NOT NULL DEFAULT 0,
    shares          INTEGER NOT NULL DEFAULT 0,
    clicks          INTEGER NOT NULL DEFAULT 0,
    engagement_rate NUMERIC(7,6),
    last_collected  TIMESTAMPTZ NOT NULL,
    last_event_seq  BIGINT NOT NULL
);

CREATE INDEX idx_proj_engage_ws ON proj_engagement_latest(workspace_id);
CREATE INDEX idx_proj_engage_channel ON proj_engagement_latest(channel_id);

-- Projection: Workspace member directory
CREATE TABLE proj_workspace_members (
    workspace_id    UUID NOT NULL,
    user_id         UUID NOT NULL,
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL,
    joined_at       TIMESTAMPTZ NOT NULL,
    last_event_seq  BIGINT NOT NULL,
    PRIMARY KEY (workspace_id, user_id)
);

-- Projection: Channel directory
CREATE TABLE proj_channels (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    name            TEXT NOT NULL,
    platform        TEXT NOT NULL,
    peso_category   TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL,
    account_name    TEXT,
    token_expires_at TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL
);

CREATE INDEX idx_proj_channels_ws ON proj_channels(workspace_id);
```

## Projection Rebuild Infrastructure

```sql
-- Tracks the last processed event for each projection
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_seq  BIGINT NOT NULL,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- INSERT INTO projection_checkpoints VALUES
--   ('content_pieces', '...', 15234, now()),
--   ('calendar_entries', '...', 15230, now()),
--   ('approval_queue', '...', 15234, now()),
--   ('engagement_latest', '...', 15100, now());
```

---

## Example Queries

### Replay content piece history

```sql
-- Full history of a content piece (every change, in order)
SELECT
    e.event_type,
    e.actor_id,
    e.data,
    e.occurred_at
FROM events e
WHERE e.aggregate_type = 'content_piece'
  AND e.aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY e.sequence_number ASC;
```

### Point-in-time reconstruction

```sql
-- What was the state of this content piece on June 1st?
-- Application code replays events up to the target timestamp:
SELECT
    e.event_type,
    e.data,
    e.occurred_at
FROM events e
WHERE e.aggregate_type = 'content_piece'
  AND e.aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
  AND e.occurred_at <= '2026-06-01T00:00:00Z'
ORDER BY e.sequence_number ASC;
-- The application replays these events to reconstruct the state at that point
```

### Approval cycle time analytics

```sql
-- Average approval cycle time by month (from submitted to completed)
SELECT
    date_trunc('month', submitted.occurred_at) AS month,
    AVG(completed.occurred_at - submitted.occurred_at) AS avg_cycle_time,
    COUNT(*) AS total_approvals
FROM events submitted
JOIN events completed
    ON completed.aggregate_id = submitted.aggregate_id
    AND completed.event_type = 'approval.completed'
WHERE submitted.workspace_id = '...'
  AND submitted.event_type = 'approval.submitted'
GROUP BY 1
ORDER BY 1 DESC;
```

### Content velocity by status

```sql
-- Average time spent in each status (for AI to learn bottlenecks)
WITH status_transitions AS (
    SELECT
        aggregate_id,
        data->>'status' AS new_status,
        occurred_at,
        LAG(occurred_at) OVER (PARTITION BY aggregate_id ORDER BY sequence_number) AS prev_occurred_at,
        LAG(data->>'status') OVER (PARTITION BY aggregate_id ORDER BY sequence_number) AS prev_status
    FROM events
    WHERE workspace_id = '...'
      AND event_type = 'content.status_changed'
)
SELECT
    prev_status,
    new_status,
    AVG(occurred_at - prev_occurred_at) AS avg_duration,
    COUNT(*) AS transitions
FROM status_transitions
WHERE prev_status IS NOT NULL
GROUP BY prev_status, new_status
ORDER BY transitions DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (partitioned by month at scale) |
| Projections — Content | 2 | proj_content_pieces, proj_calendar_entries |
| Projections — Approval | 1 | proj_approval_queue |
| Projections — Analytics | 1 | proj_engagement_latest |
| Projections — Directory | 2 | proj_workspace_members, proj_channels |
| Infrastructure | 1 | projection_checkpoints |
| **Total** | **8** | Plus partitions; new projections can be added without schema migration |

---

## Key Design Decisions

1. **Single event store table, not per-aggregate-type tables** — Simplifies infrastructure and enables cross-aggregate queries (e.g., "all events in workspace this week"). Partitioning by month handles scale.

2. **CloudEvents 1.0 envelope** — Every event follows the CloudEvents structure (type, source, subject, time, data), making events directly publishable to external consumers via webhooks or message queues without transformation.

3. **Sequence numbers per aggregate** — The `sequence_number` column enables optimistic concurrency control: a command checks the current sequence before appending, preventing conflicting writes without pessimistic locks.

4. **Projections are disposable** — Any projection table can be dropped and rebuilt from the event store. This means new read models (e.g., a new analytics dashboard) can be created retroactively without backfilling data.

5. **Denormalised projections** — Read models like `proj_calendar_entries` include denormalised fields (channel_name, author_name) to eliminate JOINs at query time. This is the CQRS trade-off: write complexity for read simplicity.

6. **Event payloads as JSONB** — Event data varies by type (a `content.created` event has different fields than a `post.scheduled` event). JSONB in the `data` column handles this naturally. Application-level schemas validate payloads per event type.

7. **Actor type distinguishes human from system events** — The `actor_type` field (user/system/ai/webhook) enables filtering audit trails to show only human actions or only AI-generated changes, critical for compliance review.

8. **AI training-ready architecture** — The complete event history for every content piece provides the training data for approval prediction, performance estimation, and content strategy recommendation models. No separate ETL pipeline needed to extract training data.
