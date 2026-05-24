---
name: brand-applicator
description: >
  Apply your personal or company brand to any output Claude produces.
  Trigger this skill whenever the user says "make this branded", "apply my branding",
  "brand this", "use my brand", "make it on-brand", "add our branding", or any similar
  phrase indicating they want their visual identity applied to a document, slide deck,
  carousel, code, report, invoice, or interface.

  Also trigger when the user asks to CREATE a new branded asset — e.g. "create a branded
  LinkedIn carousel", "write a branded report", "build a branded dashboard" — even if they
  don't explicitly say "apply branding". Anytime an output should look and feel like it
  came from the user's brand, use this skill.

  This skill covers: PowerPoint decks (.pptx), Word documents (.docx), PDF reports,
  LinkedIn carousels, React components, HTML pages, invoices, email templates, and
  any other content type Claude can produce.
---

# Brand Applicator

You are acting as a brand designer with deep knowledge of this user's brand identity.
Your job is to take any content — a document, a presentation, a carousel, a coded interface,
a report — and make it look and feel unmistakably like it came from this brand.

## Step 1: Load the Brand Identity

**Always read the brand reference first:**

```
references/brand-identity.md
```

This file contains the complete brand spec: colours, typography, logo rules, tone of voice,
layout principles, and output-specific overrides. Treat it as your single source of truth.
Never invent brand values — if something is marked `▶ [placeholder]`, note it to the user
and use a sensible neutral default until they fill it in.

If key fields are still placeholders (especially colours and fonts), flag them at the start
of your response with a brief note like:
> "⚠️ I noticed a few brand values are still placeholders in your brand-identity.md
> (primary colour, headline font). I've used defaults for now — update the file whenever
> you're ready and I'll apply your real values."

---

## Step 2: Identify the Output Type

Determine what type of branded output is being requested. Match to one of these modes:

| Mode              | Triggers                                                           |
|-------------------|--------------------------------------------------------------------|
| **PPTX**          | "slide deck", "presentation", "PowerPoint", .pptx                 |
| **DOCX**          | "Word doc", "report", "document", .docx                           |
| **PDF**           | "PDF", "pdf report", "branded PDF"                                 |
| **CAROUSEL**      | "LinkedIn carousel", "carousel", "social slides"                   |
| **REACT**         | "React component", "dashboard", "React app", .jsx, .tsx            |
| **HTML**          | "webpage", "landing page", "HTML", .html                           |
| **INVOICE**       | "invoice", "bill", "receipt"                                       |
| **EMAIL**         | "email template", "newsletter", "HTML email"                       |
| **OTHER**         | Any output not listed above — apply brand colours, fonts, and tone |

Then follow the instructions for that mode below.

---

## Mode: PPTX (PowerPoint / Slide Deck)

Use the `pptx` skill. Before generating slides, read:
- `references/brand-identity.md` → Section 6 (PowerPoint overrides) and Section 2 (Colours) and Section 3 (Typography)

Apply these brand rules to every slide:
- **Master layout**: Use the brand's slide dimensions and background style
- **Title slide**: Apply the brand's title slide treatment (colour, logo placement)
- **Headings**: Use the primary typeface at the brand's specified sizes
- **Body text**: Use the secondary typeface
- **Colour**: Use only palette colours — never default PowerPoint theme colours
- **Logo**: Place in the position specified in the brand spec
- **Charts/data**: Use brand colours for data series (primary for hero data, neutrals for context)
- **Divider slides**: Apply the brand's section divider style
- **Consistent footer**: Include brand name / slide number as per spec

Tone-check all text copy against the tone of voice rules in Section 5.

---

## Mode: DOCX (Word Document / Report)

Use the `docx` skill. Apply these brand rules:
- **Margins**: Use brand-specified document margins
- **Heading styles**: Map H1/H2/H3 to brand typography and colours
- **Body text**: Use secondary typeface, brand-specified body size and line height
- **Header**: Include brand logo and document title in header
- **Footer**: Page number + website URL as per brand spec
- **Tables**: Apply brand table style (header row colour, alternating rows)
- **Pull quotes / callouts**: Use primary or secondary brand colour as highlight
- **Colour accents**: Brand primary for key elements; neutrals for structure

---

## Mode: PDF

Use the `pdf` skill. Apply these brand rules:
- **Cover page**: Full brand cover treatment (see Section 7 – PDF overrides)
- **Section dividers**: Brand-coloured divider pages with white section titles
- **Body pages**: Consistent header/footer with brand colours and logo
- **Typography**: Match brand font sizes and weights throughout
- **Charts & visuals**: Brand colour palette for all data visualisations

---

## Mode: CAROUSEL (LinkedIn / Social Carousel)

Use the `branded-carousel-generator` skill if available. Otherwise, generate the carousel
directly using the brand spec.

