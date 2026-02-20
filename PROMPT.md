**MINIONS CONTENT — CONTENT PUBLISHING**

You are tasked with building **minions-content**, a structured content publishing and lifecycle management system built on the Minions SDK. This is a structured approach to article creation, SEO optimization, and publishing workflows designed for content teams and AI agents.

---

**PROJECT OVERVIEW**

**minions-content** provides structured content creation with built-in SEO analysis, content calendar integration, and multi-channel publishing pipelines. Built on top of `minions-docs` for the document layer, it adds publishing-specific workflows, metadata management, and content graph analysis.

**Core value proposition:** Agents can autonomously create, optimize, schedule, and publish content while tracking cross-article relationships and SEO performance.

**Positioning:** Structured content management meets AI-native publishing workflows.

---

**MINIONS SDK REFERENCE — REQUIRED DEPENDENCY**

This project depends on `minions-sdk`, a published package that provides the foundational primitives. The GH Agent building this project MUST install it from the public registries and use the APIs documented below — do NOT reimplement minions primitives from scratch.

**Installation:**
```bash
# TypeScript (npm)
npm install minions-sdk
# or: pnpm add minions-sdk

# Python (PyPI) — package name is minions-sdk, but you import as "minions"
pip install minions-sdk
```

**TypeScript SDK — Core Imports:**
```typescript
import {
  // Core types
  type Minion, type MinionType, type Relation,
  type FieldDefinition, type FieldValidation, type FieldType,
  type CreateMinionInput, type UpdateMinionInput, type CreateRelationInput,
  type MinionStatus, type MinionPriority, type RelationType,
  type ExecutionResult, type Executable,
  type ValidationError, type ValidationResult,

  // Validation
  validateField, validateFields,

  // Built-in Schemas (10 MinionType instances — reuse where applicable)
  noteType, linkType, fileType, contactType,
  agentType, teamType, thoughtType, promptTemplateType, testCaseType, taskType,
  builtinTypes,

  // Registry — stores and retrieves MinionTypes by id or slug
  TypeRegistry,

  // Relations — in-memory directed graph with traversal utilities
  RelationGraph,

  // Lifecycle — CRUD operations with validation
  createMinion, updateMinion, softDelete, hardDelete, restoreMinion,

  // Evolution — migrate minions when schemas change (preserves removed fields in _legacy)
  migrateMinion,

  // Utilities
  generateId, now, SPEC_VERSION,
} from 'minions-sdk';
```

**Python SDK — Core Imports:**
```python
from minions import (
    # Types
    Minion, MinionType, Relation, FieldDefinition, FieldValidation,
    CreateMinionInput, UpdateMinionInput, CreateRelationInput,
    ExecutionResult, Executable, ValidationError, ValidationResult,
    # Validation
    validate_field, validate_fields,
    # Built-in Schemas (10 types)
    note_type, link_type, file_type, contact_type,
    agent_type, team_type, thought_type, prompt_template_type,
    test_case_type, task_type, builtin_types,
    # Registry
    TypeRegistry,
    # Relations
    RelationGraph,
    # Lifecycle
    create_minion, update_minion, soft_delete, hard_delete, restore_minion,
    # Evolution
    migrate_minion,
    # Utilities
    generate_id, now, SPEC_VERSION,
)
```

**Key Concepts:**
- A `MinionType` defines a schema (list of `FieldDefinition`s) — each field has `name`, `type`, `label`, `required`, `defaultValue`, `options`, `validation`
- A `Minion` is an instance with `id`, `title`, `minionTypeId`, `fields` (dict), `status`, `tags`, timestamps
- A `Relation` is a typed directional link (12 types: `parent_of`, `depends_on`, `implements`, `relates_to`, `inspired_by`, `triggers`, `references`, `blocks`, `alternative_to`, `part_of`, `follows`, `integration_link`)
- Field types: `string`, `number`, `boolean`, `date`, `select`, `multi-select`, `url`, `email`, `textarea`, `tags`, `json`, `array`
- `TypeRegistry` auto-loads 10 built-in types; register custom types with `registry.register(myType)`
- `createMinion(input, type)` validates fields against the schema and returns `{ minion, validation }` (TS) or `(minion, validation)` tuple (Python)
- Both SDKs serialize to identical camelCase JSON; Python provides `to_dict()` / `from_dict()` for conversion

