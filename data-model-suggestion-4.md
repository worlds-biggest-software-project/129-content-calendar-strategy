# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Content Calendar & Strategy · Created: 2026-05-19

## Philosophy

This model combines a relational core for operational CRUD (workspaces, users, channels, scheduling) with a property graph layer for modelling the complex web of relationships that a content strategy platform must navigate: content repurposing chains (blog post spawns tweet thread spawns carousel spawns email excerpt), campaign-to-content hierarchies, topic clusters and pillar pages, competitor content networks, and contributor expertise graphs.

Content marketing strategy is fundamentally a graph problem. A single idea becomes a brief, which produces a blog post, which is repurposed into 5 social posts, each published to different channels at different times, each referencing shared assets, each tagged with overlapping topics, and each attributed to the same campaign. The relationships between these entities (derives-from, references, competes-with, targets-same-audience, shares-topic-with) are as important as the entities themselves, and relational JOINs across 4-5 junction tables become unwieldy for traversal queries.

This approach uses a dedicated `graph_nodes` and `graph_edges` table pair (a property graph stored in PostgreSQL) alongside standard relational tables. The graph layer powers content lineage queries, topic clustering, content gap analysis, and AI-driven recommendations. The relational layer handles scheduling, approvals, and authentication. Apache AGE (a PostgreSQL extension for graph queries) or recursive CTEs provide graph traversal at query time.

**Best for:** Platforms where content relationships, repurposing chains, topic clustering, and strategic recommendations are primary value propositions — not just scheduling and publishing.

**Trade-offs:**
- Pro: Content lineage ("show me everything derived from this blog post") is a single graph traversal, not a recursive CTE across multiple tables
- Pro: Topic clustering and pillar-page strategy become natural graph operations
- Pro: AI recommendation engine can traverse the graph to find content gaps and opportunities
- Pro: Competitor content relationships and market positioning are first-class graph queries
- Pro: Flexible — new relationship types require only a new edge label, no schema migration
- Con: Graph query syntax (Cypher or recursive CTE) has a steeper learning curve
- Con: Graph indexes are less mature in PostgreSQL than in dedicated graph databases (Neo4j)
- Con: Two query paradigms (SQL for CRUD, graph for traversal) increases cognitive load
- Con: Property graph on PostgreSQL performs well to ~10M edges; beyond that, a dedicated graph DB may be needed
- Con: Fewer ORM frameworks support graph patterns natively

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C Activity Streams 2.0 | Graph edge types align with Activity Streams relationship vocabulary (Create, Derive, Reference) |
| schema.org CreativeWork | Graph nodes for content pieces carry schema.org properties (headline, author, datePublished) |
| schema.org isBasedOn / hasPart | Content derivation edges map directly to schema.org relationship properties |
| RFC 8288 (Web Linking) | Graph edges implement Web Linking relation types (canonical, alternate, related) |
| RFC 5545 (iCalendar) | Calendar entries in relational tables; graph edges connect scheduled posts to content lineage |
| PESO Model | Channel nodes carry PESO category; graph queries can filter by media type traversal |
| SKOS (Simple Knowledge Organization System) | Topic/keyword taxonomy modelled as graph with broader/narrower/related relationships |
| ISO 8601 | All timestamps in TIMESTAMPTZ format |
| CloudEvents 1.0 | Webhook events reference graph node IDs for relationship context |

---

## Relational Core (Operational CRUD)

