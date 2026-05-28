# Content Calendar & Strategy — Phased Development Plan

> Project: 129-content-calendar-strategy · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript | Full-stack unified language — API, frontend, and shared types. Strong ecosystem for web dashboards, real-time calendar UIs, and social platform SDK wrappers. Prisma ORM provides type-safe database access. |
| API framework | NestJS | Provides modular architecture, built-in OpenAPI 3.1 generation via Swagger plugin, dependency injection for testability, and first-class support for WebSockets (real-time calendar updates). Guards and interceptors support OAuth and RBAC. |
| Database | PostgreSQL 16 | Required by the Hybrid Relational + JSONB data model (Suggestion 3) — relational integrity for core entities, JSONB for platform-specific content fields, GIN indexes for tag/metadata search, TIMESTAMPTZ for cross-timezone scheduling, and RLS-ready workspace_id on every table. |
| ORM | Prisma 6 | Type-safe schema, auto-generated migrations, JSONB support, and TypeScript integration. Generates typed client from schema. |
| Task queue | BullMQ (Redis-backed) | Async workloads include social publishing, engagement metric collection, AI brief generation, and webhook delivery. BullMQ provides delayed jobs (for scheduled publishing), retries, rate limiting per social platform, and dashboard visibility. |
| Cache / sessions | Redis 7 | Shared with BullMQ. Caches optimal posting times, session tokens, and rate-limit counters for social platform APIs. |
| Frontend | Next.js 15 (App Router) | Server components for SEO on public pages, client components for interactive calendar drag-and-drop. Shares TypeScript types with the API. shadcn/ui for consistent component library. |
| Calendar UI | @dnd-kit + custom | Drag-and-drop calendar grid using @dnd-kit. No mature open-source content calendar component exists that handles multi-channel scheduling. Custom week/month/day views built on a shared CalendarEntry type. |
| AI / LLM | OpenAI SDK + Anthropic SDK | Brief generation, content repurposing, and performance prediction require LLM calls. Abstract behind a provider interface to support model swapping. Prompt templates stored as versioned files. |
| Authentication | NextAuth.js v5 | OAuth 2.0 + OpenID Connect for Google, SAML for enterprise SSO. JWT sessions with workspace-scoped claims. Supports the SCIM 2.0 provisioning endpoint pattern. |
| Social platform SDKs | Platform-specific REST wrappers | Meta Graph API, LinkedIn Marketing API, X API v2, TikTok Content Publishing API. Each wrapped in an adapter implementing a common `SocialPublisher` interface for swap-out per standards.md guidance. |
| File storage | S3-compatible (MinIO for self-hosted) | Assets (images, videos, documents) stored in S3. Pre-signed URLs for upload/download. Alt-text metadata stored in DB per WCAG 2.2. |
| Containerisation | Docker + Docker Compose | Multi-service compose: API, worker, frontend, PostgreSQL, Redis, MinIO. Single `docker compose up` for local development. |
| Testing | Vitest + Playwright | Vitest for unit and integration tests (fast, TypeScript-native). Playwright for E2E browser tests of calendar interactions and approval workflows. |
| Linting / formatting | ESLint + Prettier + strict tsconfig | Enforce consistent code style. `strict: true` and `noUncheckedIndexedAccess: true` in tsconfig. |
| API documentation | OpenAPI 3.1 (auto-generated) | NestJS Swagger plugin generates OpenAPI spec from decorators. Spec published at `/api/docs`. |
| Event bus | CloudEvents 1.0 envelope over BullMQ | Internal events (content.published, brief.approved) follow CloudEvents structure for consistency with webhook delivery. |
| iCalendar export | ical.js | RFC 5545 iCalendar generation for calendar sync with Google Calendar and Outlook. |
| Package manager | pnpm | Workspace-aware for monorepo (API + frontend + shared types). Faster and more disk-efficient than npm. |

### Project Structure

```
content-calendar-strategy/
├── pnpm-workspace.yaml
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.web
├── .env.example
├── packages/
│   └── shared/                        # Shared TypeScript types and constants
│       ├── src/
│       │   ├── types/
│       │   │   ├── content.ts          # ContentPiece, ContentBrief, ContentType enums
│       │   │   ├── calendar.ts         # CalendarEntry, ScheduledPost types
│       │   │   ├── channel.ts          # Channel, SocialConnection, PesoCategory
│       │   │   ├── approval.ts         # ApprovalWorkflow, ApprovalRequest, Decision
│       │   │   ├── campaign.ts         # Campaign types
│       │   │   ├── analytics.ts        # EngagementSnapshot, PerformancePrediction
│       │   │   ├── asset.ts            # Asset, AssetMetadata types
│       │   │   ├── workspace.ts        # Workspace, WorkspaceMember, Role enums
│       │   │   ├── user.ts             # User, UserPreferences
│       │   │   └── events.ts           # CloudEvents envelope, event type catalogue
│       │   ├── constants/
│       │   │   ├── platforms.ts         # Platform enum, character limits, capabilities
│       │   │   └── peso.ts             # PESO model categories
│       │   └── schemas/
│       │       ├── brief.schema.json   # JSON Schema for content briefs
│       │       └── platform-content/   # Per-platform JSON Schemas
│       │           ├── instagram.schema.json
│       │           ├── linkedin.schema.json
│       │           ├── twitter.schema.json
│       │           └── facebook.schema.json
│       ├── package.json
│       └── tsconfig.json
├── apps/
│   ├── api/                            # NestJS API server
│   │   ├── src/
│   │   │   ├── main.ts
│   │   │   ├── app.module.ts
│   │   │   ├── common/
│   │   │   │   ├── guards/             # Auth, RBAC, workspace-scope guards
│   │   │   │   ├── interceptors/       # Logging, error formatting
│   │   │   │   ├── decorators/         # @CurrentUser, @WorkspaceId
│   │   │   │   ├── filters/            # Exception filters
│   │   │   │   └── pipes/              # Validation pipes
│   │   │   ├── modules/
│   │   │   │   ├── auth/               # NextAuth integration, OAuth, SAML
│   │   │   │   ├── workspace/          # Workspace CRUD, member management
│   │   │   │   ├── content/            # Content pieces, briefs, tags
│   │   │   │   ├── calendar/           # Scheduling, calendar views, iCal export
│   │   │   │   ├── channel/            # Channel management, social connections
│   │   │   │   ├── campaign/           # Campaign CRUD
│   │   │   │   ├── approval/           # Approval workflows, requests, decisions
│   │   │   │   ├── asset/              # File upload, asset management
│   │   │   │   ├── analytics/          # Engagement metrics, predictions
│   │   │   │   ├── ai/                 # Brief generation, repurposing, predictions
│   │   │   │   ├── integration/        # External integrations (Zapier, HubSpot, Slack)
│   │   │   │   └── webhook/            # Outbound webhooks (CloudEvents)
│   │   │   ├── workers/
│   │   │   │   ├── publish.worker.ts   # Social platform publishing
│   │   │   │   ├── metrics.worker.ts   # Engagement metric collection
│   │   │   │   ├── ai.worker.ts        # AI brief/prediction generation
│   │   │   │   └── webhook.worker.ts   # Outbound webhook delivery
│   │   │   └── prisma/
│   │   │       ├── schema.prisma
│   │   │       └── migrations/
│   │   ├── test/
│   │   │   ├── unit/
│   │   │   ├── integration/
│   │   │   └── fixtures/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── web/                            # Next.js frontend
│       ├── src/
│       │   ├── app/
│       │   │   ├── (auth)/             # Login, signup, SSO pages
│       │   │   ├── (dashboard)/        # Authenticated workspace pages
│       │   │   │   ├── calendar/       # Calendar views (week, month, day)
│       │   │   │   ├── content/        # Content list, editor, brief view
│       │   │   │   ├── campaigns/      # Campaign management
│       │   │   │   ├── approvals/      # Approval queue and review
│       │   │   │   ├── analytics/      # Engagement dashboards
│       │   │   │   ├── channels/       # Channel management and connections
│       │   │   │   ├── assets/         # Asset library
│       │   │   │   └── settings/       # Workspace and user settings
│       │   │   └── api/                # Next.js API routes (NextAuth)
│       │   ├── components/
│       │   │   ├── ui/                 # shadcn/ui components
│       │   │   ├── calendar/           # Calendar grid, day cell, entry card
│       │   │   ├── content/            # Content editor, brief panel
│       │   │   ├── approval/           # Approval timeline, decision buttons
│       │   │   └── analytics/          # Charts, metric cards
│       │   ├── hooks/                  # Custom React hooks
│       │   ├── lib/                    # API client, utilities
│       │   └── styles/
│       ├── e2e/                        # Playwright E2E tests
│       ├── package.json
│       └── tsconfig.json
├── prompts/                            # Versioned LLM prompt templates
│   ├── brief-generation.md
│   ├── content-repurpose.md
│   ├── performance-prediction.md
│   └── approval-prediction.md
└── scripts/
    ├── seed.ts                         # Database seed for development
    └── migrate.ts                      # Migration runner
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the monorepo, database schema, authentication, and workspace multi-tenancy. After this phase, a developer can register, create a workspace, invite members, and authenticate via API. This is the bedrock every subsequent phase builds on.

### Tasks

#### 1.1 — Monorepo Setup & Configuration

**What**: Initialise the pnpm workspace with shared types package, API app, and web app. Configure TypeScript, ESLint, Prettier, Docker Compose, and CI scaffolding.

**Design**:

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
  - "apps/*"
```

```jsonc
// packages/shared/tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src"
  }
}
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: content_calendar
      POSTGRES_USER: cc_user
      POSTGRES_PASSWORD: cc_dev_password
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minio_dev
      MINIO_ROOT_PASSWORD: minio_dev_password
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]

volumes:
  pgdata:
  miniodata:
```

```dotenv
# .env.example
DATABASE_URL=postgresql://cc_user:cc_dev_password@localhost:5432/content_calendar
REDIS_URL=redis://localhost:6379
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minio_dev
S3_SECRET_KEY=minio_dev_password
S3_BUCKET=content-calendar-assets
NEXTAUTH_SECRET=dev-secret-change-in-production
NEXTAUTH_URL=http://localhost:3000
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
```

**Testing**:
- `Unit: pnpm install completes without errors`
- `Unit: pnpm build in packages/shared produces dist/ with .d.ts files`
- `Unit: TypeScript strict mode catches type errors across all packages`
- `Integration: docker compose up starts postgres, redis, minio successfully`
- `Integration: apps/api can import from @content-calendar/shared`
- `Integration: apps/web can import from @content-calendar/shared`

#### 1.2 — Database Schema & Prisma Setup

**What**: Implement the Hybrid Relational + JSONB data model (based on Data Model Suggestion 3) in Prisma schema with initial migration.

**Design**:

