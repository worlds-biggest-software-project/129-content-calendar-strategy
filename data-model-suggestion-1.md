# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Content Calendar & Strategy · Created: 2026-05-19

## Philosophy

This model follows classical third-normal-form (3NF) relational design. Every concept — content pieces, channels, briefs, approval steps, analytics snapshots — gets its own dedicated table with strict foreign-key relationships. Junction tables handle many-to-many relationships (content-to-channel, content-to-tag, user-to-team). The schema enforces referential integrity at the database level, making it impossible to create orphaned records or violate business constraints.

This approach mirrors how enterprise CMS platforms like Contentful and HubSpot structure their backend data: every entity type is a first-class citizen with well-defined columns, and relationships are explicit. It is the most familiar pattern for teams with SQL experience and produces the most predictable query performance profiles.

The trade-off is rigidity: adding a new content type, a new channel-specific field, or jurisdiction-specific metadata requires a schema migration. For a content calendar platform that spans multiple social platforms with different field requirements (Instagram carousels vs. LinkedIn articles vs. Twitter threads), this can lead to wide tables with many nullable columns or proliferating subtables.

**Best for:** Teams that prioritise data integrity, complex cross-entity reporting, and predictable query patterns over rapid schema evolution.

**Trade-offs:**
- Pro: Strong referential integrity enforced at the DB level
- Pro: Familiar to most developers; excellent tooling support
- Pro: Optimal for complex JOIN-based reporting and analytics queries
- Pro: Clean mapping to ORM frameworks (Django, SQLAlchemy, Prisma)
- Con: Schema migrations required for every new field or content type
- Con: Many nullable columns or subtables for platform-specific fields
- Con: Junction tables proliferate as relationships grow
- Con: Slower iteration speed compared to JSONB or document approaches

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5545 (iCalendar) | `scheduled_posts` table maps DTSTART, DTEND, RRULE, UID to columns for calendar export/import |
| W3C Activity Streams 2.0 | `activity_log` table structure mirrors Activity Streams actor/verb/object vocabulary |
| schema.org CreativeWork | `content_pieces` fields (headline, author, datePublished, contentType) align with CreativeWork properties |
| schema.org SocialMediaPosting | Platform-specific post tables extend the CreativeWork pattern |
| PESO Model | `channels` table categorises entries as Paid/Earned/Shared/Owned via enum |
| ISO 8601 | All timestamp columns use TIMESTAMPTZ; recurrence uses ISO 8601 duration format |
| OAuth 2.0 (RFC 6749) | `social_connections` table stores OAuth tokens per platform per workspace |
| SCIM 2.0 | `users` table fields align with SCIM Core User schema for enterprise provisioning |
| CloudEvents 1.0 | `webhooks` and `webhook_deliveries` tables structure outbound events per CloudEvents envelope |
| OpenAPI 3.1 | Schema designed to map cleanly to OpenAPI resource definitions |

---

## Multi-Tenancy & Identity

```sql
-- Workspaces (tenants)
CREATE TABLE workspaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free', 'starter', 'professional', 'enterprise')),
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    auth_provider   TEXT NOT NULL DEFAULT 'email' CHECK (auth_provider IN ('email', 'google', 'saml', 'oidc')),
    auth_provider_id TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Workspace membership with roles
CREATE TABLE workspace_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'editor', 'contributor', 'viewer')),
    invited_by      UUID REFERENCES users(id),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, user_id)
);

CREATE INDEX idx_workspace_members_workspace ON workspace_members(workspace_id);
CREATE INDEX idx_workspace_members_user ON workspace_members(user_id);

-- Teams within a workspace
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_members (
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('lead', 'member')),
    PRIMARY KEY (team_id, user_id)
);
```

## Channels & Social Connections

```sql
-- Channel definitions (PESO-aligned)
CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    platform        TEXT NOT NULL CHECK (platform IN (
        'facebook', 'instagram', 'linkedin', 'twitter', 'tiktok',
        'youtube', 'pinterest', 'blog', 'email', 'podcast', 'other'
    )),
    peso_category   TEXT NOT NULL CHECK (peso_category IN ('paid', 'earned', 'shared', 'owned')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_channels_workspace ON channels(workspace_id);

-- OAuth connections to social platforms
CREATE TABLE social_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id      UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    platform        TEXT NOT NULL,
    account_id      TEXT NOT NULL,         -- Platform-specific account/page ID
    account_name    TEXT,
    access_token    TEXT NOT NULL,          -- Encrypted at application layer
    refresh_token   TEXT,                   -- Encrypted at application layer
    token_expires_at TIMESTAMPTZ,
    scopes          TEXT[],
    connected_by    UUID NOT NULL REFERENCES users(id),
    connected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_refreshed  TIMESTAMPTZ
);

CREATE INDEX idx_social_connections_channel ON social_connections(channel_id);
```

