# Abdur Rahman – Portfolio & Blog

Personal portfolio and blog site built with [Jekyll](https://jekyllrb.com/), hosted on [GitHub Pages](https://pages.github.com/).

## Features

- **Portfolio** – About, Skills, Experience, Projects, Education, Contact
- **Blog** – Markdown posts in `_posts/`, listing at `/blog/`, RSS at `/feed.xml`
- **Resume** – Resume page at `/resume/` with PDF download
- **Dark/light theme** – Toggle in nav; preference saved in `localStorage`
- **SEO** – jekyll-seo-tag, sitemap, Open Graph
- **Analytics** – Optional GA4 via `_config.yml`

## Local development

Requires Ruby 3.0+ (or use GitHub Actions / push to GitHub for automatic build).

1. Install Ruby and Bundler, then:
   ```bash
   bundle install
   bundle exec jekyll serve
   ```
2. Open [http://localhost:4000](http://localhost:4000).

## Project structure

- `_config.yml` – Site config, plugins, `google_analytics` (optional)
- `_layouts/` – default, page, post
- `_includes/` – head, nav, footer, theme-toggle, analytics
- `_sass/` – SCSS partials (variables, base, nav, hero, sections, blog, resume, theme, responsive)
- `assets/css/style.scss` – Main stylesheet (imports `_sass/`)
- `assets/js/main.js` – Scroll reveal, nav, mobile menu, theme toggle
- `_posts/` – Blog posts (Markdown with YAML front matter)
- `blog/index.html` – Blog listing (paginated)
- `resume/index.html` – Resume page

## Adding content

- **Blog post**: Create a file in `_posts/` named `YYYY-MM-DD-slug.md` with front matter (`layout: post`, `title`, `date`, `tags`, `description`) and Markdown body.
- **Resume PDF**: Place your PDF as `assets/resume.pdf` so the “Download PDF” button on `/resume/` works.
- **Google Analytics**: Set `google_analytics: "G-XXXXXXXXXX"` in `_config.yml` (production only).

## License

MIT (see LICENSE).