```sql
-- Workspaces, users, membership (standard relational)
CREATE TABLE workspaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workspace_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, user_id)
);

CREATE INDEX idx_wm_workspace ON workspace_members(workspace_id);
CREATE INDEX idx_wm_user ON workspace_members(user_id);

-- Channels (relational for scheduling operations)
CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    platform        TEXT NOT NULL,
    peso_category   TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    connection      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_channels_workspace ON channels(workspace_id);

-- Content pieces (relational for CRUD and scheduling)
CREATE TABLE content_pieces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    content_type    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'idea',
    body            TEXT,
    author_id       UUID REFERENCES users(id),
    assignee_id     UUID REFERENCES users(id),
    publish_date    TIMESTAMPTZ,
    brief           JSONB,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cp_workspace ON content_pieces(workspace_id);
CREATE INDEX idx_cp_status ON content_pieces(workspace_id, status);
CREATE INDEX idx_cp_publish ON content_pieces(workspace_id, publish_date);

-- Scheduled posts (relational for time-based operations)
CREATE TABLE scheduled_posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    content_piece_id UUID NOT NULL REFERENCES content_pieces(id) ON DELETE CASCADE,
    channel_id      UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    scheduled_at    TIMESTAMPTZ NOT NULL,
    published_at    TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'scheduled',
    platform_content JSONB NOT NULL DEFAULT '{}',
    platform_response JSONB,
    ical_uid        TEXT NOT NULL DEFAULT gen_random_uuid()::TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sp_workspace ON scheduled_posts(workspace_id, scheduled_at);
CREATE INDEX idx_sp_content ON scheduled_posts(content_piece_id);
CREATE INDEX idx_sp_status ON scheduled_posts(status) WHERE status = 'scheduled';

-- Approval requests (relational for workflow state machine)
CREATE TABLE approval_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_piece_id UUID NOT NULL REFERENCES content_pieces(id) ON DELETE CASCADE,
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    workflow_config  JSONB NOT NULL,         -- Snapshot of workflow steps at submission time
    current_step    SMALLINT NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'pending',
    submitted_by    UUID NOT NULL REFERENCES users(id),
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    decisions       JSONB NOT NULL DEFAULT '[]'
);

CREATE INDEX idx_ar_workspace ON approval_requests(workspace_id, status);

-- Assets (relational for storage operations)
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    filename        TEXT NOT NULL,
    mime_type       TEXT NOT NULL,
    file_size       BIGINT NOT NULL,
    storage_url     TEXT NOT NULL,
    alt_text        TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_workspace ON assets(workspace_id);

-- Engagement metrics (relational time-series)
CREATE TABLE engagement_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scheduled_post_id UUID NOT NULL REFERENCES scheduled_posts(id) ON DELETE CASCADE,
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    impressions     INTEGER NOT NULL DEFAULT 0,
    reach           INTEGER NOT NULL DEFAULT 0,
    engagement_rate NUMERIC(7,6),
    platform_metrics JSONB NOT NULL DEFAULT '{}',
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_engage_post ON engagement_snapshots(scheduled_post_id);
CREATE INDEX idx_engage_ws ON engagement_snapshots(workspace_id, collected_at DESC);
```

---

## Property Graph Layer