**IMPORTANT:** Do NOT recreate these primitives. Import them from `minions-sdk` (npm) / `minions` (PyPI). Build your domain-specific types and utilities ON TOP of the SDK.

---

**CORE PRIMITIVES**

The system is built from these minion types:

### `article`
A publishable content piece.

**Fields:**
- `title` (string, required) — article title
- `slug` (string, required) — URL-friendly slug
- `excerpt` (textarea) — short summary or description
- `content` (textarea, required) — full article content (Markdown)
- `author` (string) — author name or ID
- `publishedAt` (date) — when published (null if unpublished)
- `updatedAt` (date) — last content update
- `status` (select: draft/scheduled/published/archived, required)
- `channel` (select: blog/newsletter/social/docs/other) — publishing channel
- `readingTimeMinutes` (number) — estimated reading time
- `wordCount` (number) — auto-calculated word count
- `featuredImageUrl` (url) — hero/featured image
- `canonicalUrl` (url) — canonical URL if syndicated

**Relations:**
- `implements` → `document` from `@minions-docs/sdk` (underlying structured document)
- `references` → `content-brief` (if created from a brief)
- `references` → `publication` (when published)
- `references` → other `article` minions (cross-references)
- `follows` → previous `article` version or `draft`

---

### `draft`
A work-in-progress version of an article.

**Fields:**
- `title` (string, required) — working title
- `content` (textarea, required) — draft content
- `author` (string) — draft author
- `version` (number) — draft iteration number
- `feedback` (textarea) — editorial feedback
- `status` (select: writing/review/revising/approved, required)
- `lastEditedAt` (date) — last edit timestamp

**Relations:**
- `follows` → previous `draft` (version chain)
- `references` → `content-brief` (if writing from a brief)
- `parent_of` → `revision` minions (track changes)

---

### `revision`
A tracked change or snapshot of a draft or article.

**Fields:**
- `summary` (textarea, required) — what changed
- `author` (string, required) — who made the change
- `snapshot` (json, required) — full content snapshot
- `createdAt` (date, required) — when revision was created

**Relations:**
- `part_of` → `draft` or `article`
- `follows` → previous `revision`

---

### `publication`
A published instance of an article on a specific channel.

**Fields:**
- `channel` (select: blog/newsletter/linkedin/medium/twitter/other, required) — where it was published
- `publishedUrl` (url, required) — live URL
- `publishedAt` (date, required) — publish timestamp
- `views` (number) — view count
- `clicks` (number) — click-through count
- `shares` (number) — social shares
- `engagementRate` (number) — calculated engagement metric

**Relations:**
- `references` → `article`

---

### `seo-metadata`
SEO-specific metadata and analysis for an article.

**Fields:**
- `metaTitle` (string, required) — SEO title tag
- `metaDescription` (textarea, required) — SEO meta description
- `focusKeyword` (string) — primary target keyword
- `keywords` (tags) — secondary keywords
- `ogTitle` (string) — Open Graph title
- `ogDescription` (textarea) — Open Graph description
- `ogImageUrl` (url) — Open Graph image
- `readabilityScore` (number) — Flesch reading ease score
- `keywordDensity` (json) — keyword frequency analysis
- `seoScore` (number) — overall SEO score (0-100)
- `recommendations` (array) — SEO improvement suggestions

**Relations:**
- `references` → `article`

---

### `content-brief`
A structured brief outlining what content should be created.

