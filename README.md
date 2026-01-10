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

The site uses GitHub Actions to build and deploy automatically. Just push to `master`:

```bash
git add .
git commit -m "Update site"
git push origin master
```

### First-time setup

In your GitHub repository settings:

1. Go to **Settings** > **Pages**
2. Under **Source**, select **GitHub Actions**
3. Push any change to trigger the first build

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
