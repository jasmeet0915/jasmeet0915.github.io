---
# ============================================================
# SAMPLE PRODUCT PAGE — Reference for layouts/_default/product.html
# ============================================================
# Place this file inside a page bundle under content/shop/<product-name>/
# alongside a featured.jpg (or featured.png) for the hero image.
#
# To preview: copy this folder into content/shop/, add a featured image,
# and run `hugo server`.
#
# Set draft: true so it never appears on the live site.
# ============================================================

title: "Sample Product Name"
description: "A one-line SEO description of the product."
layout: product
draft: true

# ── Display toggles ──
showHero: false
showTableOfContents: false
showBreadcrumbs: true
showDate: false
showReadingTime: false
showAuthor: false

product:

  # ── Hero section ──
  # category: small label above the title
  # tagline: subtitle / elevator pitch below the title
  category: "Category Label"
  tagline: "One or two sentences that sell the product. Keep it punchy."

  # ── CTA buttons (hero + final section) ──
  # primary: true → filled accent button; false/omitted → secondary outline
  # external: true → opens in new tab
  cta:
    - label: "🛒 Primary Action"
      url: "#order"
      primary: true
    - label: "💬 Secondary Action"
      url: "https://example.com"
      external: true

  # ── Feature cards (3-column grid) ──
  intro:
    label: "Section Label"          # small uppercase label
    heading: "Why this product?"    # section heading
    description: "Optional paragraph expanding on the heading."
    features:
      - icon: "⚡"
        title: "Feature One"
        description: "Short explanation of what this feature does or why it matters."
      - icon: "🎯"
        title: "Feature Two"
        description: "Short explanation of what this feature does or why it matters."
      - icon: "🔒"
        title: "Feature Three"
        description: "Short explanation of what this feature does or why it matters."

  # ── Image gallery (placeholder boxes) ──
  # Replace descriptions with actual images once available.
  gallery:
    label: "Gallery"
    heading: "See it in action"
    images:
      - "Photo description / alt text for image 1"
      - "Photo description / alt text for image 2"
      - "Photo description / alt text for image 3"

  # ── Use-case pills ──
  useCases:
    label: "Use Cases"
    heading: "Who is this for?"
    items:
      - "👩‍💻 Use case one"
      - "🏢 Use case two"
      - "🎓 Use case three"
      - "🎉 Use case four"
    note: "Optional note displayed below the pills."

  # ── Customization section (checklist + image placeholder) ──
  customization:
    label: "Customization"
    heading: "Make it yours"
    items:
      - "Customization option A"
      - "Customization option B"
      - "Customization option C"
      - "Customization option D"
    note: "Optional footnote about the customization process."
    imagePlaceholder: "Description of what the customization image should show"

  # ── Pricing tiers (card grid) ──
  # featured: true → highlighted border + glow
  pricing:
    label: "Pricing"
    heading: "Simple, transparent pricing"
    subtitle: "Optional subtitle with pricing context."
    tiers:
      - tier: "Starter"
        price: "$19"
        unit: "per unit"
      - tier: "Popular"
        price: "$15"
        unit: "per unit"
        featured: true
        note: "Most popular"
      - tier: "Bulk"
        price: "$9"
        unit: "per unit"
        note: "Best value"
    footnote: "Optional footnote — e.g. 'Shipping calculated separately.'"

  # ── Product specs (left column) ──
  specs:
    label: "Specs"
    heading: "Technical details"
    items:
      - label: "Dimensions"
        value: "50 × 30 × 15 mm"
      - label: "Weight"
        value: "25 g"
      - label: "Material"
        value: "PLA / PETG"
      - label: "Origin"
        value: "Made in India 🇮🇳"

  # ── Timeline (right column, beside specs) ──
  timeline:
    label: "Timeline"
    heading: "Order to delivery"
    steps:
      - title: "Step one"
        duration: "1–2 days"
      - title: "Step two"
        duration: "3–5 days"
      - title: "Step three"
        duration: "Depends on location"
    note: "Optional note — e.g. 'Rush orders available on request.'"

  # ── Final CTA banner ──
  finalCta:
    heading: "Ready to get started?"
    description: "Closing line that nudges the visitor to take action."
    buttons:
      - label: "📩 Get a Quote"
        url: "mailto:hello@example.com?subject=Product%20Inquiry"
        primary: true
        external: true
      - label: "💬 Chat"
        url: "https://example.com/chat"
        external: true

# ============================================================
# Any markdown content below the front matter renders at the
# bottom of the page inside a prose section. Leave empty if
# the front matter covers everything.
# ============================================================
---