```sql
-- Graph nodes: every entity that participates in relationships
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    node_type       TEXT NOT NULL,
    -- Node types:
    -- 'content'    — maps to content_pieces.id
    -- 'campaign'   — a campaign grouping
    -- 'topic'      — a topic/keyword in the taxonomy
    -- 'persona'    — a target audience persona
    -- 'channel'    — maps to channels.id
    -- 'asset'      — maps to assets.id
    -- 'competitor'  — a competitor entity
    -- 'competitor_content' — a tracked competitor content piece
    -- 'user'       — maps to users.id (for contributor expertise graph)
    entity_id       UUID,                    -- FK to the relational table (content_pieces.id, channels.id, etc.)
    label           TEXT NOT NULL,           -- Human-readable label
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example (content node):
    -- {
    --   "content_type": "blog_post",
    --   "status": "published",
    --   "word_count": 2400,
    --   "publish_date": "2026-06-15",
    --   "performance_score": 0.82
    -- }
    --
    -- Example (topic node):
    -- {
    --   "search_volume": 12000,
    --   "difficulty": 0.45,
    --   "intent": "informational",
    --   "pillar": true
    -- }
    --
    -- Example (persona node):
    -- {
    --   "name": "B2B SaaS Decision Maker",
    --   "seniority": "VP/Director",
    --   "industry": "Technology",
    --   "pain_points": ["cost", "integration complexity", "time to value"]
    -- }
    --
    -- Example (competitor node):
    -- {
    --   "name": "Competitor Corp",
    --   "website": "https://competitor.com",
    --   "estimated_domain_authority": 65,
    --   "content_frequency": "3x/week"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gn_workspace ON graph_nodes(workspace_id);
CREATE INDEX idx_gn_type ON graph_nodes(workspace_id, node_type);
CREATE INDEX idx_gn_entity ON graph_nodes(entity_id) WHERE entity_id IS NOT NULL;
CREATE INDEX idx_gn_properties ON graph_nodes USING GIN(properties jsonb_path_ops);
CREATE INDEX idx_gn_label ON graph_nodes(workspace_id, label);

-- Graph edges: typed, weighted, temporal relationships
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    source_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       TEXT NOT NULL,
    -- Edge types:
    --
    -- Content lineage:
    --   'derived_from'        — content B was repurposed from content A
    --   'inspired_by'         — content B was inspired by content A (looser than derived)
    --   'is_part_of'          — content B is a section/excerpt of content A (schema.org hasPart)
    --   'references'          — content A links to or cites content B
    --   'is_update_of'        — content B is a refreshed version of content A
    --
    -- Campaign/organisation:
    --   'belongs_to_campaign' — content belongs to a campaign
    --   'published_on'        — content was published on a channel
    --   'created_by'          — content was authored by a user
    --   'assigned_to'         — content is assigned to a user
    --   'uses_asset'          — content uses a media asset
    --
    -- Topic/taxonomy (SKOS-aligned):
    --   'has_topic'           — content covers a topic
    --   'broader_topic'       — topic A is a broader category of topic B (skos:broader)
    --   'narrower_topic'      — topic A is a subcategory of topic B (skos:narrower)
    --   'related_topic'       — topics are related but not hierarchical (skos:related)
    --   'targets_persona'     — content or campaign targets a persona
    --
    -- Competitive intelligence:
    --   'competes_with'       — our content competes with competitor content for same keywords
    --   'outranks'            — our content outranks competitor content
    --   'outranked_by'        — competitor content outranks ours
    --   'gap_opportunity'     — topic where competitor publishes but we don't
    --
    -- Contributor expertise:
    --   'expert_in'           — user has expertise in a topic
    --   'has_written_about'   — user has published content on a topic

    weight          NUMERIC(5,4) DEFAULT 1.0, -- Relationship strength (0.0–1.0 or count)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example (derived_from edge):
    -- {
    --   "derivation_type": "repurposed",
    --   "adaptation": "blog_to_carousel",
    --   "automated": true,
    --   "ai_model": "repurpose-v2.0"
    -- }
    --
    -- Example (competes_with edge):
    -- {
    --   "keywords": ["product launch best practices"],
    --   "our_rank": 3,
    --   "their_rank": 1,
    --   "last_checked": "2026-05-19"
    -- }
    --
    -- Example (expert_in edge):
    -- {
    --   "articles_written": 12,
    --   "avg_engagement_rate": 0.045,
    --   "last_published": "2026-05-10"
    -- }
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,             -- NULL = currently active; supports temporal edges
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_workspace ON graph_edges(workspace_id);
CREATE INDEX idx_ge_source ON graph_edges(source_id, edge_type);
CREATE INDEX idx_ge_target ON graph_edges(target_id, edge_type);
CREATE INDEX idx_ge_type ON graph_edges(workspace_id, edge_type);
CREATE INDEX idx_ge_active ON graph_edges(workspace_id, edge_type)
    WHERE valid_to IS NULL;  -- Active edges only
CREATE INDEX idx_ge_properties ON graph_edges USING GIN(properties jsonb_path_ops);
```

---

## Activity Log

```sql
CREATE TABLE activity_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    actor_id        UUID NOT NULL,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    changes         JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_ws ON activity_log(workspace_id, occurred_at DESC);
```

---

## Example Graph Queries

### Content lineage: Everything derived from a blog post

```sql
-- Recursive CTE to traverse the derivation chain
WITH RECURSIVE lineage AS (
    -- Start from the source content node
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.node_type,
        gn.properties,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.entity_id = '550e8400-...'  -- The original blog post content_pieces.id
      AND gn.node_type = 'content'

    UNION ALL

    -- Traverse derived_from edges (in reverse: find things derived FROM the source)
    SELECT
        child.id,
        child.label,
        child.node_type,
        child.properties,
        l.depth + 1,
        l.path || child.id
    FROM lineage l
    JOIN graph_edges ge ON ge.target_id = l.node_id
        AND ge.edge_type = 'derived_from'
        AND ge.valid_to IS NULL
    JOIN graph_nodes child ON child.id = ge.source_id
    WHERE child.id != ALL(l.path)  -- Prevent cycles
      AND l.depth < 5              -- Max traversal depth
)
SELECT
    depth,
    label,
    properties->>'content_type' AS content_type,
    properties->>'status' AS status
FROM lineage
ORDER BY depth, label;
```