```prisma
// apps/api/src/prisma/schema.prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [pgcrypto]
}

model Workspace {
  id        String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  name      String
  slug      String   @unique
  plan      String   @default("free")
  settings  Json     @default("{}")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  members           WorkspaceMember[]
  channels          Channel[]
  contentPieces     ContentPiece[]
  campaigns         Campaign[]
  approvalWorkflows ApprovalWorkflow[]
  assets            Asset[]
  integrations      Integration[]
  activityLog       ActivityLog[]

  @@map("workspaces")
}

model User {
  id             String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  email          String   @unique
  displayName    String   @map("display_name")
  avatarUrl      String?  @map("avatar_url")
  preferences    Json     @default("{}")
  createdAt      DateTime @default(now()) @map("created_at")
  updatedAt      DateTime @updatedAt @map("updated_at")

  memberships    WorkspaceMember[]
  contentAuthored ContentPiece[]  @relation("author")
  contentAssigned ContentPiece[]  @relation("assignee")
  assetsUploaded  Asset[]
  activityLog     ActivityLog[]

  @@map("users")
}

model WorkspaceMember {
  id          String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  workspaceId String   @map("workspace_id") @db.Uuid
  userId      String   @map("user_id") @db.Uuid
  role        String   @default("member")
  permissions Json     @default("{}")
  joinedAt    DateTime @default(now()) @map("joined_at")

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([workspaceId, userId])
  @@index([workspaceId])
  @@index([userId])
  @@map("workspace_members")
}

model Channel {
  id           String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  workspaceId  String   @map("workspace_id") @db.Uuid
  name         String
  platform     String
  pesoCategory String   @map("peso_category")
  isActive     Boolean  @default(true) @map("is_active")
  connection   Json     @default("{}")
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")

  workspace      Workspace       @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  scheduledPosts ScheduledPost[]

  @@index([workspaceId])
  @@index([workspaceId, platform])
  @@map("channels")
}

model ContentPiece {
  id          String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  workspaceId String    @map("workspace_id") @db.Uuid
  title       String
  contentType String    @map("content_type")
  status      String    @default("idea")
  body        String?
  authorId    String?   @map("author_id") @db.Uuid
  assigneeId  String?   @map("assignee_id") @db.Uuid
  campaignId  String?   @map("campaign_id") @db.Uuid
  publishDate DateTime? @map("publish_date")
  metadata    Json      @default("{}")
  brief       Json?
  tags        String[]  @default([])
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")

  workspace        Workspace          @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  author           User?              @relation("author", fields: [authorId], references: [id])
  assignee         User?              @relation("assignee", fields: [assigneeId], references: [id])
  campaign         Campaign?          @relation(fields: [campaignId], references: [id])
  scheduledPosts   ScheduledPost[]
  approvalRequests ApprovalRequest[]
  predictions      PerformancePrediction[]

  @@index([workspaceId])
  @@index([workspaceId, status])
  @@index([workspaceId, publishDate])
  @@index([workspaceId, contentType])
  @@index([campaignId])
  @@map("content_pieces")
}

model ScheduledPost {
  id              String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  workspaceId     String    @map("workspace_id") @db.Uuid
  contentPieceId  String    @map("content_piece_id") @db.Uuid
  channelId       String    @map("channel_id") @db.Uuid
  scheduledAt     DateTime  @map("scheduled_at")
  publishedAt     DateTime? @map("published_at")
  status          String    @default("scheduled")
  icalUid         String    @default(dbgenerated("gen_random_uuid()::TEXT")) @map("ical_uid")
  platformContent Json      @default("{}") @map("platform_content")
  platformResponse Json?    @map("platform_response")
  errorDetails    Json?     @map("error_details")
  retryCount      Int       @default(0) @map("retry_count")
  createdAt       DateTime  @default(now()) @map("created_at")
  updatedAt       DateTime  @updatedAt @map("updated_at")

  contentPiece        ContentPiece          @relation(fields: [contentPieceId], references: [id], onDelete: Cascade)
  channel             Channel               @relation(fields: [channelId], references: [id], onDelete: Cascade)
  engagementSnapshots EngagementSnapshot[]

  @@index([workspaceId, scheduledAt])
  @@index([contentPieceId])
  @@index([channelId])
  @@map("scheduled_posts")
}

model Campaign {
  id          String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  workspaceId String   @map("workspace_id") @db.Uuid
  name        String
  description String?
  startDate   DateTime? @map("start_date") @db.Date
  endDate     DateTime? @map("end_date") @db.Date
  status      String   @default("draft")
  config      Json     @default("{}")
  createdBy   String?  @map("created_by") @db.Uuid
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  workspace     Workspace      @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  contentPieces ContentPiece[]

  @@index([workspaceId])
  @@map("campaigns")
}

model ApprovalWorkflow {
  id          String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  workspaceId String   @map("workspace_id") @db.Uuid
  name        String
  isDefault   Boolean  @default(false) @map("is_default")
  steps       Json     @default("[]")
  createdAt   DateTime @default(now()) @map("created_at")

  workspace Workspace         @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  requests  ApprovalRequest[]

  @@map("approval_workflows")
}

model ApprovalRequest {
  id             String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  contentPieceId String    @map("content_piece_id") @db.Uuid
  workflowId     String    @map("workflow_id") @db.Uuid
  workspaceId    String    @map("workspace_id") @db.Uuid
  currentStep    Int       @default(1) @map("current_step") @db.SmallInt
  status         String    @default("pending")
  submittedBy    String    @map("submitted_by") @db.Uuid
  submittedAt    DateTime  @default(now()) @map("submitted_at")
  completedAt    DateTime? @map("completed_at")
  decisions      Json      @default("[]")
  aiPrediction   Json?     @map("ai_prediction")

  contentPiece ContentPiece     @relation(fields: [contentPieceId], references: [id], onDelete: Cascade)
  workflow     ApprovalWorkflow @relation(fields: [workflowId], references: [id])

  @@index([contentPieceId])
  @@index([workspaceId, status])
  @@map("approval_requests")
}

model Asset {
  id          String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  workspaceId String   @map("workspace_id") @db.Uuid
  filename    String
  mimeType    String   @map("mime_type")
  fileSize    BigInt   @map("file_size")
  storageUrl  String   @map("storage_url")
  altText     String?  @map("alt_text")
  metadata    Json     @default("{}")
  uploadedBy  String   @map("uploaded_by") @db.Uuid
  createdAt   DateTime @default(now()) @map("created_at")

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  uploader  User      @relation(fields: [uploadedBy], references: [id])

  @@index([workspaceId])
  @@index([workspaceId, mimeType])
  @@map("assets")
}

model EngagementSnapshot {
  id              String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  scheduledPostId String   @map("scheduled_post_id") @db.Uuid
  workspaceId     String   @map("workspace_id") @db.Uuid
  impressions     Int      @default(0)
  reach           Int      @default(0)
  engagementRate  Decimal? @map("engagement_rate") @db.Decimal(7, 6)
  platformMetrics Json     @default("{}") @map("platform_metrics")
  collectedAt     DateTime @default(now()) @map("collected_at")

  scheduledPost ScheduledPost @relation(fields: [scheduledPostId], references: [id], onDelete: Cascade)

  @@index([scheduledPostId])
  @@index([workspaceId, collectedAt(sort: Desc)])
  @@map("engagement_snapshots")
}

model PerformancePrediction {
  id             String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  contentPieceId String   @map("content_piece_id") @db.Uuid
  workspaceId    String   @map("workspace_id") @db.Uuid
  predictions    Json
  predictedAt    DateTime @default(now()) @map("predicted_at")

  contentPiece ContentPiece @relation(fields: [contentPieceId], references: [id], onDelete: Cascade)

  @@index([contentPieceId])
  @@map("performance_predictions")
}

model ActivityLog {
  id           String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  workspaceId  String   @map("workspace_id") @db.Uuid
  actorId      String   @map("actor_id") @db.Uuid
  actorType    String   @default("user") @map("actor_type")
  action       String
  resourceType String   @map("resource_type")
  resourceId   String   @map("resource_id") @db.Uuid
  changes      Json?
  occurredAt   DateTime @default(now()) @map("occurred_at")

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  actor     User      @relation(fields: [actorId], references: [id])

  @@index([workspaceId, occurredAt(sort: Desc)])
  @@index([resourceType, resourceId])
  @@map("activity_log")
}

model Integration {
  id          String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  workspaceId String   @map("workspace_id") @db.Uuid
  provider    String
  isActive    Boolean  @default(true) @map("is_active")
  config      Json     @default("{}")
  connectedBy String?  @map("connected_by") @db.Uuid
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@index([workspaceId])
  @@map("integrations")
}
```

**Testing**:
- `Unit: prisma validate passes with no schema errors`
- `Unit: prisma generate produces typed client with all models`
- `Integration: prisma migrate dev creates all 14 tables in PostgreSQL`
- `Integration: insert workspace with JSONB settings → read back with correct structure`
- `Integration: content_piece with tags TEXT[] → GIN index query returns matching rows`
- `Integration: content_piece with JSONB brief → jsonb_path_ops query filters by keyword`
- `Integration: workspace_members unique constraint prevents duplicate user-workspace pairs`
- `Integration: cascade delete workspace → all child records removed`

#### 1.3 — Authentication & User Management

**What**: Implement NextAuth.js v5 authentication with email/password and Google OAuth, user registration, and session management.

**Design**:

```typescript
// packages/shared/src/types/user.ts
export interface User {
  id: string;
  email: string;
  displayName: string;
  avatarUrl: string | null;
  preferences: UserPreferences;
  createdAt: Date;
  updatedAt: Date;
}

export interface UserPreferences {
  timezone: string;
  notificationChannels: ('email' | 'slack' | 'in_app')[];
  calendarView: 'day' | 'week' | 'month';
  theme: 'light' | 'dark' | 'system';
}

export const DEFAULT_USER_PREFERENCES: UserPreferences = {
  timezone: 'UTC',
  notificationChannels: ['email', 'in_app'],
  calendarView: 'week',
  theme: 'system',
};
```

```typescript
// packages/shared/src/types/workspace.ts
export type WorkspaceRole = 'owner' | 'admin' | 'editor' | 'contributor' | 'viewer';

export interface Workspace {
  id: string;
  name: string;
  slug: string;
  plan: 'free' | 'starter' | 'professional' | 'enterprise';
  settings: WorkspaceSettings;
  createdAt: Date;
  updatedAt: Date;
}

export interface WorkspaceSettings {
  timezone: string;
  defaultLanguage: string;
  brandVoice: string;
  brandColors: string[];
  contentTypesEnabled: string[];
  approvalRequired: boolean;
  defaultApprovalWorkflowId: string | null;
}

export interface WorkspaceMember {
  id: string;
  workspaceId: string;
  userId: string;
  role: WorkspaceRole;
  permissions: MemberPermissions;
  joinedAt: Date;
}

export interface MemberPermissions {
  canPublish: boolean;
  canApprove: boolean;
  canManageChannels: boolean;
  channelRestrictions: string[];
  maxScheduleDaysAhead: number | null;
}

export const ROLE_PERMISSIONS: Record<WorkspaceRole, MemberPermissions> = {
  owner:       { canPublish: true, canApprove: true, canManageChannels: true, channelRestrictions: [], maxScheduleDaysAhead: null },
  admin:       { canPublish: true, canApprove: true, canManageChannels: true, channelRestrictions: [], maxScheduleDaysAhead: null },
  editor:      { canPublish: true, canApprove: true, canManageChannels: false, channelRestrictions: [], maxScheduleDaysAhead: null },
  contributor: { canPublish: false, canApprove: false, canManageChannels: false, channelRestrictions: [], maxScheduleDaysAhead: 30 },
  viewer:      { canPublish: false, canApprove: false, canManageChannels: false, channelRestrictions: [], maxScheduleDaysAhead: null },
};
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/auth/register` | Create account (email, password, displayName) |
| POST | `/api/auth/login` | Email/password login → JWT |
| GET | `/api/auth/session` | Current session user + workspace memberships |
| GET | `/api/users/me` | Current user profile |
| PATCH | `/api/users/me` | Update profile (displayName, avatarUrl, preferences) |

**Testing**:
- `Unit: register with valid email/password → user created, JWT returned`
- `Unit: register with duplicate email → 409 Conflict`
- `Unit: register with invalid email format → 400 ValidationError`
- `Unit: login with correct credentials → 200, JWT with user.id claim`
- `Unit: login with wrong password → 401 Unauthorized`
- `Unit: GET /users/me without auth header → 401`
- `Unit: GET /users/me with valid JWT → 200, user profile`
- `Unit: PATCH /users/me with partial update → only changed fields updated`
- `Integration (mocked): Google OAuth callback → user created or matched by email, JWT returned`

#### 1.4 — Workspace CRUD & Member Management

**What**: Implement workspace creation, slug generation, member invitation, role assignment, and workspace-scoped access control.

**Design**:

```typescript
// apps/api/src/modules/workspace/workspace.service.ts
export class WorkspaceService {
  async create(ownerId: string, dto: CreateWorkspaceDto): Promise<Workspace>;
  async findBySlug(slug: string): Promise<Workspace | null>;
  async findByUserId(userId: string): Promise<WorkspaceWithRole[]>;
  async update(workspaceId: string, dto: UpdateWorkspaceDto): Promise<Workspace>;
  async delete(workspaceId: string): Promise<void>;
  async addMember(workspaceId: string, dto: AddMemberDto): Promise<WorkspaceMember>;
  async updateMemberRole(workspaceId: string, memberId: string, role: WorkspaceRole): Promise<WorkspaceMember>;
  async removeMember(workspaceId: string, memberId: string): Promise<void>;
  async listMembers(workspaceId: string): Promise<WorkspaceMember[]>;
}

interface CreateWorkspaceDto {
  name: string;
  slug?: string;       // Auto-generated from name if omitted
  plan?: string;
  settings?: Partial<WorkspaceSettings>;
}

interface AddMemberDto {
  email: string;       // Invite by email — creates user if not exists
  role: WorkspaceRole;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces` | Create workspace (current user becomes owner) |
| GET | `/api/workspaces` | List workspaces for current user |
| GET | `/api/workspaces/:slug` | Get workspace by slug |
| PATCH | `/api/workspaces/:id` | Update workspace (admin+) |
| DELETE | `/api/workspaces/:id` | Delete workspace (owner only) |
| GET | `/api/workspaces/:id/members` | List members |
| POST | `/api/workspaces/:id/members` | Invite member by email |
| PATCH | `/api/workspaces/:id/members/:memberId` | Update member role |
| DELETE | `/api/workspaces/:id/members/:memberId` | Remove member |

Workspace-scoped guard:

```typescript
// apps/api/src/common/guards/workspace.guard.ts
@Injectable()
export class WorkspaceGuard implements CanActivate {
  // Extracts workspace ID from route param or header
  // Verifies current user is a member of the workspace
  // Attaches workspace and member role to request context
  canActivate(context: ExecutionContext): Promise<boolean>;
}

@Injectable()
export class RoleGuard implements CanActivate {
  // Checks that the current member's role meets the minimum required role
  // Usage: @Roles('editor') on controller methods
  canActivate(context: ExecutionContext): Promise<boolean>;
}
```

**Testing**:
- `Unit: create workspace with name "My Agency" → slug auto-generated as "my-agency"`
- `Unit: create workspace with duplicate slug → slug gets numeric suffix "my-agency-2"`
- `Unit: addMember with valid email and role "editor" → member created, invite email queued`
- `Unit: addMember with email already a member → 409 Conflict`
- `Unit: updateMemberRole from contributor to editor → role changed`
- `Unit: removeMember who is the only owner → 400 "Cannot remove last owner"`
- `Unit: WorkspaceGuard with valid member → passes, attaches role to context`
- `Unit: WorkspaceGuard with non-member → 403 Forbidden`
- `Unit: RoleGuard with editor trying admin action → 403`
- `Unit: RoleGuard with admin trying admin action → passes`
- `Integration: create workspace → GET by slug → returns workspace with owner member`

---

## Phase 2: Content Core & Calendar Foundation

### Purpose
Build the content piece lifecycle, campaign management, and the foundational calendar data model. After this phase, users can create content pieces, assign them to campaigns, manage tags, and view a basic calendar of scheduled items. This is the "heart" of the application — the entities that everything else connects to.

### Tasks

#### 2.1 — Content Piece CRUD

**What**: Implement full CRUD for content pieces with status management, tagging, and workspace-scoped queries.

**Design**:

```typescript
// packages/shared/src/types/content.ts
export type ContentType =
  | 'blog_post' | 'social_post' | 'email' | 'newsletter'
  | 'video' | 'podcast' | 'infographic' | 'whitepaper'
  | 'case_study' | 'webinar' | 'other';

export type ContentStatus =
  | 'idea' | 'briefed' | 'in_progress' | 'in_review'
  | 'approved' | 'scheduled' | 'published' | 'archived';

export const CONTENT_STATUS_TRANSITIONS: Record<ContentStatus, ContentStatus[]> = {
  idea:        ['briefed', 'in_progress', 'archived'],
  briefed:     ['in_progress', 'archived'],
  in_progress: ['in_review', 'archived'],
  in_review:   ['approved', 'in_progress'],  // rejection sends back to in_progress
  approved:    ['scheduled', 'in_progress'],  // can revert if edits needed
  scheduled:   ['published', 'approved'],     // can unschedule
  published:   ['archived'],
  archived:    ['idea'],                       // can reactivate
};

export interface ContentPiece {
  id: string;
  workspaceId: string;
  title: string;
  contentType: ContentType;
  status: ContentStatus;
  body: string | null;
  authorId: string | null;
  assigneeId: string | null;
  campaignId: string | null;
  publishDate: Date | null;
  metadata: Record<string, unknown>;
  brief: ContentBrief | null;
  tags: string[];
  createdAt: Date;
  updatedAt: Date;
}
```

```typescript
// apps/api/src/modules/content/content.service.ts
export class ContentService {
  async create(workspaceId: string, dto: CreateContentDto): Promise<ContentPiece>;
  async findById(workspaceId: string, id: string): Promise<ContentPiece>;
  async list(workspaceId: string, filters: ContentFilters): Promise<PaginatedResult<ContentPiece>>;
  async update(workspaceId: string, id: string, dto: UpdateContentDto): Promise<ContentPiece>;
  async updateStatus(workspaceId: string, id: string, newStatus: ContentStatus): Promise<ContentPiece>;
  async delete(workspaceId: string, id: string): Promise<void>;
  async addTag(workspaceId: string, id: string, tag: string): Promise<ContentPiece>;
  async removeTag(workspaceId: string, id: string, tag: string): Promise<ContentPiece>;
  async listTags(workspaceId: string): Promise<TagCount[]>;
}

interface ContentFilters {
  status?: ContentStatus[];
  contentType?: ContentType[];
  tags?: string[];
  campaignId?: string;
  authorId?: string;
  assigneeId?: string;
  publishDateFrom?: Date;
  publishDateTo?: Date;
  search?: string;       // Full-text search on title and body
  page?: number;
  pageSize?: number;
  sortBy?: 'createdAt' | 'updatedAt' | 'publishDate' | 'title';
  sortOrder?: 'asc' | 'desc';
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces/:wid/content` | Create content piece |
| GET | `/api/workspaces/:wid/content` | List with filters and pagination |
| GET | `/api/workspaces/:wid/content/:id` | Get single content piece |
| PATCH | `/api/workspaces/:wid/content/:id` | Update content piece |
| PATCH | `/api/workspaces/:wid/content/:id/status` | Transition status |
| DELETE | `/api/workspaces/:wid/content/:id` | Soft delete (archive) |
| POST | `/api/workspaces/:wid/content/:id/tags` | Add tag |
| DELETE | `/api/workspaces/:wid/content/:id/tags/:tag` | Remove tag |
| GET | `/api/workspaces/:wid/tags` | List all tags with counts |

**Testing**:
- `Unit: create content piece with valid data → saved with status "idea"`
- `Unit: create content piece with invalid contentType → 400 ValidationError`
- `Unit: updateStatus from "idea" to "in_progress" → status updated, activity logged`
- `Unit: updateStatus from "idea" to "published" → 400 "Invalid status transition"`
- `Unit: list with status filter ["idea", "briefed"] → only matching pieces returned`
- `Unit: list with tags filter ["product-launch"] → GIN index query, matching pieces returned`
- `Unit: list with search "Q3 launch" → full-text matches on title/body`
- `Unit: addTag "urgent" → tag appended to tags array`
- `Unit: addTag duplicate → no change, no error`
- `Unit: removeTag → tag removed from array`
- `Unit: listTags → aggregated tag names with content piece counts`
- `Unit: delete → status set to "archived", not physically deleted`
- `Integration: content piece in workspace A not visible in workspace B`

#### 2.2 — Campaign Management

**What**: Implement campaign CRUD with content association and date-range management.

**Design**:

```typescript
// packages/shared/src/types/campaign.ts
export type CampaignStatus = 'draft' | 'active' | 'paused' | 'completed';

export interface Campaign {
  id: string;
  workspaceId: string;
  name: string;
  description: string | null;
  startDate: string | null;    // ISO 8601 date
  endDate: string | null;
  status: CampaignStatus;
  config: CampaignConfig;
  createdBy: string | null;
  createdAt: Date;
  updatedAt: Date;
}

export interface CampaignConfig {
  goal?: string;
  targetMetrics?: {
    totalReach?: number;
    engagementRate?: number;
    linkClicks?: number;
  };
  channels?: string[];
  personas?: string[];
  budget?: {
    total: number;
    currency: string;
    perChannel?: Record<string, number>;
  };
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces/:wid/campaigns` | Create campaign |
| GET | `/api/workspaces/:wid/campaigns` | List campaigns |
| GET | `/api/workspaces/:wid/campaigns/:id` | Get campaign with content piece summary |
| PATCH | `/api/workspaces/:wid/campaigns/:id` | Update campaign |
| DELETE | `/api/workspaces/:wid/campaigns/:id` | Delete campaign (disassociates content) |

**Testing**:
- `Unit: create campaign with start/end dates → saved, status "draft"`
- `Unit: create campaign with endDate before startDate → 400 ValidationError`
- `Unit: get campaign → includes count of associated content pieces by status`
- `Unit: delete campaign → content pieces retain data but campaignId set to null`
- `Integration: associate 3 content pieces with campaign → campaign GET returns summary`

#### 2.3 — Channel Management

**What**: Implement channel CRUD with PESO model categorisation and platform-specific connection metadata.

**Design**:

```typescript
// packages/shared/src/types/channel.ts
export type Platform =
  | 'facebook' | 'instagram' | 'linkedin' | 'twitter' | 'tiktok'
  | 'youtube' | 'pinterest' | 'blog' | 'email' | 'podcast' | 'other';

export type PesoCategory = 'paid' | 'earned' | 'shared' | 'owned';

export interface Channel {
  id: string;
  workspaceId: string;
  name: string;
  platform: Platform;
  pesoCategory: PesoCategory;
  isActive: boolean;
  connection: ChannelConnection;
  createdAt: Date;
  updatedAt: Date;
}

export interface ChannelConnection {
  accountId?: string;
  accountName?: string;
  accessToken?: string;      // Encrypted at rest
  refreshToken?: string;     // Encrypted at rest
  tokenExpiresAt?: string;
  scopes?: string[];
  capabilities?: string[];
  characterLimits?: Record<string, number>;
  connectedBy?: string;
}

// Platform capabilities for UI rendering
export const PLATFORM_CAPABILITIES: Record<Platform, string[]> = {
  facebook:  ['text', 'image', 'video', 'carousel', 'story', 'reel', 'link'],
  instagram: ['image', 'video', 'carousel', 'story', 'reel'],
  linkedin:  ['text', 'image', 'video', 'article', 'document', 'poll'],
  twitter:   ['text', 'image', 'video', 'thread', 'poll'],
  tiktok:    ['video'],
  youtube:   ['video', 'short'],
  pinterest: ['image', 'video', 'idea_pin'],
  blog:      ['article'],
  email:     ['html_email', 'plain_text'],
  podcast:   ['audio'],
  other:     ['text'],
};

export const PLATFORM_CHARACTER_LIMITS: Record<Platform, Record<string, number>> = {
  facebook:  { text: 63206, link_description: 500 },
  instagram: { caption: 2200, bio: 150 },
  linkedin:  { post: 3000, article: 120000, comment: 1250 },
  twitter:   { tweet: 280, dm: 10000 },
  tiktok:    { caption: 2200 },
  youtube:   { title: 100, description: 5000 },
  pinterest: { description: 500, title: 100 },
  blog:      {},
  email:     { subject: 200, preview: 150 },
  podcast:   { title: 200, description: 4000 },
  other:     {},
};
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces/:wid/channels` | Create channel |
| GET | `/api/workspaces/:wid/channels` | List channels (with connection status) |
| GET | `/api/workspaces/:wid/channels/:id` | Get channel details |
| PATCH | `/api/workspaces/:wid/channels/:id` | Update channel |
| DELETE | `/api/workspaces/:wid/channels/:id` | Deactivate channel |
| POST | `/api/workspaces/:wid/channels/:id/connect` | Initiate OAuth flow for social platform |
| POST | `/api/workspaces/:wid/channels/:id/disconnect` | Revoke OAuth connection |

