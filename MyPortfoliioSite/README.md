# Patrick Lau - Professional Portfolio & Blog

A professional Hugo-based website showcasing AI and technology leadership experience with an integrated blog platform.

## Features

- **Responsive Design**: Elegant, professional design that works on all devices
- **Blog Platform**: Built-in blogging capability with sample articles
- **Professional Typography**: Using Playfair Display and IBM Plex Sans fonts
- **Smooth Animations**: Scroll-based animations and micro-interactions
- **SEO Optimized**: Proper meta tags and semantic HTML
- **Fast Loading**: Minimal dependencies, optimized CSS and JavaScript

## Local Development Setup

### Prerequisites

Install Hugo Extended version (required for SCSS processing):

**macOS:**
```bash
brew install hugo
```

**Linux:**
```bash
# Download the latest extended version
wget https://github.com/gohugoio/hugo/releases/download/v0.138.0/hugo_extended_0.138.0_Linux-64bit.tar.gz
tar -xzf hugo_extended_0.138.0_Linux-64bit.tar.gz
sudo mv hugo /usr/local/bin/
```

**Windows:**
```bash
choco install hugo-extended
```

Or download from: https://github.com/gohugoio/hugo/releases

### Running Locally

1. Navigate to the site directory:
```bash
cd patrick-lau-site
```

2. Start the Hugo development server:
```bash
hugo server -D
```

3. Open your browser and visit:
```
http://localhost:1313
```

The site will auto-reload when you make changes to content or layouts.

## Content Management

### Adding a New Blog Post

1. Create a new markdown file in `content/blog/`:
```bash
hugo new blog/my-new-post.md
```

2. Edit the front matter and content:
```markdown
---
title: "Your Post Title"
date: 2026-01-29
draft: false
tags: ["AI", "Technology", "Leadership"]
summary: "A brief summary of your post"
---

Your content here...
```

3. Save the file and the site will automatically update.

### Editing Your Information

Update your personal information in `hugo.toml`:
- Email address
- Phone number
- LinkedIn URL
- Location

### Customizing Colors and Fonts

Edit `static/css/style.css` and modify the CSS variables at the top:
```css
:root {
    --color-primary: #1a1a2e;
    --color-accent: #0f4c75;
    --font-display: 'Playfair Display', Georgia, serif;
    --font-body: 'IBM Plex Sans', -apple-system, sans-serif;
}
```

## Building for Production

To build the static site for deployment:

```bash
hugo
```

This generates the static site in the `public/` directory.

## Deploying to GitHub Pages

### Option 1: GitHub Actions (Recommended)

1. Create a new GitHub repository for your site.

2. Create `.github/workflows/hugo.yml`:
```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.138.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Build with Hugo
        env:
          HUGO_ENVIRONMENT: production
        run: |
          hugo \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

3. Push your code to GitHub:
```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git push -u origin main
```

4. Enable GitHub Pages:
   - Go to your repository settings
   - Navigate to "Pages" section
   - Under "Source", select "GitHub Actions"

5. Your site will be available at: `https://YOUR_USERNAME.github.io/YOUR_REPO/`

### Option 2: Manual Deployment

1. Build the site:
```bash
hugo
```

2. Push the `public/` directory to the `gh-pages` branch:
```bash
cd public
git init
git add .
git commit -m "Deploy site"
git branch -M gh-pages
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git push -f origin gh-pages
```

3. Enable GitHub Pages in repository settings, selecting `gh-pages` branch.

## Custom Domain Setup

If you want to use a custom domain (e.g., patricklau.dev):

1. Create a file `static/CNAME` with your domain:
```
patricklau.dev
```

2. Update `baseURL` in `hugo.toml`:
```toml
baseURL = 'https://patricklau.dev/'
```

3. Configure your domain's DNS settings:
   - Add an A record pointing to GitHub's IPs:
     - 185.199.108.153
     - 185.199.109.153
     - 185.199.110.153
     - 185.199.111.153
   - Or add a CNAME record pointing to `YOUR_USERNAME.github.io`

## Directory Structure

```
patrick-lau-site/
├── content/
│   ├── _index.md          # Home page content
│   └── blog/              # Blog posts
│       ├── _index.md
│       ├── implementing-eu-ai-act.md
│       ├── infrastructure-to-ai.md
│       └── mlops-maturity-gap.md
├── layouts/
│   ├── _default/
│   │   ├── baseof.html    # Base template
│   │   ├── list.html      # List pages (blog index)
│   │   └── single.html    # Single pages (blog posts)
│   ├── partials/
│   │   ├── header.html
│   │   └── footer.html
│   └── index.html         # Home page template
├── static/
│   ├── css/
│   │   └── style.css      # Main stylesheet
│   └── js/
│       └── script.js      # JavaScript
├── hugo.toml              # Site configuration
└── README.md
```

## Customization Tips

### Adding New Sections

Create new content sections in `content/` and corresponding templates in `layouts/`.

### Modifying the Navigation

Edit the menu configuration in `hugo.toml`:
```toml
[[menu.main]]
  name = 'New Page'
  url = '/new-page/'
  weight = 6
```

### Changing Fonts

1. Update font imports in `layouts/_default/baseof.html`
2. Update CSS variables in `static/css/style.css`

### Adding Images

Place images in `static/images/` and reference them in content:
```markdown
![Alt text](/images/my-image.jpg)
```

## Performance Optimization

- Images are loaded on-demand
- CSS and JS are minified in production builds
- Fonts are loaded with proper preconnect
- Lazy loading for off-screen content

## Browser Support

- Chrome/Edge (latest 2 versions)
- Firefox (latest 2 versions)
- Safari (latest 2 versions)
- Mobile browsers (iOS Safari, Chrome Mobile)

## License

© 2026 Patrick Lau. All rights reserved.

## Support

For issues or questions:
- Email: ppklau@gmail.com
- LinkedIn: https://www.linkedin.com/in/patricklau001

---

Built with [Hugo](https://gohugo.io/) - The world's fastest framework for building websites.