### Topic cluster map: Pillar page with all supporting content

```sql
-- Find all content connected to a pillar topic and its subtopics
WITH RECURSIVE topic_tree AS (
    -- Start from the pillar topic
    SELECT gn.id AS topic_id, gn.label AS topic_name, 0 AS depth
    FROM graph_nodes gn
    WHERE gn.workspace_id = '...'
      AND gn.node_type = 'topic'
      AND gn.properties->>'pillar' = 'true'
      AND gn.label = 'Content Marketing'

    UNION ALL

    -- Traverse narrower_topic edges to get subtopics
    SELECT child.id, child.label, tt.depth + 1
    FROM topic_tree tt
    JOIN graph_edges ge ON ge.source_id = tt.topic_id
        AND ge.edge_type = 'narrower_topic'
        AND ge.valid_to IS NULL
    JOIN graph_nodes child ON child.id = ge.target_id
    WHERE tt.depth < 3
)
SELECT
    tt.topic_name,
    tt.depth AS topic_depth,
    cp.title AS content_title,
    cp.status,
    cp.content_type
FROM topic_tree tt
JOIN graph_edges ge ON ge.target_id = tt.topic_id
    AND ge.edge_type = 'has_topic'
    AND ge.valid_to IS NULL
JOIN graph_nodes cn ON cn.id = ge.source_id
    AND cn.node_type = 'content'
JOIN content_pieces cp ON cp.id = cn.entity_id
ORDER BY tt.depth, cp.publish_date DESC;
```

### Content gap analysis: Topics competitors cover that we do not

```sql
-- Find topics where competitors have content but we don't
SELECT
    topic.label AS topic,
    topic.properties->>'search_volume' AS search_volume,
    COUNT(DISTINCT comp_content.id) AS competitor_pieces,
    STRING_AGG(DISTINCT comp.label, ', ') AS competitors_covering
FROM graph_nodes topic
-- Topics linked to competitor content
JOIN graph_edges ce ON ce.target_id = topic.id
    AND ce.edge_type = 'has_topic'
    AND ce.valid_to IS NULL
JOIN graph_nodes comp_content ON comp_content.id = ce.source_id
    AND comp_content.node_type = 'competitor_content'
JOIN graph_edges comp_edge ON comp_edge.source_id = comp_content.id
    AND comp_edge.edge_type = 'created_by'
JOIN graph_nodes comp ON comp.id = comp_edge.target_id
    AND comp.node_type = 'competitor'
-- Exclude topics we already cover
WHERE topic.workspace_id = '...'
  AND topic.node_type = 'topic'
  AND NOT EXISTS (
      SELECT 1
      FROM graph_edges our_edge
      JOIN graph_nodes our_content ON our_content.id = our_edge.source_id
          AND our_content.node_type = 'content'
      WHERE our_edge.target_id = topic.id
        AND our_edge.edge_type = 'has_topic'
        AND our_edge.valid_to IS NULL
  )
GROUP BY topic.id, topic.label, topic.properties
ORDER BY (topic.properties->>'search_volume')::int DESC NULLS LAST;
```

### Find the best author for a topic

```sql
-- Recommend an author based on expertise graph
SELECT
    u.display_name,
    u.id AS user_id,
    ge.weight AS expertise_score,
    (ge.properties->>'articles_written')::int AS articles_written,
    (ge.properties->>'avg_engagement_rate')::numeric AS avg_engagement
FROM graph_nodes topic
JOIN graph_edges ge ON ge.target_id = topic.id
    AND ge.edge_type = 'expert_in'
    AND ge.valid_to IS NULL
JOIN graph_nodes user_node ON user_node.id = ge.source_id
    AND user_node.node_type = 'user'
JOIN users u ON u.id = user_node.entity_id
WHERE topic.workspace_id = '...'
  AND topic.label = 'Product Launch Strategy'
ORDER BY ge.weight DESC, (ge.properties->>'avg_engagement_rate')::numeric DESC
LIMIT 5;
```