**Testing**:
- `Unit: create channel with platform "instagram" and peso_category "shared" → saved`
- `Unit: create channel with invalid platform → 400 ValidationError`
- `Unit: list channels → includes connection status (connected/disconnected/expired)`
- `Unit: disconnect channel → connection JSONB cleared, isActive set to false`
- `Unit: channel connection with expired token_expires_at → status shows "expired"`
- `Integration: PLATFORM_CAPABILITIES lookup returns correct capabilities per platform`

#### 2.4 — Scheduled Posts & Calendar View API

**What**: Implement scheduled post creation, the calendar query endpoint (week/month/day views), and iCalendar export (RFC 5545).

**Design**:

```typescript
// packages/shared/src/types/calendar.ts
export type PostStatus = 'draft' | 'scheduled' | 'publishing' | 'published' | 'failed' | 'cancelled';

export interface ScheduledPost {
  id: string;
  workspaceId: string;
  contentPieceId: string;
  channelId: string;
  scheduledAt: Date;
  publishedAt: Date | null;
  status: PostStatus;
  icalUid: string;
  platformContent: PlatformContent;
  platformResponse: Record<string, unknown> | null;
  errorDetails: Record<string, unknown> | null;
  retryCount: number;
  createdAt: Date;
  updatedAt: Date;
}

// Calendar entry: denormalised for rendering
export interface CalendarEntry {
  id: string;
  contentPieceId: string;
  contentTitle: string;
  contentType: ContentType;
  channelId: string;
  channelName: string;
  platform: Platform;
  pesoCategory: PesoCategory;
  scheduledAt: Date;
  publishedAt: Date | null;
  status: PostStatus;
  campaignName: string | null;
  authorName: string | null;
  thumbnailUrl: string | null;
}

// Union type for platform-specific content
export type PlatformContent =
  | InstagramPostContent
  | LinkedInPostContent
  | TwitterPostContent
  | FacebookPostContent
  | EmailContent
  | GenericPostContent;

export interface InstagramPostContent {
  caption: string;
  hashtags: string[];
  mentions: string[];
  postType: 'image' | 'carousel' | 'reel' | 'story';
  slides?: { mediaId: string; altText: string }[];
  firstComment?: string;
  locationTag?: string;
}

export interface TwitterPostContent {
  postType: 'single' | 'thread';
  tweets: { text: string; mediaIds: string[] }[];
}

export interface LinkedInPostContent {
  postType: 'post' | 'article' | 'document';
  text?: string;
  headline?: string;
  bodyHtml?: string;
  visibility: 'PUBLIC' | 'CONNECTIONS';
  hashtags: string[];
}
```

```typescript
// apps/api/src/modules/calendar/calendar.service.ts
export class CalendarService {
  async getCalendarEntries(
    workspaceId: string,
    dateFrom: Date,
    dateTo: Date,
    filters?: { channelIds?: string[]; platforms?: Platform[]; statuses?: PostStatus[] }
  ): Promise<CalendarEntry[]>;

  async exportICalendar(workspaceId: string, dateFrom: Date, dateTo: Date): Promise<string>;
  // Returns RFC 5545 iCalendar string with VEVENT per scheduled post
}
```

iCalendar export format (RFC 5545):

```
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//Content Calendar//EN
X-WR-CALNAME:Content Calendar - {workspace_name}
BEGIN:VEVENT
UID:{ical_uid}
DTSTART:{scheduled_at in UTC}
SUMMARY:{content_title} [{platform}]
DESCRIPTION:{caption or first 200 chars of body}
CATEGORIES:{peso_category}
STATUS:{CONFIRMED|TENTATIVE|CANCELLED}
END:VEVENT
END:VCALENDAR
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces/:wid/scheduled-posts` | Schedule a post |
| GET | `/api/workspaces/:wid/scheduled-posts/:id` | Get scheduled post |
| PATCH | `/api/workspaces/:wid/scheduled-posts/:id` | Update scheduled post (reschedule, edit content) |
| DELETE | `/api/workspaces/:wid/scheduled-posts/:id` | Cancel scheduled post |
| GET | `/api/workspaces/:wid/calendar` | Calendar entries (date range + filters) |
| GET | `/api/workspaces/:wid/calendar/export.ics` | iCalendar export |

**Testing**:
- `Unit: schedule post for future date → status "scheduled", icalUid generated`
- `Unit: schedule post for past date → 400 "Cannot schedule in the past"`
- `Unit: schedule post with invalid platform_content for Instagram → 400 JSON Schema validation error`
- `Unit: reschedule post → scheduledAt updated, activity logged`
- `Unit: cancel post → status "cancelled"`
- `Unit: calendar entries for date range → returns denormalised CalendarEntry objects`
- `Unit: calendar entries filtered by platform ["instagram"] → only Instagram posts`
- `Unit: iCalendar export → valid RFC 5545 output parsed by ical.js`
- `Unit: iCalendar VEVENT has correct UID, DTSTART, SUMMARY format`
- `Integration: schedule 3 posts across 2 channels → calendar endpoint returns all 3 in correct date range`
- `Integration: post outside date range → not included in calendar query`

---

## Phase 3: Frontend Calendar & Content Management

### Purpose
Build the web frontend with interactive calendar views, content management UI, and campaign dashboard. After this phase, users have a fully functional web application for managing content pieces, viewing the calendar, and navigating campaigns — the core user experience.

### Tasks

#### 3.1 — Authentication & Layout Shell

**What**: Implement Next.js app with NextAuth.js integration, login/signup pages, workspace selector, and authenticated dashboard layout with sidebar navigation.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/layout.tsx
// Dashboard layout with:
// - Sidebar: workspace selector, navigation links (Calendar, Content, Campaigns, Approvals, Analytics, Channels, Assets, Settings)
// - Top bar: search, notifications bell, user avatar/menu
// - Main content area
// - Workspace context provider wrapping all dashboard routes

// apps/web/src/lib/api-client.ts
export class ApiClient {
  constructor(private baseUrl: string, private sessionToken: string);

  // Workspace-scoped methods
  content: {
    list(workspaceId: string, filters: ContentFilters): Promise<PaginatedResult<ContentPiece>>;
    create(workspaceId: string, dto: CreateContentDto): Promise<ContentPiece>;
    update(workspaceId: string, id: string, dto: UpdateContentDto): Promise<ContentPiece>;
    updateStatus(workspaceId: string, id: string, status: ContentStatus): Promise<ContentPiece>;
    delete(workspaceId: string, id: string): Promise<void>;
  };
  calendar: {
    entries(workspaceId: string, from: Date, to: Date, filters?: CalendarFilters): Promise<CalendarEntry[]>;
    exportIcs(workspaceId: string, from: Date, to: Date): Promise<Blob>;
  };
  scheduledPosts: {
    create(workspaceId: string, dto: CreateScheduledPostDto): Promise<ScheduledPost>;
    update(workspaceId: string, id: string, dto: UpdateScheduledPostDto): Promise<ScheduledPost>;
    cancel(workspaceId: string, id: string): Promise<void>;
  };
  // ... additional resource clients
}
```

**Testing**:
- `E2E: visit / without auth → redirected to /login`
- `E2E: login with valid credentials → redirected to /dashboard/calendar`
- `E2E: workspace selector shows user's workspaces → click switches workspace`
- `E2E: sidebar navigation links route to correct pages`
- `E2E: logout → session cleared, redirected to /login`

#### 3.2 — Calendar Views (Week/Month/Day)

**What**: Build interactive calendar grid with week, month, and day views. Calendar entries rendered as coloured cards by platform. Drag-and-drop rescheduling.

**Design**:

```typescript
// apps/web/src/components/calendar/CalendarGrid.tsx
interface CalendarGridProps {
  view: 'day' | 'week' | 'month';
  entries: CalendarEntry[];
  onEntryClick: (entry: CalendarEntry) => void;
  onEntryDrop: (entryId: string, newDate: Date) => void;
  onDateSelect: (date: Date) => void;
  filters: CalendarFilters;
  onFiltersChange: (filters: CalendarFilters) => void;
}

// Each CalendarEntryCard shows:
// - Platform icon (coloured by platform)
// - Content title (truncated)
// - Scheduled time
// - Status indicator (dot colour)
// - PESO category badge
// - Campaign name (if assigned)

// Colour scheme by platform:
// Facebook: #1877F2, Instagram: #E4405F, LinkedIn: #0A66C2, Twitter: #1DA1F2
// TikTok: #000000, YouTube: #FF0000, Blog: #4CAF50, Email: #FF9800

interface CalendarFilters {
  platforms: Platform[];
  channels: string[];
  statuses: PostStatus[];
  campaigns: string[];
  search: string;
}
```

Calendar drag-and-drop reschedule flow:
1. User drags CalendarEntryCard to a new date/time cell
2. `onEntryDrop` fires with entryId and new Date
3. Optimistic UI update moves the card
4. API call: `PATCH /scheduled-posts/:id` with new `scheduledAt`
5. On failure: revert card to original position, show error toast

**Testing**:
- `E2E: month view shows all entries for current month`
- `E2E: switch to week view → 7-day grid with hourly slots`
- `E2E: switch to day view → single day with 30-minute slots`
- `E2E: click entry card → opens detail slide-over panel`
- `E2E: drag entry from Monday to Wednesday → PATCH API called, entry moves`
- `E2E: drag entry to past date → error toast, entry reverts`
- `E2E: filter by platform "instagram" → only Instagram entries visible`
- `E2E: filter by campaign → only campaign entries visible`
- `E2E: navigate forward/back week → date range updates, new entries load`
- `E2E: calendar with 50+ entries in one month → renders without layout overlap`

#### 3.3 — Content List & Editor

**What**: Build content piece list view with filters/search, content detail view with inline editing, and status transition UI.

**Design**:

Content list view: table layout with columns (Title, Type, Status, Author, Campaign, Publish Date, Tags) and filter bar. Supports bulk actions (archive, tag, assign).

Content detail view: split-pane layout with content editor on the left (title, body as markdown/rich-text, metadata fields), and context panel on the right (status workflow, brief summary, tags, campaign assignment, scheduled posts timeline).

Status workflow visualisation: horizontal stepper showing the status pipeline (idea > briefed > in_progress > in_review > approved > scheduled > published). Current status highlighted. Clickable transitions show confirmation dialog.

**Testing**:
- `E2E: content list loads with pagination (20 items per page)`
- `E2E: filter by status "in_review" → only in-review pieces shown`
- `E2E: search "product launch" → matching titles highlighted`
- `E2E: click content piece → detail view opens`
- `E2E: edit title inline → PATCH API called on blur, title updated`
- `E2E: click status transition "idea → in_progress" → confirmation dialog → status updated`
- `E2E: click invalid transition → button disabled, tooltip explains why`
- `E2E: assign tag via tag input → tag added to piece`
- `E2E: bulk select 3 pieces → bulk archive → all 3 archived`

#### 3.4 — Campaign Dashboard

**What**: Build campaign list and detail views with content piece association and progress tracking.

**Design**:

Campaign detail view shows: campaign name and description, date range bar, content piece count by status (stacked bar chart), list of associated content pieces sortable by status/date, and a mini-calendar showing only this campaign's scheduled posts.

