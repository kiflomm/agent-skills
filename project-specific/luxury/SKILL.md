---
name: luxury
description: >-
  Design and build UI for a Home Décor & Accessories ecommerce store using a
  Rockett St George–led dark luxury layout with a Graham & Green–inspired news
  banner and nav structure (black header, socials top-right). Use when building
  or redesigning luxury home décor sites, headers, PLPs, PDPs, or when the user
  mentions luxury, Rockett St George, Graham & Green, or this skill.
---

# Luxury Home Décor Design Skill

Build storefront UI for a Home Décor & Accessories brand. Placeholder brand wordmark: `BRAND` unless the user supplies a real name.

### Reference sites

- **Primary layout & design:** https://www.rockettstgeorge.co.uk/
- **Nav + news banner structure only:** https://www.grahamandgreen.co.uk/home-accessories/view-all-home-accessories

## Source hierarchy (hard rules)

| Area | Follow |
|------|--------|
| Page layout, dark palette, hero, grids, PDP, editorial mood | **Rockett St George** (rockettstgeorge.co.uk) |
| News/announcement banner + main nav bar **structure** | **Graham & Green only** (grahamandgreen.co.uk) |
| Social icon placement | **Rockett** (top-right of header) inside the G&G nav layout |
| Footer | **Original** — never copy Rockett or G&G footers |

Do not copy body chrome, product cards, or page layouts from Graham & Green. Do not replicate either site’s footer.

## Hybrid header

Black header (Rockett color), G&G structure, Rockett social placement:

```
[ News banner — G&G promo-bar pattern ]
[ Black header ]
  Row 1: Logo LEFT | Search CENTER | Socials TOP-RIGHT (white)
         above Wishlist / Account / Cart
  Row 2: Centered uppercase category nav
```

### News banner (G&G structure)

- Full-width thin promo bar; short uppercase or title-case lines
- Prefer dual messages (left + center/right), not Rockett’s pink carousel as default
- Colors fit the black site: muted accent bar (e.g. warm stone, soft blush, or deep charcoal-on-black edge) — **not** Graham & Green forest green clone
- Optional single rotating line if needed; keep the clean promo-bar feel

### Header row (black)

- Solid near-black background; white/off-white type and icons
- **Logo:** left-aligned wordmark, all-caps or refined display
- **Search:** centered field like G&G — thin light border on dark, magnifying glass + placeholder (e.g. “Search the collection…”)
- **Socials:** small monochrome white glyphs, top-right of the header block (Rockett), above utilities
- **Utilities:** heart (wishlist), account, bag — G&G set — under/near socials on the right
- **Nav:** centered uppercase categories, generous letter-spacing and gaps (G&G); dropdowns allowed (Rockett)

### Mobile header

- Collapse category nav into a menu; keep search and bag reachable
- Socials may move into the menu drawer or footer only on small screens

## Page blueprints (Rockett-led)

### Homepage

- Dominant full-bleed or edge-to-edge hero; sparse headline + one short line + one CTA group
- Brand/wordmark as a strong signal; do not bury it as nav-only text
- Editorial storytelling blocks, category entry points, featured collections
- Image-first, calm hierarchy; avoid white “emporium” chrome from G&G outside the header
- Hero budget: brand, one headline, one supporting sentence, one CTA group, one dominant visual — no stat strips or promo card clutter in the first viewport

### Category / PLP

- Product grid (image-first cards); title + price below image
- Filters/sort as a quiet toolbar — not a dashboard
- Generous whitespace; dark or high-contrast surfaces consistent with Rockett
- Hover: subtle image scale or fade; no heavy shadows or glow

### Product / PDP

- Gallery + details column; price, add-to-bag, short description
- Related products strip below; keep hierarchy editorial and calm
- Avoid noisy badges, sticker overlays, and multi-layer card chrome on media

### Footer (original — invent, do not copy references)

Use this structure (not Rockett/G&G):

```
[ Full-width band matching black header system ]
  Top: Brand wordmark LEFT | Newsletter field + submit RIGHT
  Middle: 3–4 link columns (Shop, Help, About, Follow) — Follow repeats socials
  Bottom legal row: © year · Privacy · Terms · Cookies — hairline separator above
```

- Light-on-dark; no multi-column newsletter mega-footers copied from either reference
- One restrained accent on the subscribe control only

## Design tokens

```css
:root {
  --bg: #0a0a0a;
  --bg-elevated: #141414;
  --surface: #1a1a1a;
  --text: #f5f2eb;
  --text-muted: #a8a29a;
  --border: rgba(245, 242, 235, 0.18);
  --accent: #c4a484; /* muted warm metal — use sparingly */
  --banner-bg: #2a2420;
  --banner-text: #f5f2eb;
  --font-ui: "DM Sans", "Helvetica Neue", sans-serif; /* swap for project fonts; avoid Inter/Roboto/Arial defaults */
  --font-display: "Cormorant Garamond", Georgia, serif; /* brand/hero only */
  --tracking-nav: 0.12em;
}
```

- One muted accent only; no purple-indigo gradients, cream+terracotta clichés, or glow stacks
- Nav: uppercase + `--tracking-nav`
- Motion (2–3 max): nav underline reveal, soft fade-in, product image hover scale (~1.03)

## Do / don’t

**Do**

- Rockett mood for all page bodies; G&G only for banner + nav structure
- Black header; white socials top-right above utilities
- Original footer as specified above
- Expressive fonts; dark atmospheric backgrounds (gradients/texture OK if subtle)

**Don’t**

- Copy G&G page layouts, white header, or forest-green banner literally
- Copy Rockett or G&G footers
- Cards in the hero; floating promo stickers on media
- Default AI luxury looks (purple gradients, warm cream + terracotta broadsheet)

## Implementation checklist

When building or redesigning UI with this skill:

- [ ] News banner uses G&G promo-bar structure with site-fitting (non–forest-green) colors
- [ ] Header is black; logo left, search center, socials top-right, utilities below socials
- [ ] Nav is centered, uppercase, letter-spaced
- [ ] Home / PLP / PDP follow Rockett-led dark editorial layouts
- [ ] Footer is original (wordmark + newsletter + columns + legal) — not from references
- [ ] Tokens applied; accent used sparingly; 2–3 restrained motions
- [ ] Mobile: menu collapse; search + bag reachable; socials relocated sanely
- [ ] No verbatim copy or assets from grahamandgreen.co.uk or rockettstgeorge.co.uk