**Fields:**
- `title` (string, required) — brief title
- `objective` (textarea, required) — what this content should achieve
- `targetAudience` (string) — who this is for
- `primaryKeyword` (string) — main keyword to target
- `secondaryKeywords` (tags) — supporting keywords
- `targetWordCount` (number) — desired article length
- `tone` (select: professional/casual/technical/conversational/other) — writing tone
- `requiredSections` (array) — sections that must be included
- `referenceSources` (array) — URLs or articles to reference
- `competitorUrls` (array) — competitor articles to analyze
- `dueDate` (date) — when content is needed
- `status` (select: new/assigned/in-progress/completed, required)

**Relations:**
- `references` → `event` from `@minions-calendar/sdk` (scheduled deadline)
- `parent_of` → `article` or `draft` (content created from this brief)

---

**BEYOND THE STANDARD PATTERN**

### 1. Built on minions-docs

Content articles extend the document layer from `@minions-docs/sdk`, inheriting:
- Block-based content structure
- Revision tracking
- Multi-format rendering (Markdown, HTML, PDF)
- Collaborative commenting

**Additional publishing features:**
- SEO metadata layer
- Publishing pipeline stages
- Multi-channel publication tracking
- Content graph analysis

---

### 2. SEO Analysis

Automated SEO scoring and recommendations.

**Analysis dimensions:**
- **Meta tags:** Title and description length validation
- **Keyword optimization:** Focus keyword placement and density
- **Readability:** Flesch reading ease score
- **Content structure:** Headings, lists, and formatting
- **Internal linking:** Cross-references to other articles
- **Image optimization:** Alt tags and file size

**Implementation:**
```typescript
class SEOAnalyzer {
  analyze(article: Minion): SEOMetadata;
  validateMetaTags(metadata: SEOMetadata): ValidationResult[];
  calculateReadability(content: string): number;
  suggestImprovements(article: Minion): Recommendation[];
  keywordDensityReport(content: string, keywords: string[]): KeywordReport;
}
```

**CLI Integration:**
```bash
content seo-check <article-id>
content seo-report <article-id> --detailed
content seo-optimize <article-id> --auto-fix
```

---

### 3. Content Calendar Integration

Link content briefs and articles to calendar events from `@minions-calendar/sdk`.

**Features:**
- Schedule content deadlines
- Visualize content calendar by month
- Detect scheduling conflicts
- Track content velocity (articles published per week/month)

**Implementation:**
```typescript
class ContentCalendar {
  scheduleArticle(articleId: string, publishDate: Date): Event;
  getCalendar(month: string): CalendarView;
  detectConflicts(): Conflict[];
  contentVelocity(timeRange: DateRange): VelocityReport;
}
```

**CLI Integration:**
```bash
content calendar --month march
content schedule <article-id> --publish-date "2026-03-15"
content calendar conflicts
```

---

### 4. Content Graph

Track relationships between articles to build a knowledge graph.

**Relationship types:**
- `references` → articles link to other articles
- `follows` → series or multi-part content
- `alternative_to` → different takes on the same topic

**Graph analysis:**
- Orphaned articles (no incoming or outgoing links)
- Hub articles (highly referenced)
- Topic clusters (groups of related articles)
- Link depth (how many hops to reach from homepage)

**Implementation:**
```typescript
class ContentGraph {
  buildGraph(articles: Minion[]): Graph;
  findOrphans(): Minion[];
  findHubs(): Minion[];
  clusterByTopic(): Map<string, Minion[]>;
  calculateLinkDepth(articleId: string): number;
  suggestInternalLinks(article: Minion): Suggestion[];
}
```

**CLI Integration:**
```bash
content graph --output graph.json
content orphans
content hubs
content clusters --topic "AI agents"
content link-suggestions <article-id>
```

---

### 5. Publishing Pipeline

Configurable multi-stage publishing workflow.

**Default pipeline stages:**
1. **Brief** → content-brief created
2. **Draft** → draft written
3. **Review** → editorial review
4. **Revision** → changes made
5. **Approved** → ready to publish
6. **Scheduled** → publish date set
7. **Published** → live on channel(s)

**Pipeline tracking:**
- Time spent in each stage
- Bottleneck detection
- Reviewer assignment and workload

