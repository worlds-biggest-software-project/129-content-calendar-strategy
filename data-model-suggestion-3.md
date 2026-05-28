# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Content Calendar & Strategy · Created: 2026-05-19

## Philosophy

This model uses traditional relational tables for stable, high-query entities (workspaces, users, channels, campaigns) but pushes variable, platform-specific, and rapidly evolving fields into JSONB columns. The core insight is that a content calendar platform must support many social platforms, each with different post formats, different metadata fields, and different API responses — and those requirements change frequently as platforms evolve their APIs.

Rather than creating separate subtables for Instagram carousel metadata vs. LinkedIn article metadata vs. TikTok video properties (which leads to table proliferation and constant schema migrations), this approach stores a typed `platform_data` JSONB column alongside stable relational fields. The relational fields handle what is common across all platforms (title, status, scheduled time, author); the JSONB handles what varies (carousel slide count, LinkedIn article sections, TikTok sound attribution).

This is the approach used by Contentful (structured content with flexible fields), Airtable (typed columns + formula fields), and modern headless CMS platforms. It offers the fastest path to MVP while preserving the ability to add relational structure later as patterns stabilise.

**Best for:** Rapid MVP development, multi-platform content scheduling where platform-specific fields vary widely, teams that need to iterate on the schema without database migrations for every feature.

**Trade-offs:**
- Pro: Fastest path to working product — new platform support requires no schema migration
- Pro: Platform-specific fields are self-documenting in JSONB rather than hidden in nullable columns
- Pro: GIN indexes on JSONB provide good query performance for containment queries
- Pro: Easy to add new content types, channels, or metadata fields without ALTER TABLE
- Pro: Clean separation between "universal" relational fields and "variable" JSONB extensions
- Con: JSONB fields lack database-level type enforcement (must validate at application layer)
- Con: Complex JSONB queries are less readable than simple column queries
- Con: Schema documentation must be maintained separately (not enforced by DDL)
- Con: Reporting across JSONB fields requires extraction functions, complicating analytics
- Con: Risk of "JSONB everything" creep if discipline is not maintained

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JSON Schema 2020-12 | Application-layer validation schemas for every JSONB column, documented per content type |
| RFC 5545 (iCalendar) | Calendar-relevant fields (DTSTART, RRULE) stored as relational columns for direct iCalendar export |
| schema.org CreativeWork | Relational fields align with CreativeWork; JSONB `structured_data` holds schema.org extensions |
| PESO Model | Channel `peso_category` as relational enum; PESO-specific metrics in JSONB |
| ISO 8601 | All timestamps as TIMESTAMPTZ; JSONB timestamps also follow ISO 8601 string format |
| CloudEvents 1.0 | Webhook payloads structured as CloudEvents; event metadata in JSONB |
| OAuth 2.0 | Social connection tokens in relational columns; platform-specific OAuth scopes in JSONB |
| OpenAPI 3.1 | API responses use JSON Schema for documenting JSONB field structures |
| WCAG 2.2 | Brief JSONB includes `accessibility_guidance` field for alt-text and reading-level recommendations |

---

## Multi-Tenancy & Identity

```sql
CREATE TABLE workspaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    -- Flexible workspace settings: timezone, default language, brand guidelines, etc.
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "timezone": "America/New_York",
    --   "default_language": "en-US",
    --   "brand_voice": "Professional but approachable",
    --   "brand_colors": ["#1DA1F2", "#14171A"],
    --   "content_types_enabled": ["blog_post", "social_post", "email", "video"],
    --   "approval_required": true,
    --   "default_approval_workflow_id": "a1b2c3d4-..."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example preferences:
    -- {
    --   "timezone": "Europe/London",
    --   "notification_channels": ["email", "slack"],
    --   "calendar_view": "week",
    --   "theme": "dark"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workspace_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member',
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- Example permissions (overrides role defaults):
    -- {
    --   "can_publish": true,
    --   "can_approve": false,
    --   "can_manage_channels": true,
    --   "channel_restrictions": ["linkedin", "blog"],
    --   "max_schedule_days_ahead": 90
    -- }
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, user_id)
);

CREATE INDEX idx_wm_workspace ON workspace_members(workspace_id);
CREATE INDEX idx_wm_user ON workspace_members(user_id);
```

