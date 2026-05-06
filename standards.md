# Standards & API Reference

> Project: Content Calendar & Strategy · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO 8601:2019 — Date and time format**
  - URL: https://www.iso.org/iso-8601-date-and-time-format.html
  - Relevance: Canonical format for representing scheduled publish dates, timezones, and recurrence windows in any content calendar API. Essential for cross-timezone publishing.

- **ISO/IEC 27001:2022 — Information Security Management Systems**
  - URL: https://www.iso.org/standard/27001
  - Relevance: Required for SaaS platforms storing brand voice data, drafts, and customer engagement data; most enterprise content marketing buyers require ISO 27001 attestation.

- **ISO/IEC 27701:2019 — Privacy Information Management**
  - URL: https://www.iso.org/standard/71670.html
  - Relevance: Extends 27001 with PII handling controls relevant when calendars store contact lists for email/newsletter distribution.

- **ISO 30415:2021 — Diversity and Inclusion (Human Resource Management)**
  - URL: https://www.iso.org/standard/71164.html
  - Relevance: Informs inclusive content guidelines that AI brief generators should reference when producing copy and imagery suggestions.

- **ISO 19005 (PDF/A) — Document Archiving**
  - URL: https://www.iso.org/standard/71670.html
  - Relevance: Long-form content briefs and exported editorial calendars often need PDF/A archival format for compliance retention.

### W3C & IETF Standards

- **RFC 5545 — iCalendar (Internet Calendaring and Scheduling Core Object Specification)**
  - URL: https://datatracker.ietf.org/doc/html/rfc5545
  - Relevance: De-facto standard for representing calendar events; content calendar tools must export/import iCalendar to integrate with Google Calendar, Outlook, and team calendars.

- **RFC 7986 — New Properties for iCalendar**
  - URL: https://datatracker.ietf.org/doc/html/rfc7986
  - Relevance: Extends iCalendar with properties (NAME, DESCRIPTION, COLOR, IMAGE) useful for richer content calendar metadata.

- **RFC 6321 — xCal (XML Format for iCalendar)** and **RFC 7265 — jCal (JSON Format)**
  - URLs: https://datatracker.ietf.org/doc/html/rfc6321 ; https://datatracker.ietf.org/doc/html/rfc7265
  - Relevance: JSON/XML serialisations of iCalendar useful for modern web-API exposure of editorial calendars.

- **RFC 4791 — CalDAV (Calendaring Extensions to WebDAV)**
  - URL: https://datatracker.ietf.org/doc/html/rfc4791
  - Relevance: Protocol for synchronising calendars across clients; relevant for two-way sync with team productivity calendars.

- **RFC 7231 / RFC 9110 — HTTP Semantics**
  - URL: https://datatracker.ietf.org/doc/html/rfc9110
  - Relevance: Foundational for any REST API the content platform exposes.

- **RFC 8288 — Web Linking**
  - URL: https://datatracker.ietf.org/doc/html/rfc8288
  - Relevance: Canonical form for representing relationships between content pieces (campaign → asset → derivatives).

- **W3C Activity Streams 2.0**
  - URL: https://www.w3.org/TR/activitystreams-core/
  - Relevance: Standard vocabulary for describing social activities; useful for representing scheduled publishes, approvals, and engagement events.

- **W3C WCAG 2.2 — Web Content Accessibility Guidelines**
  - URL: https://www.w3.org/TR/WCAG22/
  - Relevance: Brief generation should produce alt-text, reading-level guidance, and accessible structure recommendations.

- **schema.org Article / CreativeWork / Event vocabularies**
  - URL: https://schema.org/CreativeWork
  - Relevance: Standard structured-data vocabulary for tagging planned content with type, audience, and intent metadata.

### Data Model & API Specifications

- **OpenAPI Specification 3.1**
  - URL: https://spec.openapis.org/oas/v3.1.0
  - Relevance: Industry standard for documenting REST APIs; content platforms should publish OpenAPI for calendar, brief, asset, and publishing endpoints.

- **GraphQL Specification (October 2021)**
  - URL: https://spec.graphql.org/October2021/
  - Relevance: Used by several content platforms (Contentful, Sanity) for flexible content modelling queries.

- **JSON Schema 2020-12**
  - URL: https://json-schema.org/specification.html
  - Relevance: Define content brief schemas, persona schemas, and validation contracts for AI-generated outputs.

- **AsyncAPI 3.0**
  - URL: https://www.asyncapi.com/docs/reference/specification/v3.0.0
  - Relevance: Specification for documenting event-driven APIs (publish events, approval events, engagement webhooks).

