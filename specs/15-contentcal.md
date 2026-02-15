# ContentCal — AI Social Media Content Planner

## Executive Summary

ContentCal generates a full month of social media content tailored to your business using AI. Describe your business and goals, and it produces posts with copy, hashtags, and images for every platform. Edit, approve, schedule, and auto-publish — turning social media from a constant chore into a 30-minute monthly task.

---

## Problem Statement

**The pain:**
- Small businesses know they need to post on social media but can't afford a marketing team
- Coming up with ideas, writing copy, and designing graphics for 3-5 platforms is overwhelming
- Most give up after a few weeks of inconsistent posting
- Existing scheduling tools (Buffer, Hootsuite) help you schedule but don't help you create
- Hiring a social media manager costs $3-5K/month; agencies charge $1-3K/month
- AI tools generate individual posts but don't plan a coherent content strategy

---

## Target Users

### Primary Personas

**1. Emma — Small Business Owner (bakery)**
- Has Instagram and Facebook but posts once a month at best
- Knows social media drives foot traffic but can't dedicate time
- Needs: content ideas generated for her, simple editing, one-click scheduling

**2. Raj — SaaS Founder**
- Needs LinkedIn and Twitter presence to build authority
- Can write but hates the overhead of planning and scheduling
- Needs: thought leadership content generated from his expertise

**3. Olivia — Marketing Freelancer**
- Manages social media for 5 small business clients
- Spends 15+ hours/week creating and scheduling content
- Needs: bulk content generation per client, client approval workflow

---

## Core Features

### F1: Business Profile & Content Strategy
- Onboarding wizard:
  - Business name, industry, target audience
  - Brand voice: professional, casual, witty, inspirational, educational
  - Key topics and expertise areas
  - Content goals: brand awareness, lead generation, community, sales
  - Competitor accounts to learn from
  - Platforms: Twitter/X, LinkedIn, Instagram, Facebook, TikTok
- AI generates a content strategy:
  - Content pillars (3-5 recurring themes)
  - Posting frequency recommendation per platform
  - Best times to post (based on industry benchmarks)
  - Content mix (educational, entertaining, promotional, behind-the-scenes)

### F2: AI Content Generation
- Generate a full month of content in one click
- Per post:
  - Platform-optimized copy (different length/style for each platform)
  - Hashtag suggestions (trending + evergreen)
  - Emoji usage matching brand voice
  - Call-to-action suggestions
  - AI-generated image (DALL-E / Stable Diffusion)
  - Carousel slide suggestions for Instagram
  - Thread structure for Twitter
- Content types:
  - Tips and how-tos
  - Industry news commentary
  - Behind-the-scenes
  - Customer stories/testimonials
  - Product highlights
  - Polls and questions (engagement posts)
  - Holiday and seasonal content
  - User-generated content prompts
- Regenerate any individual post
- "More like this" to generate similar posts

### F3: Content Calendar
- Monthly calendar view with posts on their scheduled dates
- Drag-and-drop to reschedule
- Color-coded by platform
- Week view and list view alternatives
- Filter by platform, content pillar, status (draft/approved/scheduled/published)
- Gap detection: "You have no posts scheduled for Thursday"
- Bulk actions: approve all, reschedule, delete

### F4: Post Editor
- Rich text editor with platform-specific preview
- Side-by-side view: edit on left, preview on right (how it'll look on Twitter, IG, etc.)
- Image editor: crop, filter, text overlay, resize per platform
- Media library: upload images, access AI-generated images, stock photos (Unsplash integration)
- Carousel builder for Instagram
- Thread builder for Twitter/X
- Video thumbnail selector
- Link preview for posts with URLs
- Character count and platform limits indicator

### F5: Multi-Platform Publishing
- **Direct publishing**: Twitter/X, LinkedIn, Facebook Pages, Instagram Business
- **Auto-scheduling**: queue posts at optimal times
- **Platform adaptation**: automatically adjust copy length, hashtags, image sizes per platform
- **Cross-posting**: one post adapted to multiple platforms
- **Instagram specifics**: support for feed posts, Stories, Reels
- **TikTok**: caption and hashtag optimization (video uploaded manually)
- Publishing queue: see everything scheduled in order

### F6: Content Recycling
- Identify top-performing posts
- AI rephrases for re-sharing without feeling repetitive
- Evergreen content queue: posts that are always relevant, auto-reshared on schedule
- "This post from 6 months ago performed well — want to reshare?"

### F7: Engagement Analytics
- Per-post metrics: likes, comments, shares, clicks, impressions, reach
- Platform-level dashboard: follower growth, engagement rate, best-performing posts
- Content pillar performance: which topics resonate most
- Best day/time analysis
- Competitor benchmarking (v2)
- Monthly report generation (PDF)
- ROI attribution: link clicks to website conversions (with UTM tracking)

### F8: Client Management (Freelancer/Agency)
- Multiple brand profiles
- Client approval workflow: generate → send for approval → client approves/edits → schedule
- Client portal: view calendar, approve posts, leave comments
- White-label option
- Per-client billing tracking

### F9: Content Inspiration
- Trending topics in your industry (daily updates)
- Holiday and awareness day calendar
- Competitor post monitoring: see what similar accounts are posting
- Content idea generator: "Give me 10 post ideas about [topic]"
- Viral post templates: proven formats adapted to your brand

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS, React DnD (calendar), Radix UI |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) |
| AI Content | OpenAI GPT-4o (copy), DALL-E 3 (images) |
| Queue | BullMQ + Redis (scheduled publishing) |
| Social APIs | Twitter API v2, LinkedIn API, Meta Graph API, TikTok API |
| Auth | Clerk |
| Image Storage | Cloudflare R2 |
| Email | Resend |
| Hosting | Vercel |