## Channels & Social Connections

```sql
CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    platform        TEXT NOT NULL,           -- 'facebook', 'instagram', 'linkedin', etc.
    peso_category   TEXT NOT NULL,           -- 'paid', 'earned', 'shared', 'owned'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Platform-specific connection details and capabilities
    connection      JSONB NOT NULL DEFAULT '{}',
    -- Example (Facebook Page):
    -- {
    --   "account_id": "123456789",
    --   "account_name": "Acme Corp",
    --   "page_id": "987654321",
    --   "access_token": "EAABsbCS...",  (encrypted at app layer)
    --   "refresh_token": "...",
    --   "token_expires_at": "2026-08-01T00:00:00Z",
    --   "scopes": ["pages_manage_posts", "pages_read_engagement"],
    --   "capabilities": ["text", "image", "video", "carousel", "story", "reel"],
    --   "character_limits": {"text": 63206, "link_description": 500},
    --   "connected_by": "u1v2w3x4-..."
    -- }
    --
    -- Example (Email/Newsletter):
    -- {
    --   "provider": "mailchimp",
    --   "list_id": "abc123",
    --   "api_key": "...",  (encrypted at app layer)
    --   "subscriber_count": 15000,
    --   "capabilities": ["html_email", "plain_text", "a_b_test"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_channels_workspace ON channels(workspace_id);
CREATE INDEX idx_channels_platform ON channels(workspace_id, platform);
```

## Content Pieces (Universal + Flexible)

```sql
CREATE TABLE content_pieces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    -- Universal relational fields (stable, frequently queried)
    title           TEXT NOT NULL,
    content_type    TEXT NOT NULL,            -- 'blog_post', 'social_post', 'email', etc.
    status          TEXT NOT NULL DEFAULT 'idea',
    body            TEXT,
    author_id       UUID REFERENCES users(id),
    assignee_id     UUID REFERENCES users(id),
    campaign_id     UUID REFERENCES campaigns(id),
    publish_date    TIMESTAMPTZ,
    -- Flexible content metadata (varies by content type)
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example (blog_post):
    -- {
    --   "seo_title": "Q3 Product Launch: Everything You Need to Know",
    --   "meta_description": "Discover the new features...",
    --   "slug": "q3-product-launch",
    --   "reading_time_minutes": 8,
    --   "word_count": 2400,
    --   "internal_links": ["/blog/q2-recap", "/features/analytics"],
    --   "schema_org": {
    --     "@type": "BlogPosting",
    --     "keywords": ["product launch", "saas"]
    --   }
    -- }
    --
    -- Example (social_post):
    -- {
    --   "tone": "casual",
    --   "cta_type": "link_click",
    --   "source_content_id": "original-blog-uuid",
    --   "repurposed_from": "blog_post"
    -- }
    --
    -- Example (email):
    -- {
    --   "subject_line": "Big news from Acme",
    --   "preview_text": "Our Q3 launch is here...",
    --   "segment_id": "active-subscribers",
    --   "send_type": "broadcast",
    --   "consent_required": true
    -- }

    -- AI brief data (embedded, not a separate table)
    brief           JSONB,
    -- Example brief:
    -- {
    --   "topic": "Q3 Product Launch",
    --   "target_persona": "B2B SaaS Decision Maker",
    --   "target_keywords": ["product launch", "enterprise features"],
    --   "keyword_intent": "commercial",
    --   "recommended_structure": "Problem → Solution → Features → CTA",
    --   "tone_guidelines": "Professional but approachable",
    --   "word_count_target": 2500,
    --   "internal_linking_map": [
    --     {"url": "/blog/q2-recap", "anchor": "previous quarter"},
    --     {"url": "/pricing", "anchor": "see pricing"}
    --   ],
    --   "competitor_refs": ["https://competitor.com/similar-post"],
    --   "accessibility_guidance": {
    --     "reading_level": "Grade 8",
    --     "alt_text_required": true,
    --     "heading_structure": "H2 > H3, max 3 levels"
    --   },
    --   "generated_by": "ai",
    --   "model_version": "brief-gen-v2.1",
    --   "generated_at": "2026-05-19T10:00:00Z"
    -- }

    -- Tags as an array (avoids junction table for simple tagging)
    tags            TEXT[] NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cp_workspace ON content_pieces(workspace_id);
CREATE INDEX idx_cp_status ON content_pieces(workspace_id, status);
CREATE INDEX idx_cp_publish ON content_pieces(workspace_id, publish_date);
CREATE INDEX idx_cp_type ON content_pieces(workspace_id, content_type);
CREATE INDEX idx_cp_campaign ON content_pieces(campaign_id);
CREATE INDEX idx_cp_tags ON content_pieces USING GIN(tags);
CREATE INDEX idx_cp_metadata ON content_pieces USING GIN(metadata jsonb_path_ops);
```