- **JSON:API 1.1**
  - URL: https://jsonapi.org/
  - Relevance: Common conventions for REST resource representations; used by some content management vendors.

- **CloudEvents 1.0 (CNCF)**
  - URL: https://github.com/cloudevents/spec
  - Relevance: Standard envelope for webhook events emitted by the platform (e.g., `content.published`, `brief.approved`).

- **CMIS 1.1 (Content Management Interoperability Services, OASIS)**
  - URL: https://www.oasis-open.org/standard/cmis-v1-1/
  - Relevance: Legacy but still-used standard for interoperating with enterprise content repositories.

### Security & Authentication Standards

- **OAuth 2.0 (RFC 6749) and OAuth 2.1 draft**
  - URL: https://datatracker.ietf.org/doc/html/rfc6749
  - Relevance: Required for connecting to social platforms (Meta Graph, LinkedIn, X, TikTok), Google Calendar, and CMS systems.

- **OpenID Connect Core 1.0**
  - URL: https://openid.net/specs/openid-connect-core-1_0.html
  - Relevance: SSO for enterprise content teams; commonly federated via Okta, Azure AD, Google Workspace.

- **SAML 2.0**
  - URL: https://docs.oasis-open.org/security/saml/v2.0/
  - Relevance: Enterprise SSO standard required by mid-market and enterprise buyers.

- **SCIM 2.0 (RFC 7643/7644)**
  - URL: https://datatracker.ietf.org/doc/html/rfc7644
  - Relevance: User and team provisioning standard required for enterprise IT departments.

- **OWASP ASVS 4.0 / OWASP API Security Top 10 (2023)**
  - URL: https://owasp.org/www-project-api-security/
  - Relevance: Baseline security verification for any externally exposed content/calendar API.

- **NIST SP 800-53 Rev 5**
  - URL: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
  - Relevance: Required control set for U.S. public-sector and FedRAMP-aligned customers.

- **GDPR (EU 2016/679) and CCPA/CPRA**
  - URLs: https://eur-lex.europa.eu/eli/reg/2016/679/oj ; https://oag.ca.gov/privacy/ccpa
  - Relevance: Govern handling of audience/persona data and consent for distribution lists used by the calendar.

### MCP Server Specifications

- **Model Context Protocol (MCP) Specification**
  - URL: https://modelcontextprotocol.io/specification
  - Relevance: Highly relevant — an MCP server exposing calendar, brief, asset, and analytics tools would let any MCP-capable agent (Claude, Cursor, Windsurf) plan, generate, and schedule content. Recommended primitives: `list_scheduled_posts`, `create_brief`, `repurpose_asset`, `get_engagement_metrics`.

- **MCP Reference Servers (modelcontextprotocol/servers)**
  - URL: https://github.com/modelcontextprotocol/servers
  - Relevance: Reference implementations (Slack, Google Drive, Notion) demonstrate patterns directly applicable to a content-calendar MCP.

## Similar Products — Developer Documentation & APIs

### Hootsuite
- **Description:** Enterprise social media management platform with publishing, analytics, and AI-assisted content (OwlyWriter).
- **API Documentation:** https://developer.hootsuite.com/
- **SDKs/Libraries:** Official REST API; community Node.js and Python wrappers.
- **Developer Guide:** https://developer.hootsuite.com/docs/getting-started
- **Standards:** REST/JSON, OpenAPI documentation, webhooks.
- **Authentication:** OAuth 2.0 (Authorization Code).

### Sprout Social
- **Description:** Social management and analytics platform with content scheduling and listening.
- **API Documentation:** https://developers.sproutsocial.com/
- **SDKs/Libraries:** REST API; no first-party SDKs.
- **Developer Guide:** https://developers.sproutsocial.com/docs
- **Standards:** REST/JSON.
- **Authentication:** Bearer token / OAuth 2.0.

### Buffer
- **Description:** Lightweight social scheduling tool for SMBs and creators.
- **API Documentation:** https://buffer.com/developers/api
- **SDKs/Libraries:** REST API; community PHP/Python/Node.js libraries.
- **Developer Guide:** https://buffer.com/developers/api/oauth
- **Standards:** REST/JSON.
- **Authentication:** OAuth 2.0.

### CoSchedule
- **Description:** Marketing calendar and headline analyzer platform.
- **API Documentation:** https://api.coschedule.com/ (partner-gated)
- **SDKs/Libraries:** Limited public SDK availability; integrations via Zapier.
- **Developer Guide:** https://help.coschedule.com/hc/en-us
- **Standards:** REST/JSON.
- **Authentication:** API key.