**Implementation:**
```typescript
class PublishingPipeline {
  getStage(article: Minion): PipelineStage;
  moveToStage(articleId: string, stage: string): void;
  stageAnalytics(): StageReport;
  assignReviewer(articleId: string, reviewerId: string): void;
  bottlenecks(): Bottleneck[];
}
```

**CLI Integration:**
```bash
content pipeline
content pipeline stage <article-id> --to review
content pipeline analytics
content pipeline bottlenecks
```

---

**DUAL SDK SUPPORT (TYPESCRIPT + PYTHON)**

Both SDKs provide identical functionality:

**TypeScript:**
```typescript
import { ArticleBuilder, SEOAnalyzer, ContentGraph } from '@minions-content/sdk';

const article = new ArticleBuilder()
  .withTitle('AI Agents in 2026')
  .withSlug('ai-agents-2026')
  .withContent('...')
  .build();

const seo = new SEOAnalyzer();
const metadata = seo.analyze(article);
const score = metadata.seoScore; // 0-100
```

**Python:**
```python
from minions_content import ArticleBuilder, SEOAnalyzer, ContentGraph

article = (ArticleBuilder()
    .with_title('AI Agents in 2026')
    .with_slug('ai-agents-2026')
    .with_content('...')
    .build())

seo = SEOAnalyzer()
metadata = seo.analyze(article)
score = metadata.seo_score  # 0-100
```

---

**CLI COMMANDS**

The `content` CLI extends the base `minions` CLI with publishing-specific commands.

### Core Commands

```bash
# Brief management
content brief new "AI Agents in 2026"
content brief assign <brief-id> --author alice@company.com
content brief list --status new

# Article creation
content new "Article Title" --from-brief <brief-id>
content draft new --from-brief <brief-id>
content draft show <draft-id>

# Revision tracking
content diff <rev1-id> <rev2-id>
content revisions <article-id>
content restore <revision-id>

# SEO analysis
content seo-check <article-id>
content seo-report <article-id> --detailed
content seo-optimize <article-id> --auto-fix

# Publishing
content publish <article-id> --channel blog
content publish <article-id> --channel blog --scheduled "2026-03-15 09:00"
content unpublish <article-id>
content publications <article-id>

# Content calendar
content calendar --month march
content calendar --year 2026
content schedule <article-id> --publish-date "2026-03-15"

# Content graph
content graph --output graph.json
content orphans
content hubs
content clusters --topic "AI"
content link-suggestions <article-id>

# Pipeline management
content pipeline
content pipeline stage <article-id> --to review
content pipeline analytics
content pipeline bottlenecks
content pipeline assign-reviewer <article-id> --reviewer bob@company.com

# Reporting
content report --published --month march
content report --velocity
content report --seo-overview
```

---

**DOCUMENTATION SITE**

Built with **Astro Starlight** with dual-language SDK examples.

**Site structure:**
```
docs/
├── index.md                   # Landing page
├── getting-started.md         # Quick start guide
├── core-concepts/
│   ├── articles.md
│   ├── drafts-and-revisions.md
│   ├── seo-metadata.md
│   ├── content-briefs.md
│   └── publications.md
├── guides/
│   ├── content-creation.md
│   ├── seo-optimization.md
│   ├── publishing-workflow.md
│   ├── content-calendar.md
│   ├── content-graph.md
│   └── multi-channel-publishing.md
├── api-reference/
│   ├── typescript.md          # TypeScript SDK
│   └── python.md              # Python SDK
├── cli-reference.md
└── examples/
    ├── seo-workflow.md
    ├── content-series.md
    └── agent-publishing.md
```

**Dual-language code tabs:**
Every code example includes both TypeScript and Python tabs.

---

**AGENT USE CASES**

**Automated Content Creation:**
An agent generates article drafts from content briefs, optimizing for SEO and readability.

**SEO Optimization:**
An agent analyzes existing articles, suggests improvements, and automatically fixes common SEO issues.

**Content Scheduling:**
An agent schedules articles based on content calendar availability and publishing velocity targets.