## Scheduled Posts (Platform-Adaptive)

```sql
CREATE TABLE scheduled_posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    content_piece_id UUID NOT NULL REFERENCES content_pieces(id) ON DELETE CASCADE,
    channel_id      UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    -- Universal scheduling fields (relational)
    scheduled_at    TIMESTAMPTZ NOT NULL,
    published_at    TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'scheduled',
    -- iCalendar UID for calendar export
    ical_uid        TEXT NOT NULL DEFAULT gen_random_uuid()::TEXT,

    -- Platform-adapted content (the key JSONB column)
    platform_content JSONB NOT NULL DEFAULT '{}',
    -- Example (Instagram carousel):
    -- {
    --   "caption": "Exciting news! Our Q3 launch brings...",
    --   "hashtags": ["#productlaunch", "#saas", "#b2b"],
    --   "mentions": ["@partnercompany"],
    --   "post_type": "carousel",
    --   "slides": [
    --     {"media_id": "m1-uuid", "alt_text": "Product dashboard screenshot"},
    --     {"media_id": "m2-uuid", "alt_text": "New analytics feature"},
    --     {"media_id": "m3-uuid", "alt_text": "Pricing comparison"}
    --   ],
    --   "first_comment": "Link in bio for full details!",
    --   "location_tag": "San Francisco, CA"
    -- }
    --
    -- Example (LinkedIn article):
    -- {
    --   "headline": "Why Q3 Changes Everything for Enterprise SaaS",
    --   "body_html": "<article>...</article>",
    --   "visibility": "PUBLIC",
    --   "hashtags": ["#thought_leadership"],
    --   "post_type": "article"
    -- }
    --
    -- Example (Twitter/X thread):
    -- {
    --   "post_type": "thread",
    --   "tweets": [
    --     {"text": "1/5 Big news from Acme...", "media_ids": []},
    --     {"text": "2/5 The problem we set out to solve...", "media_ids": []},
    --     {"text": "3/5 What makes this different...", "media_ids": ["m1-uuid"]},
    --     {"text": "4/5 Early results show...", "media_ids": []},
    --     {"text": "5/5 Try it free at acme.com", "media_ids": []}
    --   ]
    -- }
    --
    -- Example (Email newsletter):
    -- {
    --   "subject_line": "Q3 Launch: What You Need to Know",
    --   "preview_text": "New features, better pricing...",
    --   "body_html": "<html>...</html>",
    --   "segment_id": "active-subscribers",
    --   "send_type": "broadcast",
    --   "reply_to": "team@acme.com"
    -- }

    -- Platform response after publishing
    platform_response JSONB,
    -- {
    --   "post_id": "123456789_987654321",
    --   "url": "https://facebook.com/acmecorp/posts/987654321",
    --   "created_time": "2026-06-15T14:00:05Z"
    -- }

    error_details   JSONB,                   -- Error info if publishing failed
    retry_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sp_workspace ON scheduled_posts(workspace_id, scheduled_at);
CREATE INDEX idx_sp_content ON scheduled_posts(content_piece_id);
CREATE INDEX idx_sp_channel ON scheduled_posts(channel_id);
CREATE INDEX idx_sp_status ON scheduled_posts(status) WHERE status IN ('scheduled', 'publishing');
CREATE INDEX idx_sp_platform_type ON scheduled_posts USING GIN((platform_content->'post_type'));
```

## Campaigns

