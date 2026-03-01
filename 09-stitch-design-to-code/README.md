# Stitch MCP + Design-to-Code: AI-Generating 21 Screens in One Session

## The Premise

What if you could describe an entire application's visual design in plain text and have AI generate every screen, convert each to production React components, and validate the results automatically — all in a single working session?

That is exactly what happened across a 13,432-line Claude Code session using Stitch MCP powered by Gemini 3 Pro. Twenty-one screens went from text prompts to validated Next.js components without opening Figma, without dragging a single element, without writing a single line of CSS by hand. The tool did the designing. The agent did the coding. Puppeteer did the verification. A human did the steering.

This post breaks down how it worked, what went right, what went sideways, and what it means for design-to-code workflows going forward.

---

## Establishing the Design System

Every design sprint starts with constraints. Without a locked design system, generative models drift — producing inconsistent colors, mismatched typography, and layouts that fight each other across screens. The system was defined up front and embedded into every single prompt:

**Color Palette**
- Background: `#000000` (pure black)
- Primary Accent: `#e050b0` (hot pink)
- Secondary Accent: `#4dacde` (cyan)
- Surface/Card: `#111111` - `#1a1a1a` (dark grays)
- Text: `#ffffff` primary, `#a0a0a0` secondary

**Typography**: JetBrains Mono — monospaced across all elements, reinforcing the cyberpunk aesthetic while maintaining excellent readability at every size.

**Geometry**: 0px border radius everywhere. No rounded corners. Sharp, angular, deliberate. This single constraint gave the entire application a distinctive brutalist-cyberpunk identity that would be immediately recognizable.

**Component Library**: shadcn/ui as the base, ensuring that generated designs would map cleanly to an existing React component system with well-defined APIs.

This design system was not a suggestion — it was a contract. Every prompt to Stitch included the full specification. Consistency across 21 screens depended on repetition, not memory.

---

## The Stitch MCP Workflow

Stitch MCP exposes design generation as a tool call within Claude Code. The workflow for each screen followed a consistent pattern:

1. **Craft the prompt** — Describe the screen's purpose, layout, key elements, and interactions in natural language, always appending the full design system specification.

2. **Call Stitch** — Invoke `generate_screen_from_text` with the project ID, device type (DESKTOP), model (GEMINI_3_PRO), and the prompt.

3. **Review the output** — Stitch returns a screen design that can be inspected, iterated on, or accepted.

4. **Convert to code** — Translate the generated design into a Next.js React component using the shadcn/ui component library.

5. **Validate** — Run Puppeteer against the rendered component to verify structure, content, and visual correctness.

A representative call looked like this:

```json
{
  "name": "mcp__stitch__generate_screen_from_text",
  "input": {
    "projectId": "7956164041916734130",
    "deviceType": "DESKTOP",
    "modelId": "GEMINI_3_PRO",
    "prompt": "Design a Home Page for a curated video resource platform. Dark cyberpunk aesthetic: background #000000, primary accent Hot Pink #e050b0, secondary accent Cyan #4dacde, all text in JetBrains Mono, 0px border radius on all elements. Hero section with bold headline, featured resource cards in a grid, category navigation strip, trending section. Use shadcn/ui patterns."
  }
}
```

Gemini 3 Pro handled the visual generation. Stitch handled the MCP interface. Claude Code handled the orchestration, conversion, and validation. Each layer did one thing.

---

## Twenty-One Screens

The full application required 21 distinct screens, organized across five functional categories:

### Public Screens (7)
- **Home** — Hero section, featured resources grid, category navigation, trending strip
- **Resources** — Filterable grid with search, sort controls, pagination
- **Resource Detail** — Full resource view with metadata, related items, comments
- **Categories** — Category cards with resource counts, visual hierarchy
- **Category Detail** — Filtered resource list within a single category
- **Search** — Full-text search with filters, faceted results, live preview
- **About** — Platform mission, team, statistics, contribution CTA

