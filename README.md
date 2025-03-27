# Open3FS Blog

Welcome to the Open3FS blog repository. This site is built with Jekyll and hosted on GitHub Pages.

## Writing Posts

### Method 1: GitHub Web Interface
The simplest way to write posts is directly through GitHub's web interface.

### Method 2: Local Development
Clone the repository and write posts locally.

### Post Guidelines

1. Create new posts in the `_posts` directory
2. Follow the naming convention: `YYYY-MM-DD-post-title.md`
3. Include the following front matter at the top of each post:

   ```yaml
   ---
   layout: default
   title: "Your Post Title"
   date: YYYY-MM-DD HH:MM:SS +0800
   ---
   ```

4. For images:
   - Place images in the `assets/imgs` directory
   - Reference them in posts using:
     ```markdown
     ![Image Description]({{ site.url }}/assets/imgs/your-image.png)
     ```

## Prerequisites

- Ruby (recommended version from [GitHub Pages versions](https://pages.github.com/versions/))
- Bundler
- Git

## Local Development

1. Clone the repository:
   ```bash
   git clone https://github.com/open3fs/open3fs.github.io.git
   cd open3fs.github.io
   ```

2. Install dependencies:
   ```bash
   bundle install
   ```

3. Start the local server:
   ```bash
   ./serve.sh
   ```

4. Visit [http://localhost:4000](http://localhost:4000) to view the site.

## Ruby Environment Setup on macOS

We recommend using `chruby` for Ruby version management. Follow the official guide:
[Jekyll Installation on macOS](https://jekyllrb.com/docs/installation/macos/)

## Dependencies

This site uses the following key dependencies:
- Jekyll
- GitHub Pages
- jekyll-remote-theme
- jekyll-feed

For the complete list of supported versions, refer to [GitHub Pages versions](https://pages.github.com/versions/).

**DO NOT USE UNSUPPORTED VERSIONS**.