```sql
CREATE TABLE campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    start_date      DATE,
    end_date        DATE,
    status          TEXT NOT NULL DEFAULT 'draft',
    -- Campaign-specific settings and goals
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "goal": "product_launch",
    --   "target_metrics": {
    --     "total_reach": 500000,
    --     "engagement_rate": 0.035,
    --     "link_clicks": 10000
    --   },
    --   "channels": ["linkedin", "twitter", "blog", "email"],
    --   "personas": ["b2b_decision_maker", "developer"],
    --   "budget": {"total": 5000, "currency": "USD", "per_channel": {"linkedin": 2000, "twitter": 1500}}
    -- }
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaigns_workspace ON campaigns(workspace_id);
```

## Approval Workflows

```sql
CREATE TABLE approval_workflows (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    -- Workflow steps defined as JSONB array (avoids separate steps table)
    steps           JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"order": 1, "name": "Content Review", "approver_type": "role", "approver_role": "editor", "required": true},
    --   {"order": 2, "name": "Legal Review", "approver_type": "user", "approver_id": "u1v2w3-...", "required": true},
    --   {"order": 3, "name": "Final Sign-off", "approver_type": "team", "team_id": "t4u5v6-...", "required": false}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_piece_id UUID NOT NULL REFERENCES content_pieces(id) ON DELETE CASCADE,
    workflow_id     UUID NOT NULL REFERENCES approval_workflows(id),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    current_step    SMALLINT NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'pending',
    submitted_by    UUID NOT NULL REFERENCES users(id),
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    -- Full decision history as JSONB array (replaces separate decisions table)
    decisions       JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "step_order": 1,
    --     "step_name": "Content Review",
    --     "reviewer_id": "r1s2t3-...",
    --     "reviewer_name": "Jane Editor",
    --     "decision": "approved",
    --     "comment": "Looks good. Minor typo on paragraph 3 — fixed inline.",
    --     "decided_at": "2026-05-19T14:30:00Z"
    --   },
    --   {
    --     "step_order": 2,
    --     "step_name": "Legal Review",
    --     "reviewer_id": "l4m5n6-...",
    --     "reviewer_name": "Bob Legal",
    --     "decision": "revision_requested",
    --     "comment": "Need to add disclaimer for pricing claims.",
    --     "decided_at": "2026-05-19T16:00:00Z"
    --   }
    -- ]
    -- AI prediction for likely revision requests
    ai_prediction   JSONB
    -- {
    --   "predicted_outcome": "revision_requested",
    --   "confidence": 0.73,
    --   "flagged_issues": ["Pricing claim without disclaimer", "Missing competitor comparison caveat"],
    --   "model_version": "approval-pred-v1.2",
    --   "predicted_at": "2026-05-19T12:00:00Z"
    -- }
);

CREATE INDEX idx_ar_content ON approval_requests(content_piece_id);
CREATE INDEX idx_ar_workspace ON approval_requests(workspace_id, status);
```

## Assets

```sql
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    filename        TEXT NOT NULL,
    mime_type       TEXT NOT NULL,
    file_size       BIGINT NOT NULL,
    storage_url     TEXT NOT NULL,
    alt_text        TEXT,
    -- Rich metadata varies by asset type
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example (image):
    -- {
    --   "width": 1200, "height": 630,
    --   "format": "png", "color_space": "sRGB",
    --   "thumbnail_url": "https://cdn.example.com/thumb/...",
    --   "ai_description": "Product dashboard showing analytics graphs",
    --   "tags": ["product", "screenshot", "dashboard"]
    -- }
    --
    -- Example (video):
    -- {
    --   "width": 1920, "height": 1080,
    --   "duration_seconds": 45.2,
    --   "codec": "h264",
    --   "thumbnail_url": "https://cdn.example.com/thumb/...",
    --   "transcript": "Welcome to our Q3 product launch...",
    --   "captions_url": "https://cdn.example.com/captions/..."
    -- }
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_workspace ON assets(workspace_id);
CREATE INDEX idx_assets_mime ON assets(workspace_id, mime_type);
CREATE INDEX idx_assets_metadata ON assets USING GIN(metadata jsonb_path_ops);
```

## Analytics & Engagement