### Contentful (CMS adjacent)
- **Description:** Headless CMS used as the content repository feeding many editorial calendars.
- **API Documentation:** https://www.contentful.com/developers/docs/references/
- **SDKs/Libraries:** Official SDKs for JavaScript/TypeScript, Python, Ruby, PHP, .NET, Java, Swift, Android.
- **Developer Guide:** https://www.contentful.com/developers/docs/
- **Standards:** REST + GraphQL, OpenAPI, webhooks (CloudEvents-style).
- **Authentication:** OAuth 2.0, Personal Access Tokens.

### Sanity.io (CMS adjacent)
- **Description:** Real-time content platform with structured content and live collaboration.
- **API Documentation:** https://www.sanity.io/docs/http-api
- **SDKs/Libraries:** Official JavaScript/TypeScript, Python, PHP SDKs.
- **Developer Guide:** https://www.sanity.io/docs
- **Standards:** GROQ query language, GraphQL, REST/JSON.
- **Authentication:** Bearer tokens, OAuth 2.0.

### Notion
- **Description:** Workspace platform commonly used for editorial calendars and content briefs.
- **API Documentation:** https://developers.notion.com/reference/intro
- **SDKs/Libraries:** Official JavaScript SDK; community Python (notion-client), Go, Ruby.
- **Developer Guide:** https://developers.notion.com/docs/getting-started
- **Standards:** REST/JSON, OpenAPI, webhooks.
- **Authentication:** OAuth 2.0, Internal Integration tokens.

### Airtable
- **Description:** Spreadsheet-database hybrid widely used as a content calendar backend.
- **API Documentation:** https://airtable.com/developers/web/api/introduction
- **SDKs/Libraries:** Official JavaScript SDK; community Python (pyairtable), Ruby.
- **Developer Guide:** https://airtable.com/developers/web/guides
- **Standards:** REST/JSON, webhooks.
- **Authentication:** OAuth 2.0, Personal Access Tokens.

### Meta Graph API (publishing target)
- **Description:** Underlying API for publishing to Facebook Pages and Instagram.
- **API Documentation:** https://developers.facebook.com/docs/graph-api/
- **SDKs/Libraries:** Official PHP and JavaScript Business SDKs; community Python.
- **Developer Guide:** https://developers.facebook.com/docs/development
- **Standards:** REST/JSON, GraphQL-inspired field selection.
- **Authentication:** OAuth 2.0 with long-lived Page tokens.

### LinkedIn Marketing API (publishing target)
- **Description:** API for publishing organic posts and managing LinkedIn Pages.
- **API Documentation:** https://learn.microsoft.com/en-us/linkedin/marketing/
- **SDKs/Libraries:** REST API; no broadly adopted official SDKs.
- **Developer Guide:** https://learn.microsoft.com/en-us/linkedin/marketing/quick-start
- **Standards:** REST/JSON, Restli Protocol 2.0.
- **Authentication:** OAuth 2.0 (3-legged).

### Google Calendar API (sync target)
- **Description:** Calendar service for two-way sync with team calendars.
- **API Documentation:** https://developers.google.com/calendar/api
- **SDKs/Libraries:** Official client libraries for Node.js, Python, Java, Go, Ruby, .NET, PHP.
- **Developer Guide:** https://developers.google.com/calendar/api/guides/overview
- **Standards:** REST/JSON, iCalendar import/export, Discovery Document.
- **Authentication:** OAuth 2.0.

### HubSpot Marketing Hub
- **Description:** Marketing platform with content calendar, blog, email, and social modules.
- **API Documentation:** https://developers.hubspot.com/docs/api/overview
- **SDKs/Libraries:** Official Node.js, Python, PHP, Ruby, .NET libraries.
- **Developer Guide:** https://developers.hubspot.com/docs/api/intro-to-auth
- **Standards:** REST/JSON, OpenAPI, webhooks.
- **Authentication:** OAuth 2.0, Private App tokens.

## Notes

- **Emerging:** MCP is the most significant near-term standard for this category — exposing calendar, brief, and analytics primitives via MCP enables agentic workflows that no incumbent currently supports natively.
- **Gap:** No widely adopted open standard exists for "content brief" interchange. JSON Schema-based community schemas (e.g., used by Storyblok, Contentful) are the de-facto approach. An open brief schema could be a meaningful contribution.
- **Gap:** Cross-platform "scheduled post" representation is fragmented — each social platform has its own model. iCalendar + Activity Streams + schema.org CreativeWork together provide a workable composite, but no single normative spec exists.
- **Caution:** Social platform APIs (X/Twitter especially) have changed rate limits, pricing, and access tiers significantly in 2023–2025 and remain volatile. Architect for adapter swap-out.