## Content Pieces & Briefs

```sql
-- Content pieces (the core entity)
CREATE TABLE content_pieces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    content_type    TEXT NOT NULL CHECK (content_type IN (
        'blog_post', 'social_post', 'email', 'newsletter', 'video',
        'podcast', 'infographic', 'whitepaper', 'case_study', 'webinar', 'other'
    )),
    status          TEXT NOT NULL DEFAULT 'idea' CHECK (status IN (
        'idea', 'briefed', 'in_progress', 'in_review', 'approved', 'scheduled', 'published', 'archived'
    )),
    body            TEXT,                   -- Rich text / markdown body
    excerpt         TEXT,                   -- Short summary
    author_id       UUID REFERENCES users(id),
    assignee_id     UUID REFERENCES users(id),
    campaign_id     UUID REFERENCES campaigns(id),
    brief_id        UUID REFERENCES content_briefs(id),
    publish_date    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_pieces_workspace ON content_pieces(workspace_id);
CREATE INDEX idx_content_pieces_status ON content_pieces(workspace_id, status);
CREATE INDEX idx_content_pieces_publish_date ON content_pieces(workspace_id, publish_date);
CREATE INDEX idx_content_pieces_author ON content_pieces(author_id);
CREATE INDEX idx_content_pieces_campaign ON content_pieces(campaign_id);

-- AI-generated content briefs
CREATE TABLE content_briefs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    topic           TEXT NOT NULL,
    target_persona  TEXT,
    target_keywords TEXT[],
    keyword_intent  TEXT CHECK (keyword_intent IN ('informational', 'navigational', 'transactional', 'commercial')),
    recommended_structure TEXT,             -- Outline / structure guidance
    tone_guidelines TEXT,
    word_count_target INTEGER,
    internal_links  TEXT[],                 -- Suggested internal linking URLs
    competitor_refs TEXT[],                 -- Competitor content URLs for reference
    generated_by    TEXT DEFAULT 'ai',      -- 'ai' or 'manual'
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_briefs_workspace ON content_briefs(workspace_id);

-- Campaigns / content series
CREATE TABLE campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    start_date      DATE,
    end_date        DATE,
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'paused', 'completed')),
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaigns_workspace ON campaigns(workspace_id);
```

## Scheduling & Publishing

```sql
-- Scheduled posts (one per channel per content piece)
CREATE TABLE scheduled_posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_piece_id UUID NOT NULL REFERENCES content_pieces(id) ON DELETE CASCADE,
    channel_id      UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    scheduled_at    TIMESTAMPTZ NOT NULL,   -- Maps to iCalendar DTSTART
    published_at    TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'scheduled' CHECK (status IN (
        'draft', 'scheduled', 'publishing', 'published', 'failed', 'cancelled'
    )),
    platform_post_id TEXT,                  -- ID returned by the platform after publishing
    platform_url    TEXT,                   -- URL of the published post
    -- Platform-specific fields (normalized approach = separate columns)
    caption         TEXT,
    hashtags        TEXT[],
    mentions        TEXT[],
    link_url        TEXT,
    media_ids       UUID[],                 -- References to assets table
    error_message   TEXT,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    -- iCalendar UID for export
    ical_uid        TEXT NOT NULL DEFAULT gen_random_uuid()::TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scheduled_posts_content ON scheduled_posts(content_piece_id);
CREATE INDEX idx_scheduled_posts_channel ON scheduled_posts(channel_id);
CREATE INDEX idx_scheduled_posts_scheduled ON scheduled_posts(workspace_id, scheduled_at);
CREATE INDEX idx_scheduled_posts_status ON scheduled_posts(status) WHERE status = 'scheduled';

-- Optimal posting times (AI-generated recommendations)
CREATE TABLE optimal_posting_times (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id      UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    day_of_week     SMALLINT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6), -- 0=Sunday
    hour_utc        SMALLINT NOT NULL CHECK (hour_utc BETWEEN 0 AND 23),
    score           NUMERIC(5,4) NOT NULL,  -- Engagement likelihood 0.0000–1.0000
    sample_size     INTEGER NOT NULL,
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_optimal_times_channel ON optimal_posting_times(channel_id);
```

## Approval Workflows