### Data Model

```
Organization
├── id, name, slug, plan
│
├── Brand[]
│   ├── id, org_id, name, industry
│   ├── voice_description, target_audience
│   ├── content_pillars (text[])
│   ├── logo_url, brand_colors (JSON)
│   ├── posting_config (JSON: { platform: { frequency, best_times } })
│   │
│   ├── SocialAccount[]
│   │   ├── id, brand_id, platform (twitter/linkedin/instagram/facebook/tiktok)
│   │   ├── account_name, account_id, access_token (encrypted)
│   │   ├── follower_count, last_synced_at
│   │   └── status (connected/expired/error)
│   │
│   ├── Post[]
│   │   ├── id, brand_id, content_pillar
│   │   ├── status (draft/pending_approval/approved/scheduled/published/failed)
│   │   ├── scheduled_at, published_at
│   │   ├── is_ai_generated, ai_regeneration_count
│   │   │
│   │   ├── PostVariant[] (one per platform)
│   │   │   ├── id, post_id, platform
│   │   │   ├── text, hashtags (text[])
│   │   │   ├── media_urls (text[])
│   │   │   ├── link_url, cta
│   │   │   ├── thread_texts (text[], for Twitter threads)
│   │   │   ├── carousel_slides (JSON[], for Instagram)
│   │   │   └── external_post_id (after publishing)
│   │   │
│   │   └── PostMetrics[]
│   │       ├── id, post_variant_id, platform
│   │       ├── impressions, reach, likes, comments, shares, clicks
│   │       ├── engagement_rate
│   │       └── fetched_at
│   │
│   └── MediaLibrary[]
│       ├── id, brand_id, url, type (image/video)
│       ├── source (upload/ai_generated/unsplash)
│       ├── tags[], alt_text
│       └── created_at
│
└── ContentCalendar (virtual, generated from Post.scheduled_at)
```

---

## UI/UX — Key Screens

### 1. Onboarding Wizard
- Step-by-step: business info → brand voice → platforms → topics → generate
- Preview of AI-generated content strategy before proceeding
- "Generate my first month" button

### 2. Content Calendar
- Monthly grid view, posts shown as cards on dates
- Color-coded by platform (blue=Twitter, purple=LinkedIn, pink=Instagram)
- Drag to reschedule
- Click to view/edit post
- "Generate more content" button for empty dates
- Status indicators: draft, approved, scheduled, published

### 3. Post Editor
- Left: text editor with formatting, hashtag suggestions, emoji picker
- Right: live preview as it would appear on each platform
- Image panel: upload, generate with AI, browse Unsplash
- Platform variant tabs (switch between Twitter/LinkedIn/Instagram versions)
- Schedule picker with best-time suggestions
- "Approve" and "Schedule" buttons

### 4. AI Generation Panel
- "Generate posts for next week" with options:
  - Content pillar selection
  - Number of posts
  - Platforms
  - Specific topic/angle
- Preview generated posts in a list
- Accept, regenerate, or edit each one
- Bulk add to calendar

### 5. Analytics Dashboard
- Platform-level metrics cards (followers, engagement rate, impressions)
- Best-performing posts chart
- Content pillar performance comparison
- Posting frequency vs engagement correlation
- Best day/time heatmap
- Growth trends over time

### 6. Client Portal (Agency)
- Simplified view for clients
- Calendar with pending approval posts
- Approve/reject buttons with comment
- No access to other clients' data
- Branded with agency logo

---

## Monetization

### Free Tier
- 5 posts per month
- 1 platform
- 1 brand
- AI generation (limited)
- Manual scheduling only
- ContentCal branding on shared images

### Pro — $19/month
- 30 posts per month
- 3 platforms
- AI content generation + AI images
- Auto-scheduling and publishing
- Analytics
- Content recycling
- No branding

### Business — $49/month
- Unlimited posts
- All platforms
- 5 brands
- Team collaboration (3 users)
- Client approval workflow
- Advanced analytics
- Priority support
- API access

### Agency — $99/month
- Unlimited brands
- Unlimited users
- White-label client portal
- Bulk generation
- Custom reporting
- Priority AI processing

---

## Go-to-Market Strategy

- Target small business owners on Instagram and TikTok (show the tool in action)
- Content: record the "generate a month of content in 2 minutes" flow as video content
- SEO: "social media content generator", "AI social media planner"
- Product Hunt launch
- Freelancer community targeting (freelancer forums, Fiverr/Upwork sellers)
- Free social media audit tool (analyze your account, suggest improvements)
- Affiliate program for marketing influencers

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Users | 2,000 | 10,000 |
| Posts generated/month | 30,000 | 200,000 |
| Posts published/month | 15,000 | 100,000 |
| Paying customers | 200 | 800 |
| MRR | $5,000 | $25,000 |
| AI acceptance rate (posts kept vs regenerated) | 60% | 75% |

---

## MVP Scope (v1.0)

### In Scope
- Business profile setup
- AI content generation (text only, 1 month batch)
- Content calendar with drag-and-drop
- Post editor with platform preview
- Twitter/X + LinkedIn publishing
- Basic scheduling
- Hashtag suggestions

### Out of Scope
- AI image generation
- Instagram/Facebook/TikTok publishing
- Analytics
- Content recycling
- Client management
- Carousel/thread builders
- Agency features

### MVP Timeline: 6-7 weeks
- Week 1: Data model, business profile, AI content strategy generation
- Week 2-3: AI post generation pipeline, calendar UI
- Week 4: Post editor, platform previews
- Week 5: Twitter + LinkedIn API integration, scheduling
- Week 6: Publishing queue, basic dashboard
- Week 7: Billing, onboarding polish, launch