```sql
CREATE TABLE engagement_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scheduled_post_id UUID NOT NULL REFERENCES scheduled_posts(id) ON DELETE CASCADE,
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    -- Universal metrics (relational)
    impressions     INTEGER NOT NULL DEFAULT 0,
    reach           INTEGER NOT NULL DEFAULT 0,
    engagement_rate NUMERIC(7,6),
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Platform-specific metrics (varies by platform)
    platform_metrics JSONB NOT NULL DEFAULT '{}',
    -- Example (Instagram):
    -- {
    --   "likes": 342, "comments": 28, "shares": 15, "saves": 89,
    --   "profile_visits": 120, "story_replies": 0,
    --   "carousel_swipe_rate": 0.67
    -- }
    --
    -- Example (LinkedIn):
    -- {
    --   "likes": 156, "comments": 12, "shares": 45, "clicks": 234,
    --   "follower_change": 18,
    --   "demographics": {
    --     "top_industries": ["Technology", "Financial Services"],
    --     "top_seniority": ["Senior", "Manager"]
    --   }
    -- }
    --
    -- Example (Email):
    -- {
    --   "delivered": 14500, "opens": 4350, "unique_opens": 3900,
    --   "clicks": 890, "unique_clicks": 720,
    --   "bounces": 45, "unsubscribes": 8,
    --   "open_rate": 0.269, "click_rate": 0.050
    -- }
);

CREATE INDEX idx_engage_post ON engagement_snapshots(scheduled_post_id);
CREATE INDEX idx_engage_workspace ON engagement_snapshots(workspace_id, collected_at DESC);

-- AI-generated performance predictions
CREATE TABLE performance_predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_piece_id UUID NOT NULL REFERENCES content_pieces(id) ON DELETE CASCADE,
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    predictions     JSONB NOT NULL,
    -- {
    --   "overall": {
    --     "estimated_total_reach": 25000,
    --     "estimated_engagement_rate": 0.032,
    --     "confidence": 0.78
    --   },
    --   "by_channel": {
    --     "linkedin": {"reach": 12000, "engagement_rate": 0.041, "best_time": "2026-06-15T14:00:00Z"},
    --     "twitter": {"reach": 8000, "engagement_rate": 0.022, "best_time": "2026-06-15T16:30:00Z"},
    --     "instagram": {"reach": 5000, "engagement_rate": 0.038, "best_time": "2026-06-15T18:00:00Z"}
    --   },
    --   "model_version": "perf-pred-v3.0",
    --   "factors": ["topic_trending", "optimal_time", "historical_audience_match"]
    -- }
    predicted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pred_content ON performance_predictions(content_piece_id);
```

## Activity & Audit Log

```sql
CREATE TABLE activity_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    actor_id        UUID NOT NULL,
    actor_type      TEXT NOT NULL DEFAULT 'user',
    action          TEXT NOT NULL,           -- 'created', 'updated', 'approved', 'published'
    resource_type   TEXT NOT NULL,           -- 'content_piece', 'scheduled_post', etc.
    resource_id     UUID NOT NULL,
    -- Changes captured as JSONB diff
    changes         JSONB,
    -- Example:
    -- {
    --   "status": {"from": "in_review", "to": "approved"},
    --   "title": {"from": "Draft Title", "to": "Final Title"}
    -- }
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_ws ON activity_log(workspace_id, occurred_at DESC);
CREATE INDEX idx_activity_resource ON activity_log(resource_type, resource_id);
```

## Integrations

```sql
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- All provider-specific config in JSONB (avoids provider-specific columns)
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example (HubSpot):
    -- {
    --   "access_token": "...", "refresh_token": "...",
    --   "portal_id": "12345",
    --   "sync_contacts": true, "sync_campaigns": true,
    --   "field_mapping": {"content_piece.title": "blog_post.name"}
    -- }
    --
    -- Example (Slack):
    -- {
    --   "webhook_url": "https://hooks.slack.com/...",
    --   "channel": "#content-team",
    --   "notify_on": ["content.approved", "post.published", "post.failed"]
    -- }
    connected_by    UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_integrations_ws ON integrations(workspace_id);
```

---

## Example Queries

### Find all Instagram carousels scheduled this week