### Authentication Screens (3)
- **Login** — Email/password form, social auth options, forgot password link
- **Register** — Multi-step registration with validation, terms acceptance
- **Forgot Password** — Email input, confirmation state, back-to-login flow

### User Screens (4)
- **Profile** — User info, contribution stats, activity history, settings
- **Bookmarks** — Saved resources grid with remove and organize actions
- **Favorites** — Starred resources with sorting and filtering
- **History** — Chronological browsing history with clear and search

### Learning Screens (3)
- **Learning Journeys** — Curated paths grid with progress indicators
- **Journey Detail** — Step-by-step learning path with completion tracking
- **Submit Resource** — Multi-field form for community resource submission

### Administrative & Legal (4)
- **Admin Dashboard** — 20-tab management interface covering users, resources, categories, reports, analytics, moderation, settings, and more
- **Suggest Edit** — Community edit proposal form with diff preview
- **Privacy Policy** — Legal content with structured sections and navigation
- **Terms of Service** — Legal content with table of contents and anchor links

Each screen went through the same generate-convert-validate cycle. The Admin Dashboard alone, with its 20 tabs, was the most complex single generation — requiring careful prompt engineering to ensure Stitch produced a coherent tabbed interface rather than 20 separate pages.

---

## React Conversion Strategy

Stitch generates designs, not code. The conversion from visual design to working Next.js components followed a disciplined pattern:

**Component Mapping**: Every visual element in the Stitch output mapped to a shadcn/ui component. Cards became `<Card>`. Buttons became `<Button>`. Inputs became `<Input>`. The 0px border radius was enforced globally through Tailwind configuration rather than per-component overrides.

**Layout Translation**: Grid layouts from the designs translated to CSS Grid and Flexbox via Tailwind classes. The cyberpunk aesthetic demanded precise spacing — `gap-4`, `p-6`, `space-y-8` — matching the visual density of the generated designs.

**Color Application**: The three-color system (`#000000`, `#e050b0`, `#4dacde`) mapped to CSS custom properties consumed by Tailwind. Hot pink for primary actions and emphasis. Cyan for secondary actions and informational elements. Pure black for backgrounds. This kept the palette locked regardless of which component was being styled.

**Typography Enforcement**: JetBrains Mono was set at the layout root. No component overrode the font family. This was non-negotiable — the monospaced aesthetic was the single strongest visual differentiator.

**A/B/C Variation Testing**: For critical screens (Home, Resources, Login), Stitch generated multiple design variations. These were compared side-by-side before selecting the strongest layout for conversion. This added time but produced measurably better results than accepting the first generation.

---

## Puppeteer Validation

Generating 21 screens means nothing if they do not render correctly. Puppeteer ran 107 distinct validation actions across the full screen set:

1. **Navigation** — Load each route, verify the page renders without errors, confirm the correct URL.
2. **Screenshot Capture** — Full-page screenshots at 1920x1080 for visual regression baseline.
3. **DOM Evaluation** — Query selectors to verify expected elements exist: headings, buttons, cards, forms, navigation items.
4. **Content Assertion** — Check that text content matches expectations: page titles, button labels, placeholder text.
5. **Interaction Testing** — Click buttons, fill forms, toggle tabs, scroll sections. Verify state changes propagate.

The validation loop ran as a sequence: navigate to page, capture screenshot, evaluate DOM structure, assert expected content, then move to the next page. Each screen averaged roughly 5 validation actions, though the Admin Dashboard with its 20 tabs required significantly more.

This automated validation caught several issues that would have been invisible in static design review:
- Missing hover states on interactive elements
- Form inputs without proper label associations
- Navigation links pointing to incorrect routes
- Z-index conflicts in overlapping card layouts

---

## The Branding Bug

No session this large escapes without at least one systemic error. This one's was a branding propagation bug.