**Testing**:
- `E2E: campaign list shows all campaigns with status badges`
- `E2E: create campaign → form with name, description, dates, config`
- `E2E: campaign detail → shows content pieces grouped by status`
- `E2E: associate content piece with campaign → appears in campaign detail`
- `E2E: campaign progress bar reflects content status distribution`

---

## Phase 4: Approval Workflows

### Purpose
Implement configurable multi-step approval workflows so content must pass through designated reviewers before publishing. After this phase, workspace admins can define approval chains, contributors can submit content for review, and reviewers can approve, reject, or request revisions with comments.

### Tasks

#### 4.1 — Approval Workflow Configuration

**What**: Implement approval workflow CRUD with multi-step configuration, step ordering, and default workflow assignment per workspace.

**Design**:

```typescript
// packages/shared/src/types/approval.ts
export interface ApprovalWorkflow {
  id: string;
  workspaceId: string;
  name: string;
  isDefault: boolean;
  steps: ApprovalStep[];
  createdAt: Date;
}

export interface ApprovalStep {
  order: number;
  name: string;
  approverType: 'user' | 'team' | 'role';
  approverId?: string;
  approverRole?: WorkspaceRole;
  required: boolean;
}

export type ApprovalDecisionType = 'approved' | 'rejected' | 'revision_requested';

export interface ApprovalDecision {
  stepOrder: number;
  stepName: string;
  reviewerId: string;
  reviewerName: string;
  decision: ApprovalDecisionType;
  comment: string;
  decidedAt: string;
}

export type ApprovalStatus = 'pending' | 'in_review' | 'approved' | 'rejected' | 'cancelled';

export interface ApprovalRequest {
  id: string;
  contentPieceId: string;
  workflowId: string;
  workspaceId: string;
  currentStep: number;
  status: ApprovalStatus;
  submittedBy: string;
  submittedAt: Date;
  completedAt: Date | null;
  decisions: ApprovalDecision[];
  aiPrediction: ApprovalPrediction | null;
}

export interface ApprovalPrediction {
  predictedOutcome: ApprovalDecisionType;
  confidence: number;
  flaggedIssues: string[];
  modelVersion: string;
  predictedAt: string;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces/:wid/approval-workflows` | Create workflow |
| GET | `/api/workspaces/:wid/approval-workflows` | List workflows |
| PATCH | `/api/workspaces/:wid/approval-workflows/:id` | Update workflow |
| DELETE | `/api/workspaces/:wid/approval-workflows/:id` | Delete workflow |
| PATCH | `/api/workspaces/:wid/approval-workflows/:id/default` | Set as default |

**Testing**:
- `Unit: create workflow with 3 steps → steps stored as ordered JSONB array`
- `Unit: create workflow with duplicate step_order → 400 "Step orders must be unique"`
- `Unit: set workflow as default → previous default workflow unset`
- `Unit: delete workflow with active approval requests → 400 "Cannot delete workflow with active requests"`
- `Unit: update workflow steps → existing pending requests unaffected (they snapshot steps at submission)`

#### 4.2 — Approval Request Lifecycle

**What**: Implement the approval request state machine: submit for approval, step-by-step review decisions, revision requests, and final approval/rejection.

**Design**:

Approval request state machine:
```
                      ┌──────────────────┐
            submit    │     pending      │
           ────────►  │  (waiting for    │
                      │   first step)    │
                      └────────┬─────────┘
                               │ auto-advance to step 1
                      ┌────────▼─────────┐
                      │    in_review     │
                      │  (current step   │◄──── revision resolved
                      │   under review)  │
                      └──┬─────┬────┬────┘
                         │     │    │
                 approved│     │    │ rejected
                 (step)  │     │    │
                         │     │    └────────────────────┐
                         │     │ revision_requested      │
                         ▼     ▼                         ▼
                   ┌──────┐  ┌──────────┐         ┌──────────┐
                   │next  │  │ back to  │         │ rejected │
                   │step? │  │contributor│         │  (final) │
                   │      │  │for fixes │         └──────────┘
                   └──┬───┘  └──────────┘
                      │
              all steps approved
                      │
                      ▼
                ┌──────────┐
                │ approved │
                │  (final) │
                └──────────┘
```

```typescript
// apps/api/src/modules/approval/approval.service.ts
export class ApprovalService {
  async submitForApproval(workspaceId: string, contentPieceId: string, workflowId?: string): Promise<ApprovalRequest>;
  async makeDecision(requestId: string, reviewerId: string, dto: MakeDecisionDto): Promise<ApprovalRequest>;
  async cancelRequest(requestId: string, userId: string): Promise<ApprovalRequest>;
  async getApprovalQueue(workspaceId: string, filters: ApprovalQueueFilters): Promise<ApprovalRequest[]>;
  async getRequestHistory(contentPieceId: string): Promise<ApprovalRequest[]>;
}

interface MakeDecisionDto {
  decision: ApprovalDecisionType;
  comment: string;
}

interface ApprovalQueueFilters {
  status?: ApprovalStatus[];
  reviewerId?: string;      // "my reviews" filter
  submittedById?: string;   // "my submissions" filter
}
```

Side effects on approval state changes:
- On `approved` (all steps passed): content piece status → `approved`, notify submitter
- On `rejected`: content piece status → `in_progress`, notify submitter with rejection comment
- On `revision_requested`: content piece status → `in_progress`, notify submitter with revision notes

**Testing**:
- `Unit: submitForApproval → creates request with status "pending", snapshots workflow steps`
- `Unit: submitForApproval for content not in "in_review" status → 400`
- `Unit: makeDecision "approved" on step 1 of 3 → advances to step 2, status stays "in_review"`
- `Unit: makeDecision "approved" on step 3 of 3 → status "approved", content piece status → "approved"`
- `Unit: makeDecision "rejected" → status "rejected", content piece status → "in_progress"`
- `Unit: makeDecision "revision_requested" → content piece status → "in_progress", request stays "in_review"`
- `Unit: makeDecision by non-step-reviewer → 403 "Not authorised for this step"`
- `Unit: cancelRequest by submitter → status "cancelled"`
- `Unit: getApprovalQueue with reviewerId filter → only requests where user is current step reviewer`
- `Unit: decisions array accumulates all decisions in order`
- `Integration: full approval flow (submit → step1 approve → step2 approve → content approved)`
- `Integration: revision flow (submit → step1 revision_requested → contributor fixes → resubmit → step1 approve)`

#### 4.3 — Approval UI

**What**: Build approval queue page, approval detail view with decision timeline, and inline review interface.

**Design**:

Approval queue page: table showing pending/in-review requests with columns (Content Title, Workflow, Current Step, Submitted By, Submitted Date, Status). Tabs for "My Reviews", "My Submissions", "All".

Approval detail view: split layout with content preview on the left and approval timeline on the right. Timeline shows each step with decision status (approved/rejected/revision/pending). Decision form at bottom with comment textarea and three action buttons (Approve, Request Revision, Reject).

**Testing**:
- `E2E: approval queue loads with pending requests`
- `E2E: click request → detail view with content preview and timeline`
- `E2E: click "Approve" → decision recorded, next step shown or request completed`
- `E2E: click "Request Revision" with comment → submitter notified, content status reverts`
- `E2E: reviewer not assigned to current step → decision buttons disabled`
- `E2E: "My Reviews" tab → filters to requests where current user is reviewer`

---

## Phase 5: Asset Management & File Storage

### Purpose
Implement digital asset upload, storage, and management so content pieces can include images, videos, and documents. After this phase, users can upload files, browse an asset library, attach assets to content and scheduled posts, and generate WCAG 2.2-compliant alt text suggestions.

### Tasks

#### 5.1 — Asset Upload & Storage

**What**: Implement pre-signed URL upload flow to S3-compatible storage, asset metadata extraction, and thumbnail generation.

**Design**:

```typescript
// apps/api/src/modules/asset/asset.service.ts
export class AssetService {
  async createUploadUrl(workspaceId: string, dto: CreateUploadDto): Promise<UploadUrlResponse>;
  async confirmUpload(workspaceId: string, assetId: string): Promise<Asset>;
  async list(workspaceId: string, filters: AssetFilters): Promise<PaginatedResult<Asset>>;
  async getById(workspaceId: string, id: string): Promise<Asset>;
  async update(workspaceId: string, id: string, dto: UpdateAssetDto): Promise<Asset>;
  async delete(workspaceId: string, id: string): Promise<void>;
}

interface CreateUploadDto {
  filename: string;
  mimeType: string;
  fileSize: number;
}

interface UploadUrlResponse {
  assetId: string;
  uploadUrl: string;          // Pre-signed S3 PUT URL
  expiresAt: Date;            // URL expiry (15 minutes)
}

interface AssetFilters {
  mimeType?: string;          // Filter by type: 'image/*', 'video/*', 'application/pdf'
  search?: string;            // Search filename and alt_text
  uploadedBy?: string;
  page?: number;
  pageSize?: number;
}
```

Upload flow:
1. Client calls `POST /assets/upload` with filename, mimeType, fileSize
2. Server validates file type and size limits, creates Asset record (status: "uploading"), generates pre-signed S3 PUT URL
3. Client uploads file directly to S3 using the pre-signed URL
4. Client calls `POST /assets/:id/confirm` to mark upload complete
5. Server triggers async metadata extraction job (dimensions, duration, thumbnail generation)

**Testing**:
- `Unit: createUploadUrl with valid image type → pre-signed URL returned`
- `Unit: createUploadUrl with disallowed mime type (application/exe) → 400`
- `Unit: createUploadUrl exceeding plan file size limit → 413`
- `Unit: confirmUpload → asset status "ready", metadata extraction job queued`
- `Unit: list assets filtered by mimeType "image/*" → only images returned`
- `Unit: delete asset → S3 object deleted, database record removed`
- `Integration (mocked S3): full upload flow → asset stored with correct metadata`
- `Integration: metadata extraction from uploaded image → width, height, format populated`

#### 5.2 — Asset Library UI

**What**: Build asset library page with grid/list views, upload dropzone, search, and attachment to content pieces.

**Design**:

Asset library: grid view showing thumbnails with filename, type icon, size, and upload date. Upload area at top with drag-and-drop zone. Search bar for filename and alt-text search. Bulk delete and download.

Asset picker modal: reusable component for selecting assets when editing content pieces or scheduled posts. Shows asset library in compact grid mode with multi-select capability.

Alt-text editor: inline field on each asset for WCAG 2.2 alt-text with character count and AI-suggested alt-text (Phase 7).

**Testing**:
- `E2E: drag image file onto upload zone → file uploads, thumbnail appears in grid`
- `E2E: click asset → detail panel shows metadata (dimensions, size, alt-text field)`
- `E2E: edit alt-text → saved via PATCH API`
- `E2E: asset picker modal in content editor → select 2 images → attached to content piece`
- `E2E: search assets by filename → matching results shown`
- `E2E: delete asset attached to content piece → warning dialog, confirmation removes attachment`

---

## Phase 6: Social Platform Publishing

### Purpose
Connect to social media platform APIs and implement automated publishing of scheduled posts. After this phase, scheduled posts are automatically published to Facebook, Instagram, LinkedIn, and Twitter at the scheduled time via background workers.

### Tasks

#### 6.1 — Social Publisher Interface & Adapters

**What**: Define the SocialPublisher adapter interface and implement platform-specific adapters for Facebook (Meta Graph API), Instagram, LinkedIn, and Twitter (X API v2).

**Design**:

```typescript
// apps/api/src/modules/channel/publishers/publisher.interface.ts
export interface SocialPublisher {
  readonly platform: Platform;

  publish(post: PublishRequest): Promise<PublishResult>;
  validateContent(content: PlatformContent): ValidationResult;
  refreshToken(connection: ChannelConnection): Promise<ChannelConnection>;
  getMetrics(postId: string, connection: ChannelConnection): Promise<PlatformMetrics>;
}

export interface PublishRequest {
  platformContent: PlatformContent;
  connection: ChannelConnection;
  assets: { id: string; url: string; mimeType: string; altText?: string }[];
}

export interface PublishResult {
  success: boolean;
  platformPostId?: string;
  platformUrl?: string;
  error?: { code: string; message: string; retryable: boolean };
}

export interface ValidationResult {
  valid: boolean;
  errors: { field: string; message: string }[];
  warnings: { field: string; message: string }[];
}

// apps/api/src/modules/channel/publishers/facebook.publisher.ts
export class FacebookPublisher implements SocialPublisher {
  readonly platform = 'facebook';
  // Uses Meta Graph API v21.0
  // POST /{page-id}/feed for text/link posts
  // POST /{page-id}/photos for single image
  // POST /{page-id}/videos for video upload
  async publish(post: PublishRequest): Promise<PublishResult>;
  async validateContent(content: FacebookPostContent): ValidationResult;
  async refreshToken(connection: ChannelConnection): Promise<ChannelConnection>;
  async getMetrics(postId: string, connection: ChannelConnection): Promise<PlatformMetrics>;
}

// Similar implementations for:
// InstagramPublisher — uses Meta Graph API for Instagram
// LinkedInPublisher — uses LinkedIn Marketing API (Restli 2.0)
// TwitterPublisher  — uses X API v2
```

**Testing**:
- `Unit: FacebookPublisher.validateContent with valid text post → { valid: true }`
- `Unit: FacebookPublisher.validateContent with caption exceeding 63206 chars → { valid: false, errors: ["caption too long"] }`
- `Unit: TwitterPublisher.validateContent with tweet exceeding 280 chars → validation error`
- `Unit: TwitterPublisher.validateContent with valid thread → { valid: true }`
- `Integration (mocked API): FacebookPublisher.publish → returns platformPostId and URL`
- `Integration (mocked API): publish with expired token → refreshToken called first, then publish retried`
- `Integration (mocked API): publish with rate limit response → PublishResult with retryable: true`
- `Integration (mocked API): InstagramPublisher.publish carousel → all slides uploaded, carousel published`

#### 6.2 — Publish Worker & Scheduler

**What**: Implement the BullMQ publish worker that polls for scheduled posts due for publishing and dispatches them to the appropriate SocialPublisher adapter.

**Design**:

```typescript
// apps/api/src/workers/publish.worker.ts
// BullMQ worker configuration:
// - Queue name: 'publish'
// - Job type: 'publish-post'
// - Job data: { scheduledPostId: string, workspaceId: string }
// - Concurrency: 5
// - Rate limiting: per-platform limits (Facebook: 200/hour, Twitter: 300/15min, LinkedIn: 100/day)

// Scheduler cron job (runs every minute):
// 1. Query: SELECT * FROM scheduled_posts WHERE status = 'scheduled' AND scheduled_at <= now() + INTERVAL '1 minute'
// 2. For each post, enqueue a 'publish-post' job with delay = (scheduled_at - now()) ms
// 3. Set post status to 'publishing'

// Worker job handler:
// 1. Load scheduled post with content piece and channel
// 2. Validate platform content via SocialPublisher.validateContent()
// 3. If assets referenced, generate pre-signed download URLs
// 4. Call SocialPublisher.publish()
// 5. On success: set status 'published', store platformPostId and platformUrl
// 6. On failure: if retryable, increment retry_count (max 3), re-enqueue with exponential backoff
// 7. On permanent failure: set status 'failed', store error details
// 8. Log activity in activity_log
```

**Testing**:
- `Unit: scheduler query finds posts due within 1 minute → jobs enqueued`
- `Unit: scheduler ignores posts already in "publishing" status`
- `Unit: worker publishes successfully → status "published", platform_response populated`
- `Unit: worker handles retryable failure → retry_count incremented, job re-enqueued`
- `Unit: worker handles 3 failed retries → status "failed", error_details populated`
- `Unit: worker handles non-retryable error (invalid token) → status "failed" immediately`
- `Unit: rate limiter respects per-platform limits`
- `Integration: schedule post 1 minute from now → worker picks up and publishes`
- `Integration: concurrent publishing of 5 posts → all processed without race conditions`

#### 6.3 — OAuth Connection Flow

**What**: Implement the OAuth 2.0 connection flow for each social platform, including token storage, refresh, and disconnection.

**Design**:

OAuth flow:
1. User clicks "Connect" for a channel → `POST /channels/:id/connect` → server generates OAuth URL with state parameter
2. User redirected to platform's OAuth consent screen
3. Platform redirects back to `/api/auth/callback/:platform` with authorization code
4. Server exchanges code for access/refresh tokens (per OAuth 2.0 RFC 6749)
5. Tokens encrypted at application layer (AES-256-GCM) and stored in channel `connection` JSONB
6. Channel marked as connected

Token refresh: BullMQ cron job runs daily, checks `token_expires_at` for all channels, refreshes tokens expiring within 7 days.

**Testing**:
- `Unit: connect endpoint generates correct OAuth URL per platform`
- `Unit: callback with valid code → tokens stored encrypted in channel.connection`
- `Unit: callback with invalid state → 400 "Invalid OAuth state"`
- `Unit: token refresh job → refreshes tokens expiring within 7 days`
- `Unit: disconnect → tokens cleared, channel marked inactive`
- `Integration (mocked OAuth): full OAuth flow for Facebook → channel connected with valid tokens`

---

## Phase 7: AI Brief Generation & Content Intelligence

### Purpose
Implement AI-powered content brief generation, content repurposing suggestions, and performance prediction. This is the core AI-native differentiator. After this phase, users can generate comprehensive briefs from a topic input, get cross-channel repurposing suggestions, and see predicted engagement before publishing.

### Tasks

#### 7.1 — LLM Provider Abstraction

**What**: Build a provider-agnostic LLM service that supports OpenAI and Anthropic, with prompt template management and response parsing.

**Design**:

```typescript
// apps/api/src/modules/ai/llm.service.ts
export interface LLMProvider {
  readonly name: string;
  complete(request: LLMRequest): Promise<LLMResponse>;
}

export interface LLMRequest {
  systemPrompt: string;
  userPrompt: string;
  maxTokens: number;
  temperature: number;
  responseFormat?: 'text' | 'json';
}

export interface LLMResponse {
  content: string;
  usage: { inputTokens: number; outputTokens: number };
  model: string;
}

export class LLMService {
  constructor(
    private providers: Map<string, LLMProvider>,
    private defaultProvider: string,
  );

  async complete(request: LLMRequest, provider?: string): Promise<LLMResponse>;
  loadPromptTemplate(name: string, variables: Record<string, string>): string;
}

// apps/api/src/modules/ai/providers/openai.provider.ts
export class OpenAIProvider implements LLMProvider {
  readonly name = 'openai';
  // Uses gpt-4o for brief generation, gpt-4o-mini for simpler tasks
}

// apps/api/src/modules/ai/providers/anthropic.provider.ts
export class AnthropicProvider implements LLMProvider {
  readonly name = 'anthropic';
  // Uses claude-sonnet-4-20250514 for brief generation
}
```

**Testing**:
- `Unit: loadPromptTemplate substitutes variables correctly`
- `Unit: loadPromptTemplate with missing variable → error`
- `Unit: LLMService routes to correct provider by name`
- `Unit: LLMService falls back to default provider`
- `Integration (mocked): OpenAI complete → returns parsed response`
- `Integration (mocked): Anthropic complete → returns parsed response`
- `Unit: LLMResponse includes usage metrics for cost tracking`

#### 7.2 — Content Brief Generation

**What**: Implement AI content brief generation from a topic input. The brief includes target persona, keywords with intent, recommended structure, tone guidelines, internal linking map, and WCAG 2.2 accessibility guidance.

**Design**:

```typescript
// packages/shared/src/types/content.ts
export interface ContentBrief {
  topic: string;
  targetPersona: string;
  targetKeywords: string[];
  keywordIntent: 'informational' | 'navigational' | 'transactional' | 'commercial';
  recommendedStructure: string;
  toneGuidelines: string;
  wordCountTarget: number;
  internalLinkingMap: { url: string; anchor: string }[];
  competitorRefs: string[];
  accessibilityGuidance: {
    readingLevel: string;
    altTextRequired: boolean;
    headingStructure: string;
  };
  generatedBy: 'ai' | 'manual';
  modelVersion: string;
  generatedAt: string;
}
```

```typescript
// apps/api/src/modules/ai/brief.service.ts
export class BriefService {
  async generateBrief(workspaceId: string, contentPieceId: string, topic: string): Promise<ContentBrief>;
  async regenerateBrief(workspaceId: string, contentPieceId: string, feedback: string): Promise<ContentBrief>;
}
```

Prompt template (stored in `prompts/brief-generation.md`):
```markdown
# System
You are a content strategy expert. Generate a comprehensive content brief in JSON format.

# Context
Workspace brand voice: {{brand_voice}}
Previous content topics: {{recent_topics}}
Target audience personas: {{personas}}

# Task
Generate a content brief for the topic: "{{topic}}"
Content type: {{content_type}}

Return a JSON object with these fields:
- targetPersona: the primary persona this content targets
- targetKeywords: array of 5-10 keywords ranked by relevance
- keywordIntent: one of informational/navigational/transactional/commercial
- recommendedStructure: outline with H2/H3 headings
- toneGuidelines: specific tone direction for this piece
- wordCountTarget: recommended word count
- internalLinkingMap: array of {url, anchor} for internal links
- competitorRefs: array of competitor URLs covering similar topics
- accessibilityGuidance: reading level, alt-text requirements, heading structure
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces/:wid/content/:id/generate-brief` | Generate AI brief for content piece |
| POST | `/api/workspaces/:wid/content/:id/regenerate-brief` | Regenerate with feedback |

**Testing**:
- `Unit: generateBrief with topic "Q3 Product Launch" → brief with all required fields populated`
- `Unit: generateBrief includes workspace brand_voice in prompt`
- `Unit: generateBrief includes recent content topics for context`
- `Unit: generated brief passes JSON Schema validation against brief.schema.json`
- `Unit: brief stored in content_piece.brief JSONB column`
- `Unit: content piece status transitions to "briefed" after brief generation`
- `Unit: regenerateBrief with feedback "more technical tone" → updated brief reflects feedback`
- `Integration (mocked LLM): full brief generation flow → brief saved and content status updated`
- `Unit: LLM returns invalid JSON → error handled gracefully, retry with adjusted prompt`

#### 7.3 — Cross-Channel Content Repurposing

**What**: Given a source content piece, generate platform-native derivatives for selected channels (blog to carousel, tweet thread, email excerpt, LinkedIn article).

**Design**:

```typescript
// apps/api/src/modules/ai/repurpose.service.ts
export interface RepurposeRequest {
  sourceContentPieceId: string;
  targetPlatforms: Platform[];
}

export interface RepurposeResult {
  platform: Platform;
  suggestedContent: PlatformContent;
  preview: string;    // Human-readable preview text
  estimatedEngagement: number;
}

export class RepurposeService {
  async generateRepurposeSuggestions(
    workspaceId: string,
    request: RepurposeRequest,
  ): Promise<RepurposeResult[]>;

  async acceptSuggestion(
    workspaceId: string,
    contentPieceId: string,
    result: RepurposeResult,
  ): Promise<{ contentPiece: ContentPiece; scheduledPost: ScheduledPost }>;
  // Creates a new content piece (type: social_post) linked to source via metadata.repurposed_from
  // Creates a scheduled post draft with the generated platform content
}
```

