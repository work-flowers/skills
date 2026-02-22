---
name: content-drafting
description: Draft social media posts (primarily LinkedIn) and blog articles for work.flowers. Use when Dennis asks to write, draft, or create content for LinkedIn, social media, the blog, or Flow Statements. Searches Readwise highlights for relevant themes before drafting. Saves drafts to appropriate Notion databases.
---

# Content Drafting for work.flowers

## Destinations

Save content to these Notion data sources:

- **Social media posts** (LinkedIn): `14591b07-11ac-8187-934a-000b1a3a474d`
- **Blog posts**: `1d791b07-11ac-8146-9124-000b0d6dbcc8`

## Before Drafting

Search for related themes using the Readwise MCP server (`search_readwise_highlights`). When Readwise is unavailable, search this Notion data source containing synced highlights: `2d791b07-11ac-814b-af18-000b6272383c`

This surfaces relevant reading, quotes, and ideas to inform the draft.

## Writing Style

### Voice and Tone

Write as Dennis—a founder explaining something to a peer over coffee. Direct, conversational, and confident without being preachy. The reader should feel like they're getting useful insight from someone who's thought carefully about the topic.

### Structure

**Blog posts:**
- Lead with a problem or tension the reader recognises
- Use short paragraphs (2-4 sentences typical)
- Include section breaks (`---`) between major sections
- Concrete examples over abstract principles
- End with a soft CTA linking to discovery call or WhatsApp

**Social media posts:**
- Hook in the first line—something surprising, contrarian, or directly relevant to the audience
- Short paragraphs, often single sentences for emphasis
- Plain text only—no markdown, no links in body text (LinkedIn doesn't render them)
- Tag companies/people using LinkedIn's native @ tagging in the actual post
- End with a question or observation that invites engagement
- Include source links in the `sc_first_comment` property (posted as first comment)

### Language Patterns

**Use:**
- Active voice, first person where appropriate
- Punchy sentences alternating with longer ones for rhythm
- Specific numbers and concrete details
- Phrases like "Here's what caught my attention", "The irony is...", "In plain language..."
- Direct transitions: "So,", "But here's the thing:", "The problem is..."

**Avoid:**
- AI-generated contrast constructions: "This isn't just X—it's Y", "not just... but also..."
- Hollow superlatives: "game-changing", "revolutionary", "cutting-edge"
- Excessive hedging: "I think maybe", "it seems like perhaps"
- Marketing fluff: "leverage", "synergies", "best-in-class"
- Emojis in body text (acceptable in callouts if genuinely useful)
- Bullet points in running prose—use commas or numbered lists in natural language instead

### Formatting

**Blog posts:**
- Use `##` for main sections, `###` sparingly for subsections
- Callouts (`<callout>`) for tips, quotes, or important asides
- Images referenced with `<image source="...">` tags
- Code blocks with language specified when showing technical content
- End with CTA block: `[Book a discovery call]({{https://calendar.notion.so/meet/dennis-0l10rj1xnv/kb58d4lwv}})` and `[Message us on WhatsApp]({{https://wa.me/6580839785}})`

**Social media:**
- Plain text only—no markdown, bold, italics, or hyperlinks (LinkedIn renders none of it)
- Line breaks between paragraphs for readability
- Keep under ~1500 characters unless the topic genuinely requires more
- URLs go in `sc_first_comment`, not the main post body

## Example Patterns

See `references/examples.md` for annotated samples of both blog and social content showing these principles in action.
