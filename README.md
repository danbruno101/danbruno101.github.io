# danielbruno.dev

Personal blog built with [Astro](https://astro.build), deployed to GitHub Pages at
[danielbruno.dev](https://danielbruno.dev).

## Commands

| Command           | Action                                      |
| ------------------ | -------------------------------------------- |
| `npm install`       | Install dependencies                         |
| `npm run dev`       | Start local dev server at `localhost:4321`   |
| `npm run build`     | Type-check and build the site to `./dist/`   |
| `npm run preview`   | Preview the build locally before deploying   |

## Content

Blog posts are Markdown files in `src/content/blog/`. Frontmatter fields (`title`,
`description`, `pubDate`, optional `updatedDate` and `heroImage`) are validated by the schema in
`src/content.config.ts`.

## Deployment

Pushes to `main` trigger `.github/workflows/deploy.yml`, which builds the site with
[`withastro/action`](https://github.com/withastro/action) and publishes it via GitHub Pages.
The custom domain is configured through `public/CNAME`, which GitHub Pages reads on deploy — the
domain's DNS should already point at GitHub Pages per
[GitHub's custom domain docs](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site).