### Campaign content map: All content, channels, and personas for a campaign

```sql
SELECT
    cp.title,
    cp.content_type,
    cp.status,
    ch.platform,
    persona.label AS target_persona,
    lineage_parent.label AS derived_from
FROM graph_nodes campaign_node
-- Content belonging to campaign
JOIN graph_edges camp_edge ON camp_edge.source_id = campaign_node.id
    AND camp_edge.edge_type = 'belongs_to_campaign'
JOIN graph_nodes content_node ON content_node.id = camp_edge.target_id
    AND content_node.node_type = 'content'
JOIN content_pieces cp ON cp.id = content_node.entity_id
-- Channel published on
LEFT JOIN graph_edges ch_edge ON ch_edge.source_id = content_node.id
    AND ch_edge.edge_type = 'published_on'
LEFT JOIN graph_nodes ch_node ON ch_node.id = ch_edge.target_id
LEFT JOIN channels ch ON ch.id = ch_node.entity_id
-- Target persona
LEFT JOIN graph_edges p_edge ON p_edge.source_id = content_node.id
    AND p_edge.edge_type = 'targets_persona'
LEFT JOIN graph_nodes persona ON persona.id = p_edge.target_id
-- Content lineage
LEFT JOIN graph_edges deriv_edge ON deriv_edge.source_id = content_node.id
    AND deriv_edge.edge_type = 'derived_from'
LEFT JOIN graph_nodes lineage_parent ON lineage_parent.id = deriv_edge.target_id
WHERE campaign_node.workspace_id = '...'
  AND campaign_node.label = 'Q3 Product Launch'
ORDER BY cp.publish_date;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenancy & Identity | 3 | workspaces, users, workspace_members |
| Channels | 1 | channels |
| Content & Scheduling | 2 | content_pieces, scheduled_posts |
| Approvals | 1 | approval_requests |
| Assets | 1 | assets |
| Analytics | 1 | engagement_snapshots |
| Graph Layer | 2 | graph_nodes, graph_edges (the strategy engine) |
| Activity | 1 | activity_log |
| **Total** | **12** | Relational core + 2-table graph layer |

---

## Key Design Decisions

1. **Two-table property graph, not a junction table per relationship** — A single `graph_edges` table with an `edge_type` discriminator handles all relationship types (derivation, topic mapping, competitor analysis, expertise). Adding a new relationship type is just a new edge_type value, not a new table. This is the EAV (Entity-Attribute-Value) pattern applied to relationships.

2. **Graph nodes reference relational entities via `entity_id`** — The graph layer does not replace the relational tables; it overlays them. A `content` graph node points to a `content_pieces` row via `entity_id`. This means CRUD operations use the relational tables (fast, familiar), while traversal queries use the graph layer (expressive, flexible).

3. **Temporal edges with `valid_from`/`valid_to`** — Relationships change over time: a piece of content might outrank a competitor today but not next month. Temporal edges support "as-of" queries ("What was our content gap map on March 1st?") without deleting historical relationships.

4. **SKOS-aligned topic taxonomy** — The topic hierarchy uses broader/narrower/related edge types that map directly to SKOS (Simple Knowledge Organization System), a W3C standard for taxonomies. This enables future interoperability with external taxonomy services and SEO tools.

5. **Weighted edges for AI recommendations** — The `weight` column on edges enables graph algorithms (PageRank for content importance, community detection for topic clusters, shortest path for content lineage). The AI recommendation engine can score content opportunities by traversing weighted paths through the graph.

6. **Content gap analysis as a first-class query** — By modelling competitor content as graph nodes with topic edges, finding gaps (topics they cover but we do not) becomes a straightforward graph anti-join query rather than requiring a separate competitor analysis module.

7. **Contributor expertise as graph edges** — Rather than a separate skills matrix, expertise is modelled as weighted edges between user nodes and topic nodes. The weight is updated automatically as users publish content, making author recommendations a natural graph traversal.

8. **12 tables total** — The graph layer adds only 2 tables to a minimal relational core. The graph absorbs what would otherwise be 8-10 junction tables (content-to-tag, content-to-campaign, content-to-asset, content-to-persona, topic hierarchies, competitor mappings) into a single unified relationship model.
