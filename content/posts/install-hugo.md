+++
date = '2025-07-19T15:43:08+08:00'
draft = false
title = 'Install Hugo'
+++

In this guide, I'll walk you through the complete process of setting up a Hugo blog from scratch. Hugo is a fast and flexible static site generator written in Go, perfect for building blogs, documentation sites, and portfolios.

## 1. Install Hugo

### Homebrew
```bash
brew install hugo
```

### Verify Installation
```bash
hugo version
# hugo v0.148.1+extended+withdeploy darwin/amd64 BuildDate=2025-07-11T12:56:21Z VendorInfo=brew
```

## 2. Create a New Site
```bash
hugo new site blog
cd blog
```

## 3. Initialize Git Repository
```bash
git init
```

Remember to add `/public/`, `.hugo_build.lock`, and `.DS_Store` to `.gitignore` to avoid committing built files:

Edit `.gitignore`:
```plaintext
/public/

.hugo_build.lock
.DS_Store
```

## 4. Configure Site Settings

Edit `hugo.toml`:
```toml
baseURL = 'https://mycroftwong.github.io/'  # Must match your GitHub Pages URL
title = "MycroftWong's Blog"
```

## 5. Install Your Favorite Theme
```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

Edit `hugo.toml`:
```toml
theme = "PaperMod"
```

## 6. Create Your First Post
```bash
hugo new posts/install-hugo.md
```

### Post Structure
```markdown
+++
date = '2025-07-19T15:43:08+08:00'
draft = false
title = 'Install Hugo'
+++

## Introduction
Your content here...
```

## 7. Run Local Development Server
```bash
hugo server -D
```
Visit `http://localhost:1313` to preview.

## 8. Set Up GitHub Pages Deployment

### Step 1: Create GitHub Repository
1. Create a new repository named `MycroftWong.github.io` (must match your GitHub username)
2. Push your Hugo site to the `blog` branch:
```bash
git remote add origin https://github.com/MycroftWong/MycroftWong.github.io.git
git branch -M blog
git push -u origin blog
```

### Step 2: Configure GitHub Pages
1. Go to your repository's **Settings > Pages**
2. Under **Build and deployment**:
   - Source: `Deploy from a branch`
   - Branch: `main` (select `/ (root)` folder)
3. Click **Save**

### Step 3: Set Up GitHub Actions
Create `.github/workflows/deploy.yml`:
```yaml
name: Deploy Hugo to GitHub Pages

on:
  push:
    branches: [blog]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout blog branch
        uses: actions/checkout@v4
        with:
          ref: blog
          submodules: "recursive"

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: "latest"

      - name: Build Hugo site
        run: |
          hugo

      - name: Deploy to main branch
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          publish_branch: main
          keep_files: false
          force_orphan: true
```

### Troubleshooting
- **404 Error**: Ensure `baseURL` in `hugo.toml` matches your GitHub Pages URL exactly
- **Build Failures**: Check workflow logs in **Actions** tab

## 9. Build and Publish
```bash
hugo -D  # Builds site to /public
git add .
git commit -m "Initial commit"
git push origin blog
```
Your site will auto-deploy to `https://mycroftwong.github.io` within 2 minutes.

## Next Steps
1. Add a custom domain in GitHub Pages settings