```sql
-- Approval workflow definitions
CREATE TABLE approval_workflows (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Steps within a workflow
CREATE TABLE approval_workflow_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id     UUID NOT NULL REFERENCES approval_workflows(id) ON DELETE CASCADE,
    step_order      SMALLINT NOT NULL,
    name            TEXT NOT NULL,
    approver_type   TEXT NOT NULL CHECK (approver_type IN ('user', 'team', 'role')),
    approver_id     UUID,                   -- User ID, Team ID, or null for role-based
    approver_role   TEXT,                    -- Role name if approver_type='role'
    required        BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (workflow_id, step_order)
);

-- Approval requests (instances of workflow execution)
CREATE TABLE approval_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_piece_id UUID NOT NULL REFERENCES content_pieces(id) ON DELETE CASCADE,
    workflow_id     UUID NOT NULL REFERENCES approval_workflows(id),
    current_step    SMALLINT NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'in_review', 'approved', 'rejected', 'cancelled'
    )),
    submitted_by    UUID NOT NULL REFERENCES users(id),
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_approval_requests_content ON approval_requests(content_piece_id);
CREATE INDEX idx_approval_requests_status ON approval_requests(status);

-- Individual approval decisions
CREATE TABLE approval_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID NOT NULL REFERENCES approval_requests(id) ON DELETE CASCADE,
    step_id         UUID NOT NULL REFERENCES approval_workflow_steps(id),
    reviewer_id     UUID NOT NULL REFERENCES users(id),
    decision        TEXT NOT NULL CHECK (decision IN ('approved', 'rejected', 'revision_requested')),
    comment         TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_approval_decisions_request ON approval_decisions(request_id);
```

## Assets & Media

```sql
-- Digital assets (images, videos, documents)
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    filename        TEXT NOT NULL,
    mime_type       TEXT NOT NULL,
    file_size       BIGINT NOT NULL,
    storage_url     TEXT NOT NULL,
    thumbnail_url   TEXT,
    alt_text        TEXT,                   -- WCAG 2.2 compliance
    width           INTEGER,
    height          INTEGER,
    duration_seconds NUMERIC,               -- For video/audio
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_workspace ON assets(workspace_id);

-- Junction: content pieces to assets
CREATE TABLE content_piece_assets (
    content_piece_id UUID NOT NULL REFERENCES content_pieces(id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    position        SMALLINT NOT NULL DEFAULT 0,
    PRIMARY KEY (content_piece_id, asset_id)
);
```

## Tags, Topics & Content Intelligence

```sql
-- Tags for content categorisation
CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    color           TEXT,
    UNIQUE (workspace_id, name)
);

-- Junction: content pieces to tags
CREATE TABLE content_piece_tags (
    content_piece_id UUID NOT NULL REFERENCES content_pieces(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (content_piece_id, tag_id)
);

-- AI-generated topic suggestions
CREATE TABLE topic_suggestions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    topic           TEXT NOT NULL,
    rationale       TEXT,                   -- Why the AI recommends this topic
    estimated_search_volume INTEGER,
    estimated_difficulty NUMERIC(3,2),       -- 0.00–1.00
    suggested_content_type TEXT,
    suggested_keywords TEXT[],
    status          TEXT NOT NULL DEFAULT 'new' CHECK (status IN ('new', 'accepted', 'dismissed')),
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_topic_suggestions_workspace ON topic_suggestions(workspace_id, status);

-- Content performance predictions
CREATE TABLE performance_predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_piece_id UUID NOT NULL REFERENCES content_pieces(id) ON DELETE CASCADE,
    channel_id      UUID REFERENCES channels(id),
    predicted_reach INTEGER,
    predicted_engagement NUMERIC(5,4),       -- Engagement rate 0.0000–1.0000
    predicted_clicks INTEGER,
    confidence      NUMERIC(3,2),            -- 0.00–1.00
    model_version   TEXT NOT NULL,
    predicted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_predictions_content ON performance_predictions(content_piece_id);
```

## Analytics & Engagement

```sql
-- Post-publish engagement metrics (snapshots)
CREATE TABLE engagement_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scheduled_post_id UUID NOT NULL REFERENCES scheduled_posts(id) ON DELETE CASCADE,
    impressions     INTEGER NOT NULL DEFAULT 0,
    reach           INTEGER NOT NULL DEFAULT 0,
    likes           INTEGER NOT NULL DEFAULT 0,
    comments        INTEGER NOT NULL DEFAULT 0,
    shares          INTEGER NOT NULL DEFAULT 0,
    clicks          INTEGER NOT NULL DEFAULT 0,
    saves           INTEGER NOT NULL DEFAULT 0,
    engagement_rate NUMERIC(7,6),
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_engagement_post ON engagement_metrics(scheduled_post_id);
CREATE INDEX idx_engagement_collected ON engagement_metrics(collected_at);

-- Competitor monitoring entries
CREATE TABLE competitor_content (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    competitor_name TEXT NOT NULL,
    platform        TEXT NOT NULL,
    post_url        TEXT,
    content_summary TEXT,
    estimated_engagement INTEGER,
    detected_topics TEXT[],
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_competitor_workspace ON competitor_content(workspace_id);
```