```sql
SELECT
    sp.id,
    cp.title,
    sp.scheduled_at,
    sp.platform_content->>'caption' AS caption,
    jsonb_array_length(sp.platform_content->'slides') AS slide_count
FROM scheduled_posts sp
JOIN content_pieces cp ON cp.id = sp.content_piece_id
JOIN channels ch ON ch.id = sp.channel_id
WHERE sp.workspace_id = '...'
  AND ch.platform = 'instagram'
  AND sp.platform_content->>'post_type' = 'carousel'
  AND sp.scheduled_at >= date_trunc('week', now())
  AND sp.scheduled_at < date_trunc('week', now()) + INTERVAL '7 days';
```

### Cross-platform engagement comparison

```sql
SELECT
    ch.platform,
    AVG((es.platform_metrics->>'likes')::int) AS avg_likes,
    AVG((es.platform_metrics->>'comments')::int) AS avg_comments,
    AVG(es.engagement_rate) AS avg_engagement_rate,
    COUNT(DISTINCT sp.id) AS total_posts
FROM engagement_snapshots es
JOIN scheduled_posts sp ON sp.id = es.scheduled_post_id
JOIN channels ch ON ch.id = sp.channel_id
WHERE es.workspace_id = '...'
  AND es.collected_at >= now() - INTERVAL '30 days'
GROUP BY ch.platform
ORDER BY avg_engagement_rate DESC;
```

### Content pieces with specific brief keywords

```sql
SELECT id, title, brief->>'topic' AS topic, brief->'target_keywords' AS keywords
FROM content_pieces
WHERE workspace_id = '...'
  AND brief->'target_keywords' @> '["product launch"]'::jsonb;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy & Identity | 3 | workspaces, users, workspace_members |
| Channels | 1 | channels (connection details in JSONB) |
| Content & Briefs | 1 | content_pieces (brief embedded as JSONB) |
| Campaigns | 1 | campaigns |
| Scheduling | 1 | scheduled_posts (platform content in JSONB) |
| Approval | 2 | approval_workflows (steps in JSONB), approval_requests (decisions in JSONB) |
| Assets | 1 | assets (metadata in JSONB) |
| Analytics | 2 | engagement_snapshots, performance_predictions |
| Activity | 1 | activity_log |
| Integrations | 1 | integrations |
| **Total** | **14** | Half the table count of the normalized model |

---

## Key Design Decisions

1. **Briefs embedded in content_pieces, not a separate table** — A brief is always 1:1 with a content piece and is read whenever the content piece is read. Embedding as JSONB eliminates a JOIN and makes the content piece self-contained. If briefs later need independent querying at scale, a materialised view can be created.

2. **Approval decisions as JSONB array, not a separate table** — The complete decision history for an approval request is always read together (never paginated independently). Storing as a JSONB array in the request row eliminates a JOIN and keeps the approval timeline atomic.

3. **Platform content as JSONB on scheduled_posts** — This is the highest-value JSONB usage in the model. Instagram carousels have slides, Twitter has threads, LinkedIn has article bodies, email has subject lines and segments. Rather than a union of nullable columns or a polymorphic subtable hierarchy, JSONB captures exactly what each platform needs. JSON Schema validation at the application layer ensures correctness per platform.

4. **Tags as TEXT[] array, not a junction table** — For simple tagging (no tag metadata, no tag-level permissions), a text array with GIN index provides faster reads and simpler writes than a three-table junction pattern. Trade-off: renaming a tag requires updating all content pieces.

5. **Workspace settings as JSONB** — Workspace configuration (timezone, brand voice, enabled content types, default workflows) evolves frequently. JSONB avoids migrations for every new setting. The application layer defines the schema and defaults.

6. **Engagement metrics split: universal relational + platform JSONB** — Impressions, reach, and engagement rate are universal and relational (for cross-platform comparison queries). Platform-specific metrics (carousel swipe rate, email open rate, LinkedIn demographics) go in JSONB because they vary dramatically per platform.

7. **Activity log with JSONB diff** — The `changes` column captures before/after values for auditing. This provides 80% of the audit value of full event sourcing at 20% of the implementation complexity.

8. **14 tables vs. 28 in the normalized model** — The JSONB approach halves the table count by collapsing junction tables, steps tables, and platform-specific subtables into embedded JSONB. This simplifies migrations, reduces JOIN complexity, and accelerates development velocity.
