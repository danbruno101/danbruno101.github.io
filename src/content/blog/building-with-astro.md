---
title: 'Building with Astro'
description: 'A few notes on why Astro is a good fit for a simple content-driven site.'
pubDate: 2026-08-04
---

Astro ships zero JavaScript to the client by default and lets you write pages in `.astro`,
Markdown, or MDX. For a mostly-static blog like this one, that means fast page loads without
giving up componentization for shared layout pieces like the header and footer.

Content lives in `src/content/blog` as Markdown files, validated against a schema in
`src/content.config.ts`. Adding a new post is just adding a new file.