Prompt template for repurposing (`prompts/content-repurpose.md`):
```markdown
# System
You are a cross-channel content adaptation expert. Transform content for specific platforms while maintaining the core message.

# Source Content
Title: {{title}}
Body: {{body}}
Content Type: {{content_type}}

# Target Platform: {{platform}}
Character limits: {{character_limits}}
Supported formats: {{capabilities}}
Best practices for {{platform}}: {{platform_guidelines}}

# Task
Adapt the source content for {{platform}}. Return JSON matching the {{platform}} content schema.
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces/:wid/content/:id/repurpose` | Generate repurposing suggestions |
| POST | `/api/workspaces/:wid/content/:id/repurpose/accept` | Accept a suggestion and create derivative content |

**Testing**:
- `Unit: repurpose blog_post to Instagram → carousel with slides, caption within 2200 chars`
- `Unit: repurpose blog_post to Twitter → thread with tweets, each within 280 chars`
- `Unit: repurpose blog_post to LinkedIn → article with professional tone`
- `Unit: repurpose blog_post to Email → subject line + preview text + HTML body`
- `Unit: acceptSuggestion → new content piece created with metadata.repurposed_from = source id`
- `Unit: generated platform content passes JSON Schema validation per platform`
- `Integration (mocked LLM): repurpose 2000-word blog → 4 platform suggestions returned`

#### 7.4 — Performance Prediction

**What**: Before publishing, predict estimated engagement (reach, engagement rate, clicks) based on historical performance data, content characteristics, and optimal posting time analysis.

**Design**:

```typescript
// apps/api/src/modules/ai/prediction.service.ts
export class PredictionService {
  async predictPerformance(
    workspaceId: string,
    contentPieceId: string,
  ): Promise<PerformancePredictionResult>;

  async calculateOptimalPostingTimes(
    workspaceId: string,
    channelId: string,
  ): Promise<OptimalPostingTime[]>;
}

export interface PerformancePredictionResult {
  overall: {
    estimatedTotalReach: number;
    estimatedEngagementRate: number;
    confidence: number;
  };
  byChannel: Record<string, {
    reach: number;
    engagementRate: number;
    bestTime: string;
  }>;
  modelVersion: string;
  factors: string[];
}

export interface OptimalPostingTime {
  channelId: string;
  dayOfWeek: number;     // 0-6, Sunday=0
  hourUtc: number;       // 0-23
  score: number;         // 0.0-1.0
  sampleSize: number;
}
```

Prediction factors:
1. Historical engagement data for this channel (from engagement_snapshots)
2. Content type and format (blog posts perform differently than carousels)
3. Day-of-week and time-of-day patterns from optimal_posting_times
4. Topic relevance (comparison with high-performing historical content)
5. Brief quality score (completeness of brief fields)

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces/:wid/content/:id/predict` | Predict performance |
| GET | `/api/workspaces/:wid/channels/:id/optimal-times` | Get optimal posting times |

**Testing**:
- `Unit: predictPerformance with sufficient historical data → predictions with confidence > 0.5`
- `Unit: predictPerformance with no historical data → predictions with confidence < 0.3, factors: ["insufficient_data"]`
- `Unit: predictions stored in performance_predictions table`
- `Unit: calculateOptimalPostingTimes from 100+ engagement snapshots → 7x24 grid of scores`
- `Unit: calculateOptimalPostingTimes with < 10 snapshots → empty results, warning returned`
- `Integration (mocked LLM): prediction for blog_post with 6 months of history → realistic reach estimates`

---

## Phase 8: Analytics & Engagement Tracking

### Purpose
Collect post-publish engagement metrics from social platforms, build analytics dashboards, and provide performance comparison views. After this phase, users can see how their published content performs, compare performance across channels, and view engagement trends over time.

### Tasks

#### 8.1 — Metrics Collection Worker

**What**: Implement a BullMQ worker that periodically collects engagement metrics from social platform APIs for published posts.

**Design**:

```typescript
// apps/api/src/workers/metrics.worker.ts
// Collection schedule:
// - First 48 hours after publishing: every 4 hours
// - Days 3-7: every 12 hours
// - Days 8-30: daily
// - After 30 days: weekly for 3 months, then stop

// Worker flow:
// 1. Query published posts needing metric collection based on schedule
// 2. For each post, call SocialPublisher.getMetrics(platformPostId, connection)
// 3. Create new EngagementSnapshot row (time-series, not overwrite)
// 4. Update PerformancePrediction accuracy scores (compare predicted vs actual)
```

**Testing**:
- `Unit: collection scheduler identifies posts needing metrics based on publish age`
- `Unit: 24-hour-old post → collected every 4 hours`
- `Unit: 10-day-old post → collected daily`
- `Unit: 60-day-old post → collected weekly`
- `Unit: new snapshot created per collection (not overwriting previous)`
- `Integration (mocked API): collect Facebook metrics → snapshot with likes, comments, shares, reach`
- `Integration (mocked API): collect LinkedIn metrics → snapshot with demographics data in platform_metrics JSONB`
- `Unit: failed collection (API error) → logged, retried in next cycle`

#### 8.2 — Analytics Dashboard

**What**: Build analytics pages with engagement overview, per-channel performance, content performance ranking, and trend charts.

**Design**:

Analytics dashboard sections:
1. **Overview cards**: Total reach (30d), avg engagement rate, total posts published, top performing post
2. **Engagement trend chart**: Line chart showing impressions/reach/engagement over time, filterable by channel
3. **Channel comparison**: Bar chart comparing engagement rate, reach, and click-through across channels
4. **Content leaderboard**: Table ranking content pieces by total engagement, sortable by metric
5. **Best posting times heatmap**: 7x24 grid showing engagement score by day/hour per channel

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/workspaces/:wid/analytics/overview` | Summary metrics (30d) |
| GET | `/api/workspaces/:wid/analytics/trends` | Time-series engagement data |
| GET | `/api/workspaces/:wid/analytics/channels` | Per-channel comparison |
| GET | `/api/workspaces/:wid/analytics/leaderboard` | Top content pieces |
| GET | `/api/workspaces/:wid/analytics/posting-times/:channelId` | Optimal time heatmap data |

**Testing**:
- `E2E: analytics overview shows total reach, engagement rate for last 30 days`
- `E2E: trend chart renders with correct data points`
- `E2E: channel comparison shows all active channels with metrics`
- `E2E: leaderboard sorts by engagement rate descending`
- `E2E: posting times heatmap shows coloured cells for available data`
- `Unit: overview API aggregates engagement_snapshots correctly`
- `Unit: trends API returns daily/weekly/monthly granularity based on date range`
- `Unit: channel comparison excludes channels with no published posts`

---

## Phase 9: Integrations & Webhooks

### Purpose
Connect to external tools (Zapier, HubSpot, Slack, Google Analytics) and implement outbound webhooks following CloudEvents 1.0 specification. After this phase, the platform interoperates with the broader marketing stack.

### Tasks

#### 9.1 — Webhook System (CloudEvents 1.0)

**What**: Implement outbound webhook subscription management and delivery with CloudEvents 1.0 envelope format, HMAC-SHA256 signing, and delivery retry logic.

**Design**:

```typescript
// packages/shared/src/types/events.ts
// CloudEvents 1.0 envelope
export interface CloudEvent<T = unknown> {
  specversion: '1.0';
  id: string;                  // UUID
  source: string;              // e.g., '/workspaces/{id}'
  type: string;                // e.g., 'com.contentcalendar.content.published'
  subject?: string;            // e.g., '/content/{id}'
  time: string;                // ISO 8601
  datacontenttype: 'application/json';
  data: T;
}

export const EVENT_TYPES = [
  'content.created',
  'content.updated',
  'content.status_changed',
  'content.published',
  'content.archived',
  'brief.generated',
  'brief.updated',
  'post.scheduled',
  'post.published',
  'post.failed',
  'approval.submitted',
  'approval.approved',
  'approval.rejected',
  'approval.revision_requested',
  'campaign.created',
  'campaign.activated',
  'campaign.completed',
  'metrics.collected',
] as const;
```

```typescript
// apps/api/src/modules/webhook/webhook.service.ts
export class WebhookService {
  async createSubscription(workspaceId: string, dto: CreateWebhookDto): Promise<Webhook>;
  async listSubscriptions(workspaceId: string): Promise<Webhook[]>;
  async deleteSubscription(workspaceId: string, webhookId: string): Promise<void>;
  async deliver(workspaceId: string, event: CloudEvent): Promise<void>;
  // Deliver to all active webhooks subscribed to this event type
  // HMAC-SHA256 signature in X-Signature-256 header
  // Retry: 3 attempts with exponential backoff (1s, 10s, 60s)
  // Store delivery result in webhook_deliveries table
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces/:wid/webhooks` | Create webhook subscription |
| GET | `/api/workspaces/:wid/webhooks` | List subscriptions |
| DELETE | `/api/workspaces/:wid/webhooks/:id` | Delete subscription |
| GET | `/api/workspaces/:wid/webhooks/:id/deliveries` | View delivery history |

**Testing**:
- `Unit: createSubscription with valid URL and event types → webhook created with HMAC secret`
- `Unit: deliver event → POST to webhook URL with CloudEvents headers`
- `Unit: delivery includes X-Signature-256 header with HMAC-SHA256 of body`
- `Unit: delivery with HTTP 200 response → recorded as successful`
- `Unit: delivery with HTTP 500 → retried 3 times with backoff`
- `Unit: delivery after 3 failures → marked as failed, no more retries`
- `Unit: event "content.published" → only webhooks subscribed to that type receive delivery`
- `Integration: publish content piece → CloudEvent emitted → webhook delivered`

#### 9.2 — Slack Integration

**What**: Implement Slack notifications for configurable events (content approved, post published, post failed).

**Design**:

```typescript
// apps/api/src/modules/integration/slack.service.ts
export class SlackService {
  async sendNotification(webhookUrl: string, event: CloudEvent): Promise<void>;
  // Formats CloudEvent into Slack Block Kit message
  // Different formats per event type:
  //   content.published → "✓ Published: {title} on {channel} - {url}"
  //   approval.revision_requested → "⟲ Revision requested: {title} by {reviewer}: {comment}"
  //   post.failed → "✗ Publishing failed: {title} on {channel}: {error}"
}
```

**Testing**:
- `Unit: content.published event → Slack block with title, channel, and link`
- `Unit: post.failed event → Slack block with error details and retry info`
- `Integration (mocked Slack): notification delivery → correct Block Kit payload sent`

#### 9.3 — Zapier / Generic Webhook Integration

**What**: Implement Zapier-compatible trigger/action webhooks and a REST Hooks subscription API.

**Design**:

Zapier integration uses the webhook system (9.1) as the trigger mechanism. Zapier actions (creating content, scheduling posts) use the existing REST API with API key authentication.

API key management:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/workspaces/:wid/api-keys` | Create API key |
| GET | `/api/workspaces/:wid/api-keys` | List API keys (masked) |
| DELETE | `/api/workspaces/:wid/api-keys/:id` | Revoke API key |

**Testing**:
- `Unit: create API key → hashed key stored, plain key returned once`
- `Unit: API request with valid API key → authenticated as the key's creator`
- `Unit: API request with revoked key → 401 Unauthorized`
- `Unit: Zapier subscribe endpoint registers webhook → trigger fires on matching events`

---

## Phase 10: Activity Feed & Audit Trail

### Purpose
Implement comprehensive activity logging aligned with W3C Activity Streams 2.0 vocabulary, providing a workspace-wide feed of all actions and a per-resource audit trail. After this phase, users can see who did what and when across all workspace content.

### Tasks

#### 10.1 — Activity Logging Service

**What**: Instrument all write operations to emit activity log entries with actor/verb/object/changes structure.

**Design**:

