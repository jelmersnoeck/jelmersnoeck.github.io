# jlmr.dev

Personal blog built with [Hugo](https://gohugo.io/).

## Prerequisites

Install Hugo:

```bash
# macOS
brew install hugo

# Or download from https://gohugo.io/installation/
```

## Development

Run the local development server:

```bash
hugo server -D
```

This starts a server at `http://localhost:1313` with live reload.

## Adding a New Post

Create a new post:

```bash
hugo new posts/my-new-post.md
```

Or manually create a file in `content/posts/` with this frontmatter:

```markdown
---
title: "My New Post"
date: 2024-01-15
---

Your content here...
```

## Building for Production

Generate the static site:

```bash
hugo
```

Output is written to the `public/` directory.

## Deploying to GitHub Pages

The site is deployed to GitHub Pages. Push changes to the `master` branch:

```bash
git add .
git commit -m "Update site"
git push origin master
```

For GitHub Pages deployment, you can either:

1. **Use GitHub Actions** (recommended): Create `.github/workflows/hugo.yml` to build and deploy automatically
2. **Build locally**: Run `hugo`, commit the `public/` folder contents to the repo root

## Project Structure

```
.
├── content/
│   └── posts/          # Blog posts (markdown)
├── layouts/
│   ├── _default/       # Default templates
│   └── index.html      # Homepage template
├── static/
│   └── images/         # Static images
├── hugo.toml           # Hugo configuration
├── CNAME               # Custom domain
└── README.md
```

## License

Content is copyright Jelmer Snoeck. Code is MIT licensed.