Apply these rules to every slide:
- **Dimensions**: Use brand-specified carousel size (default: 1080×1080px)
- **Cover slide**: Apply the brand's cover treatment — bold headline, brand colour bg, white text
- **Body slides**: Consistent header bar, brand colours, brand typefaces, white background or brand bg
- **Typography**: Headline font for slide titles; body font for supporting text
- **Colour**: Primary colour for accents, CTAs, and progress indicators
- **Logo**: Include small logo/icon on every slide (consistent position)
- **CTA slide**: Last slide — follow the brand's CTA format (see Section 7)
- **Tone**: Apply brand voice rules — punchy, benefit-led headlines; concise body copy
- **Slide count**: Respect the brand's preferred range (usually 7–10 slides)

Deliver as a self-contained HTML file with individual slides that can be screenshotted,
OR as a structured visual spec the user can take into Canva / Figma.

---

## Mode: REACT (React Component / Dashboard)

Apply brand tokens as CSS custom properties at the top of every component:

```css
:root {
  /* Brand Colours */
  --brand-primary: [hex from brand-identity.md];
  --brand-secondary: [hex];
  --brand-tertiary: [hex];
  --brand-dark: [hex];
  --brand-mid: [hex];
  --brand-light: [hex];
  --brand-white: #ffffff;

  /* Brand Typography */
  --font-headline: '[headline font]', [fallback stack];
  --font-body: '[body font]', [fallback stack];

  /* Spacing & Radius */
  --space-xs: 8px;
  --space-sm: 16px;
  --space-md: 24px;
  --space-lg: 48px;
  --space-xl: 64px;
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-full: 999px;
}
```

Then apply consistently:
- Use `var(--brand-primary)` etc. — never hardcode hex values in component styles
- Headings: `font-family: var(--font-headline)` with brand weight and tracking
- Body: `font-family: var(--font-body)` with brand line height
- Buttons: Primary CTA = brand primary fill; secondary = outlined border
- Cards: Brand border radius, brand card padding, subtle neutral bg
- Link colours: Brand primary
- Import Google Fonts (or whichever source) matching the brand fonts
- Use Tailwind utility classes where applicable — map to brand values

---

## Mode: HTML (Webpage / Landing Page)

Apply brand tokens via `<style>` block or inline CSS at the top of the HTML file.
Follow the same CSS custom property pattern as React mode above.

Apply to all page elements:
- Meta `<title>` and OG tags with brand name
- Page background = brand white or light neutral
- Headings: brand headline font, primary colour or dark
- Body text: brand body font, dark/mid colour
- CTAs / buttons: brand primary colour
- Navigation: brand colours and logo placement
- Footer: brand dark background with white text, social links

---

## Mode: INVOICE

Use the `pdf` skill or generate clean HTML/PDF. Apply:
- Brand logo top-left (or per brand logo placement spec)
- Brand primary colour for the header bar and section dividers
- Brand typography throughout
- Neutral table for line items; header row in brand primary
- Footer with brand contact details, website, payment instructions
- Brand tone for any accompanying text (e.g., payment terms, thank-you note)

---

## Mode: EMAIL TEMPLATE

Generate HTML email with inline styles (email clients ignore stylesheets):
- Header: Brand primary colour banner with white logo
- Body background: Brand light/white
- Typography: Web-safe fonts that most closely match brand fonts (e.g., Georgia for serif, Arial for sans-serif), with brand fonts as preferred first choice
- CTA button: Brand primary colour, brand border radius, white text
- Footer: Brand dark, white text, unsubscribe link, brand address
- Tone: Brand voice rules applied to all copy

---

## General Principles (All Modes)

These apply regardless of output type:

**Visual consistency**
- Never mix more than 2 typefaces in one output
- Use only palette colours — never default app theme colours
- Maintain the brand's spacing system (base unit multipliers)
- Keep layout clean with generous white space — avoid cluttered designs

**Tone consistency**
- All text copy (headings, body, captions, CTAs) must reflect the brand's tone of voice
- Use the brand's vocabulary preferences — apply the "use/avoid" word list
- Headlines should lead with the benefit, follow the brand's capitalization rules

**Logo handling**
- Never stretch, recolour, or modify logos
- Apply the correct variant (full colour, white, dark, icon) based on background
- Respect clear space rules

**When brand data is missing**
- If a field in brand-identity.md is still a placeholder, use a sensible, clean default
  and flag the gap to the user in a short note at the end of your response
- Do not invent hex codes or font names that aren't in the spec

---

## Quick Start for the User

To get the best results from this skill:

1. **Open** `references/brand-identity.md` (find it inside this skill folder)
2. **Replace** every line starting with `▶` with your actual brand values
3. **Say** "make this branded" or "apply my branding" to any output you're working on

The more fields you fill in, the more precisely Claude will match your brand.

If you have a brand guidelines PDF, say:
> "Here's my brand guidelines PDF — extract everything and update my brand-identity.md"
Claude will read the PDF and fill in the reference file for you.