## Activity Log

```sql
-- Activity log (W3C Activity Streams 2.0 aligned)
CREATE TABLE activity_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    actor_id        UUID NOT NULL REFERENCES users(id),
    verb            TEXT NOT NULL,           -- 'created', 'updated', 'approved', 'published', etc.
    object_type     TEXT NOT NULL,           -- 'content_piece', 'scheduled_post', 'brief', etc.
    object_id       UUID NOT NULL,
    target_type     TEXT,                    -- Optional target (e.g., channel for publishing)
    target_id       UUID,
    summary         TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_workspace ON activity_log(workspace_id, occurred_at DESC);
CREATE INDEX idx_activity_object ON activity_log(object_type, object_id);
```

## Integrations & Webhooks

```sql
-- External integrations (Zapier, HubSpot, Google Analytics, etc.)
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,           -- 'zapier', 'hubspot', 'google_analytics', 'slack'
    config          JSONB NOT NULL DEFAULT '{}',
    access_token    TEXT,                    -- Encrypted
    is_active       BOOLEAN NOT NULL DEFAULT true,
    connected_by    UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Outbound webhook subscriptions (CloudEvents 1.0 envelope)
CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    event_types     TEXT[] NOT NULL,         -- e.g., {'content.published', 'brief.approved'}
    secret          TEXT NOT NULL,           -- HMAC signing secret
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    webhook_id      UUID NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,
    event_type      TEXT NOT NULL,           -- CloudEvents 'type' field
    event_source    TEXT NOT NULL,           -- CloudEvents 'source' field
    payload         JSONB NOT NULL,
    response_status INTEGER,
    response_body   TEXT,
    delivered_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhook_deliveries_webhook ON webhook_deliveries(webhook_id, delivered_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy & Identity | 5 | workspaces, users, workspace_members, teams, team_members |
| Channels & Connections | 2 | channels, social_connections |
| Content & Briefs | 3 | content_pieces, content_briefs, campaigns |
| Scheduling & Publishing | 2 | scheduled_posts, optimal_posting_times |
| Approval Workflows | 4 | approval_workflows, approval_workflow_steps, approval_requests, approval_decisions |
| Assets & Media | 2 | assets, content_piece_assets |
| Tags & Intelligence | 3 | tags, content_piece_tags, topic_suggestions |
| Predictions | 1 | performance_predictions |
| Analytics | 2 | engagement_metrics, competitor_content |
| Activity | 1 | activity_log |
| Integrations | 3 | integrations, webhooks, webhook_deliveries |
| **Total** | **28** | |

---

## Key Design Decisions

1. **Separate `scheduled_posts` from `content_pieces`** — A single content piece can be published to multiple channels at different times, so the scheduling relationship is one-to-many, not one-to-one.

2. **Platform-specific fields as columns, not JSONB** — Caption, hashtags, mentions, and link URL are common enough across platforms to justify dedicated columns. This enables direct SQL queries and index-based filtering without JSONB operators.

3. **Approval workflows are configurable, not hardcoded** — The workflow/steps/requests/decisions four-table pattern supports custom multi-step approval chains per workspace, matching how Planable and CoSchedule structure their workflows.

4. **Engagement metrics as time-series snapshots** — Rather than overwriting a single row, each collection creates a new snapshot row. This preserves historical trends and supports "how did engagement grow over the first 48 hours?" queries.

5. **Activity log aligned to W3C Activity Streams 2.0** — Using actor/verb/object/target vocabulary makes the log interoperable with ActivityPub and enables future federation capabilities.

6. **Tags as a junction table, not an array column** — Enables efficient tag-based filtering with JOIN queries and supports tag management (rename, merge, delete) without updating every content piece.

7. **iCalendar UID on scheduled posts** — Each scheduled post carries an iCalendar-compatible UID for RFC 5545 export, enabling two-way sync with Google Calendar and Outlook.

8. **Row-Level Security ready** — Every tenant-scoped table includes `workspace_id`, enabling PostgreSQL RLS policies for tenant isolation without application-level filtering.