**Internal Linking:**
An agent analyzes the content graph, detects orphaned articles, and suggests internal links to improve SEO.

**Multi-Channel Publishing:**
An agent publishes the same article across multiple channels (blog, newsletter, LinkedIn) with channel-specific formatting.

**Content Reporting:**
An agent generates weekly content performance reports with SEO scores, traffic metrics, and publishing velocity.

---

**PROJECT STRUCTURE**

```
minions-content/
├── packages/
│   ├── core/
│   │   ├── src/
│   │   │   ├── types/              # Article, Draft, SEOMetadata, etc.
│   │   │   ├── builders/           # ArticleBuilder, BriefBuilder
│   │   │   ├── seo/                # SEOAnalyzer
│   │   │   ├── graph/              # ContentGraph
│   │   │   ├── calendar/           # ContentCalendar
│   │   │   ├── pipeline/           # PublishingPipeline
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── python/
│   │   ├── minions_content/
│   │   │   ├── __init__.py
│   │   │   ├── types.py
│   │   │   ├── builders.py
│   │   │   ├── seo.py
│   │   │   ├── graph.py
│   │   │   ├── calendar.py
│   │   │   └── pipeline.py
│   │   ├── tests/
│   │   └── pyproject.toml
│   └── cli/
│       ├── src/
│       │   ├── commands/
│       │   │   ├── brief.ts
│       │   │   ├── draft.ts
│       │   │   ├── publish.ts
│       │   │   ├── seo.ts
│       │   │   ├── calendar.ts
│       │   │   ├── graph.ts
│       │   │   ├── pipeline.ts
│       │   │   └── report.ts
│       │   └── index.ts
│       └── package.json
├── apps/
│   ├── docs/                       # Astro Starlight
│   └── playground/                 # Optional web UI
├── spec/
│   └── v0.1.md
├── examples/
│   ├── seo-workflow/
│   ├── content-series/
│   └── agent-publishing/
├── README.md
├── CHANGELOG.md
└── package.json
```

---

**INTEGRATION WITH MINIONS-DOCS**

**minions-content** builds on **minions-docs** by:

1. **Extending document types:** Articles are specialized documents with publishing metadata
2. **Reusing renderers:** Leverage `DocumentRenderer` for multi-format export
3. **Inheriting revisions:** Use the same revision tracking system
4. **Adding SEO layer:** Extend documents with SEO-specific analysis
5. **Publishing workflows:** Add publishing-specific pipeline stages

**Shared functionality:**
- Block-based content structure
- Markdown/HTML/PDF rendering
- Collaborative commenting
- Version control

**Content-specific additions:**
- SEO metadata and scoring
- Publishing channels and tracking
- Content calendar integration
- Content graph analysis

---

**TONE & POSITIONING**

**minions-content** is a structured content publishing system designed for content teams and AI-native workflows. The messaging should emphasize:

- **SEO-native:** Built-in SEO analysis and optimization from the start
- **Publishing workflows:** Configurable pipelines from brief to publication
- **Content intelligence:** Graph analysis reveals content gaps and opportunities
- **Multi-channel:** Publish once, distribute everywhere
- **Agent-assisted:** Agents can create, optimize, schedule, and publish autonomously

The docs should speak to content marketers, editors, and publishing teams building scalable content operations.

---

**SUCCESS CRITERIA**

You will know this implementation is successful when:

1. A content creator can draft, optimize for SEO, and publish an article in under 5 minutes
2. An agent can analyze a content brief, generate a draft, optimize SEO, and schedule publication autonomously
3. A content team can visualize their publishing calendar with deadlines and detect scheduling conflicts
4. The content graph reveals orphaned articles and suggests internal linking opportunities
5. Both TypeScript and Python SDKs provide identical functionality with idiomatic APIs
6. SEO analysis provides actionable recommendations that improve search rankings

---

**Start with the spec, then core types and builders, then SEO analyzer and content graph, then publishing pipeline, then the CLI, then the docs, then the Python SDK, then the examples. Work systematically. Every file should be production quality — not stubs, not placeholders.**
