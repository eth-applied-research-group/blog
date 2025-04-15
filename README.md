# 📝 Blog - Ethereum Applied Research Group

This is the repository for the Ethereum Applied Research Group's blog built with Hugo.

## 🔍 Prerequisites

- [Hugo](https://gohugo.io/) (Extended version)

## 🚀 Installation

1. Install Hugo Extended version:

   - For macOS: `brew install hugo`
   - For Ubuntu: `sudo apt install hugo`
   - For other systems, see [Hugo Installation Guide](https://gohugo.io/installation/)

2. Clone the repository:

   ```bash
   git clone git@github.com:eth-applied-research-group/blog.git
   cd blog
   ```

3. Update submodules:

   ```bash
   git submodule update --init --recursive
   ```

## 💻 Local Development

To serve the blog locally:

```bash
hugo server -D
```

The site will be available at `http://localhost:1313`

## Creating a new post

To create a new post, use the Hugo CLI:

```bash
hugo new posts/your-post-name.md
```

This will create a new markdown file in `content/posts/` with pre-filled front matter.

Edit the file to add your content. The front matter should include:

```yaml
---
title: "Your Post Title"
date: YYYY-MM-DD
draft: true
---
```

Remove `draft: true` when ready to publish.