Early in the session, the prompt for the Home screen included the phrase "Awesome Lists" as a placeholder application name. That name was supposed to be replaced with "Awesome Video Dashboard" — the actual product name. But the initial prompt's phrasing stuck. Stitch generated the Home screen with "Awesome Lists" in the hero section. When subsequent screens referenced the Home screen's design for consistency, "Awesome Lists" propagated silently across headers, footers, and navigation elements.

By the time the bug was caught, 8 screens carried the wrong product name. The fix was mechanical — find and replace across components — but the lesson was structural: **in generative design workflows, branding errors compound**. A typo in screen 1 becomes a systemic defect by screen 10. The design system contract must include copy and naming conventions with the same rigor as colors and typography.

This is a failure mode unique to AI-generated design. In manual design workflows, a designer notices the wrong name because they are reading every element they place. In generative workflows, the model faithfully reproduces whatever it was told, and the orchestrating agent does not second-guess the content unless explicitly instructed to verify branding.

---

## Session Economics

The full session spanned 13,432 lines of conversation — prompt, response, tool calls, validation output, and iteration. Some numbers worth noting:

- **21 screens** generated from text prompts
- **107 Puppeteer validation actions** executed
- **3 A/B/C variation rounds** for critical screens
- **1 systemic branding bug** caught and remediated
- **0 Figma files** opened
- **0 CSS files** written by hand

The entire design-to-validated-code pipeline ran inside a single Claude Code session. No context switching between tools. No exporting assets. No copying hex codes from a design tool into a stylesheet. The design system lived in the prompt. The designs lived in Stitch. The code lived in the editor. The validation lived in Puppeteer. One environment, one session, one continuous flow.

---

## Lessons Learned

**1. Design system repetition is not redundant — it is essential.** Embedding the full design specification in every prompt feels wasteful. It is the only reliable way to maintain consistency across generative calls. Models do not remember previous calls. Every prompt is a fresh context.

**2. Branding must be treated as a first-class design token.** Colors get specified precisely. Typography gets specified precisely. The product name should receive the same treatment — defined once, referenced everywhere, validated automatically.

**3. Variation testing pays for itself.** Generating three variations of the Home screen and selecting the strongest layout added 15 minutes. It saved hours of iteration that would have followed a mediocre first-pass design.

**4. Automated validation is non-optional at scale.** Twenty-one screens cannot be visually inspected with sufficient rigor by a human reviewing screenshots. Puppeteer caught issues — missing labels, broken links, z-index conflicts — that visual review would have missed.

**5. The bottleneck has moved.** Design generation is no longer the slow part. Prompt engineering is. The quality of the output is directly proportional to the specificity of the input. Vague prompts produce vague designs. Precise prompts — with exact colors, exact typography, exact layout constraints — produce designs that convert to code with minimal adjustment.

**6. MCP makes this composable.** Stitch as an MCP tool means it slots into any agent workflow. It is not a standalone application with its own UI and its own context. It is a function call. That composability is what made a 21-screen session feasible — the orchestrating agent could call Stitch, process the result, generate code, run validation, and iterate without ever leaving the conversation.

---

## What This Means

Design-to-code is not a future workflow. It happened in a single session, with production-grade tooling, producing validated components. The gap between "describe what you want" and "here is working code" has collapsed to a tool call.

The remaining challenges are not technical — they are procedural. Prompt discipline. Branding governance. Validation coverage. These are process problems, and process problems have process solutions.

Twenty-one screens. One session. Zero Figma files. The tools are ready. The question is whether the workflows are.

---

## Companion Repository

**[`stitch-design-to-code`](https://github.com/krzemienski/stitch-design-to-code)** — Workflow template for Stitch MCP + AI design-to-code with Puppeteer validation

---

*Part 9 of 10 in the **Agentic Development** series — [View all posts](https://github.com/krzemienski/agentic-development-guide)*

*Nick Krzemienski — March 2026*