```typescript
// apps/api/src/modules/activity/activity.service.ts
export class ActivityService {
  async log(entry: CreateActivityDto): Promise<void>;
  async getWorkspaceFeed(workspaceId: string, filters: ActivityFilters): Promise<PaginatedResult<ActivityEntry>>;
  async getResourceHistory(resourceType: string, resourceId: string): Promise<ActivityEntry[]>;
}

interface CreateActivityDto {
  workspaceId: string;
  actorId: string;
  actorType: 'user' | 'system' | 'ai';
  action: string;           // W3C Activity Streams verb: 'Create', 'Update', 'Approve', 'Publish'
  resourceType: string;     // 'content_piece', 'scheduled_post', 'campaign', etc.
  resourceId: string;
  changes?: Record<string, { from: unknown; to: unknown }>;
}

// Use NestJS interceptor to automatically log activity for all write endpoints:
@Injectable()
export class ActivityInterceptor implements NestInterceptor {
  // Captures before/after state for PATCH operations
  // Logs Create/Update/Delete actions automatically
}
```

**Testing**:
- `Unit: create content piece → activity entry with action "Create" and resource_type "content_piece"`
- `Unit: update content title → activity entry with changes: { title: { from: "old", to: "new" } }`
- `Unit: approval decision → activity entry with action "Approve" and decision details`
- `Unit: AI brief generation → activity entry with actorType "ai"`
- `Unit: workspace feed returns entries in reverse chronological order`
- `Unit: resource history for specific content piece → all actions on that piece`
- `Integration: full content lifecycle → activity feed shows complete trail`

#### 10.2 — Activity Feed UI

**What**: Build workspace activity feed page and per-resource activity timeline component.

**Design**:

Workspace activity feed: chronological feed with entries showing avatar, actor name, action verb, resource link, and timestamp. Filterable by actor, action type, and resource type. Real-time updates via WebSocket or polling.

Resource activity timeline: embedded in content piece detail view, showing all actions taken on that piece (created, brief generated, status changed, approved, published).

**Testing**:
- `E2E: activity feed shows recent actions across workspace`
- `E2E: filter by action "Approve" → only approval entries shown`
- `E2E: content piece detail → activity timeline shows full history`
- `E2E: new activity appears without page refresh (polling or WebSocket)`

---

## Phase 11: Mobile Responsiveness & Polish

### Purpose
Ensure the entire web application is fully responsive for mobile use (on-the-go content review and approval) and add UX polish including keyboard shortcuts, loading states, error handling, and accessibility compliance (WCAG 2.2).

### Tasks

#### 11.1 — Responsive Layout & Mobile Optimization

**What**: Adapt all pages for mobile viewports (320px-768px). Calendar switches to list view on mobile. Approval actions available via swipe gestures.

**Design**:

Responsive breakpoints:
- Mobile: 320px-767px (single column, bottom nav, list views)
- Tablet: 768px-1023px (collapsible sidebar, compact calendar)
- Desktop: 1024px+ (full sidebar, calendar grid)

Mobile-specific adaptations:
- Calendar: month view shows dot indicators per day, tapping a day shows list of entries
- Content list: card-based layout replacing table
- Approval: swipe-right to approve, swipe-left to reject (with confirmation)
- Navigation: bottom tab bar replacing sidebar

**Testing**:
- `E2E: calendar on 375px viewport → list view with day headers`
- `E2E: content list on 375px viewport → card layout, not table`
- `E2E: approval swipe gesture → decision recorded`
- `E2E: sidebar collapsed on tablet → hamburger menu expands it`
- `E2E: all interactive elements have minimum 44x44px touch targets (WCAG 2.2)`

#### 11.2 — Accessibility Compliance (WCAG 2.2)

**What**: Audit and fix all pages for WCAG 2.2 Level AA compliance including keyboard navigation, ARIA labels, colour contrast, and screen reader support.

**Design**:

Accessibility requirements:
- All interactive elements keyboard-accessible (Tab, Enter, Escape)
- ARIA labels on all buttons, inputs, and dynamic content regions
- Colour contrast ratio minimum 4.5:1 for normal text, 3:1 for large text
- Calendar drag-and-drop has keyboard alternative (select + move commands)
- Form validation errors announced by screen reader
- Loading states announced by aria-live regions
- Alt text required for all uploaded images (enforced in asset upload flow)

**Testing**:
- `E2E: axe-core accessibility audit on all pages → no critical or serious violations`
- `E2E: calendar navigation via keyboard → Tab through entries, Enter to select, Arrow keys to move date`
- `E2E: approval form → screen reader announces current step and available actions`
- `E2E: colour contrast check on all text elements → passes 4.5:1 ratio`
- `E2E: image upload without alt text → warning prompt before save`

---

## Phase 12: Deployment, Security Hardening & Documentation

### Purpose
Prepare the application for production deployment with Docker images, security hardening per OWASP API Security Top 10, rate limiting, and API documentation. After this phase, the application is production-ready.

### Tasks

#### 12.1 — Docker Production Images

**What**: Build optimised multi-stage Docker images for API, web, and worker services.

**Design**:

```dockerfile
# Dockerfile.api
FROM node:22-alpine AS builder
WORKDIR /app
COPY pnpm-workspace.yaml pnpm-lock.yaml ./
COPY packages/shared ./packages/shared
COPY apps/api ./apps/api
RUN corepack enable && pnpm install --frozen-lockfile
RUN pnpm --filter @content-calendar/api build
RUN pnpm --filter @content-calendar/api deploy --prod /app/deploy

FROM node:22-alpine AS runner
WORKDIR /app
COPY --from=builder /app/deploy ./
EXPOSE 3001
CMD ["node", "dist/main.js"]
```

Docker Compose production configuration:
- PostgreSQL with volume persistence
- Redis with AOF persistence
- MinIO with TLS
- API server (2 replicas minimum)
- Worker process (separate container)
- Nginx reverse proxy with TLS termination
- Health check endpoints on all services

**Testing**:
- `Integration: docker build for all services → images build successfully`
- `Integration: docker compose up (production config) → all services healthy`
- `Integration: health check endpoints return 200`
- `Integration: API serves requests through nginx reverse proxy`

#### 12.2 — Security Hardening

**What**: Implement security measures per OWASP API Security Top 10 (2023) and OWASP ASVS 4.0.

**Design**:

Security measures:
1. **API01 (Broken Object Level Auth)**: Workspace-scoped guards on all endpoints; every query includes workspace_id filter
2. **API02 (Broken Auth)**: Rate limiting on auth endpoints (5 attempts/minute); JWT with short expiry (15min) + refresh tokens
3. **API03 (Broken Object Property Level Auth)**: DTOs strip unexpected fields; response serialisers exclude internal fields
4. **API04 (Unrestricted Resource Consumption)**: Request body size limits (10MB), pagination limits (max 100 items), file upload size limits per plan
5. **API05 (Broken Function Level Auth)**: Role-based guards on all admin/destructive endpoints
6. **API06 (Unrestricted Access to Sensitive Business Flows)**: Rate limiting on AI brief generation (10/hour free, 100/hour pro)
7. **API07 (Server Side Request Forgery)**: Webhook URLs validated against allowlist, no internal IP ranges
8. **API08 (Security Misconfiguration)**: Helmet.js headers, CORS configuration, no stack traces in production errors
9. **API09 (Improper Inventory Management)**: OpenAPI spec auto-generated, no undocumented endpoints
10. **API10 (Unsafe Consumption of APIs)**: Social platform API responses validated before storage

Additional:
- Encryption at rest for OAuth tokens (AES-256-GCM)
- CSRF protection on state-changing endpoints
- Content Security Policy headers
- SQL injection prevention via Prisma parameterised queries
- GDPR: data export endpoint, account deletion endpoint

```typescript
// Rate limiting configuration
export const RATE_LIMITS = {
  auth: { windowMs: 60_000, max: 5 },
  api: { windowMs: 60_000, max: 100 },
  aiGeneration: {
    free: { windowMs: 3_600_000, max: 10 },
    starter: { windowMs: 3_600_000, max: 50 },
    professional: { windowMs: 3_600_000, max: 100 },
    enterprise: { windowMs: 3_600_000, max: 500 },
  },
  webhookDelivery: { windowMs: 60_000, max: 1000 },
};
```

**Testing**:
- `Unit: request to /content/:id in wrong workspace → 404 (not 403, to prevent enumeration)`
- `Unit: 6th login attempt within 1 minute → 429 Too Many Requests`
- `Unit: request with body > 10MB → 413 Payload Too Large`
- `Unit: webhook URL pointing to internal IP (10.0.0.0/8) → 400 "Invalid webhook URL"`
- `Unit: API response does not include internal fields (password_hash, access_token)`
- `Unit: production error response does not include stack trace`
- `Unit: Helmet.js headers present in all responses`
- `Unit: CORS only allows configured origins`
- `Integration: OWASP ZAP scan on API → no high-severity findings`

#### 12.3 — API Documentation & OpenAPI Spec

**What**: Generate comprehensive OpenAPI 3.1 specification from NestJS decorators and publish interactive Swagger UI.

**Design**:

OpenAPI configuration:
- Auto-generated from NestJS Swagger decorators
- Published at `/api/docs` (Swagger UI) and `/api/docs/json` (spec file)
- Includes authentication schemas (Bearer JWT, API Key)
- Request/response examples for all endpoints
- Error response schemas (400, 401, 403, 404, 429, 500)

**Testing**:
- `Unit: /api/docs returns Swagger UI HTML`
- `Unit: /api/docs/json returns valid OpenAPI 3.1 JSON`
- `Unit: all endpoints have request and response schemas documented`
- `Unit: OpenAPI spec validates against openapi-schema-validator`
- `Integration: generated TypeScript client from OpenAPI spec compiles without errors`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Project Scaffolding     ─── required by everything
    │
    ├── Phase 2: Content Core & Calendar Foundation ─── requires Phase 1
    │       │
    │       ├── Phase 3: Frontend Calendar & Content Management ─── requires Phase 2
    │       │       │
    │       │       └── Phase 11: Mobile Responsiveness & Polish ─── requires Phase 3
    │       │
    │       ├── Phase 4: Approval Workflows ─── requires Phase 2
    │       │
    │       └── Phase 5: Asset Management & File Storage ─── requires Phase 2
    │
    ├── Phase 6: Social Platform Publishing ─── requires Phase 2 + Phase 5
    │       │
    │       └── Phase 8: Analytics & Engagement Tracking ─── requires Phase 6
    │
    ├── Phase 7: AI Brief Generation & Content Intelligence ─── requires Phase 2
    │
    ├── Phase 9: Integrations & Webhooks ─── requires Phase 2
    │
    ├── Phase 10: Activity Feed & Audit Trail ─── requires Phase 2
    │
    └── Phase 12: Deployment, Security Hardening & Documentation ─── requires all phases
```

**Parallelism opportunities:**
- Phases 3, 4, 5, 7, 9, and 10 can all be developed concurrently after Phase 2 is complete
- Phase 6 can begin after Phase 2 but needs Phase 5 for asset support in social posts
- Phase 8 requires Phase 6 (needs published posts to collect metrics from)
- Phase 11 requires Phase 3 (needs the frontend to optimise)
- Phase 12 should be the final phase, after all feature work is complete

---

## Definition of Done (per phase)

- [ ] All tasks implemented per the design specification
- [ ] All unit tests pass (`pnpm test`)
- [ ] All integration tests pass (`pnpm test:integration`)
- [ ] ESLint passes with zero errors (`pnpm lint`)
- [ ] Prettier formatting applied (`pnpm format:check`)
- [ ] TypeScript strict mode compilation passes (`pnpm typecheck`)
- [ ] Docker build succeeds for all affected services (`docker compose build`)
- [ ] Feature works end-to-end in local Docker environment
- [ ] New API endpoints appear in auto-generated OpenAPI spec
- [ ] Database migrations created and tested (up and down)
- [ ] New configuration options documented in `.env.example`
- [ ] No secrets committed to source control
- [ ] Activity log entries emitted for all write operations (Phase 10+)
- [ ] WCAG 2.2 Level AA compliance for all new UI (Phase 11+)
