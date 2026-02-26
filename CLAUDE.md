# CLAUDE.md — Project Memory

## Project
Personal portfolio and technical blog for **Abdur Rahman** — Lead Mobile App Developer (Flutter / React Native).
Built with Jekyll, hosted on GitHub Pages at https://arfuhad.github.io.

## Quick Commands
- `bundle exec jekyll serve` — Local dev server at http://localhost:4000
- `bundle exec jekyll serve --unpublished` — Include draft posts (published: false)
- `bundle exec jekyll build` — Build to `_site/`
- Deploy: Push to `master` branch triggers GitHub Actions (`static.yml`)

## Project Structure
```
_config.yml          # Site config, plugins, author, analytics
_layouts/
  default.html       # Base HTML shell (head, nav, main, footer, JS)
  post.html          # Blog post layout (extends default)
  page.html          # Generic page layout (extends default)
_includes/
  head.html          # <head> with theme init, fonts, CSS, {% seo %}
  nav.html           # Top nav with section links + blog + resume
  footer.html        # Footer with copyright
  theme-toggle.html  # Dark/light toggle button
  analytics.html     # GA4 (production only)
  seo-meta.html      # Custom OG overrides placeholder
_sass/
  _variables.scss    # CSS custom properties (colors, fonts)
  _base.scss         # Reset, scrollbar, grid bg, orbs, container
  _nav.scss          # Navigation styles
  _hero.scss         # Hero section
  _sections.scss     # Skills, experience, projects, education
  _contact.scss      # Contact section
  _blog.scss         # Blog list + post content styles
  _resume.scss       # Resume page styles
  _theme.scss        # Light theme overrides
  _animations.scss   # Scroll reveal, float keyframes
  _responsive.scss   # Media queries
  _footer.scss       # Footer styles
assets/
  css/style.scss     # Main entry — imports all _sass/ partials
  js/main.js         # Theme toggle, scroll reveal, nav, mobile menu
_posts/              # Blog posts (YYYY-MM-DD-slug.md)
blog/index.html      # Blog listing page (paginated)
resume/index.html    # Resume page with PDF download
index.html           # Homepage — hero, skills, experience, projects, education, contact
404.html             # Custom 404 page
```

## Key Conventions

### Blog Posts
- **Filename**: `_posts/YYYY-MM-DD-slug.md` (kebab-case slug)
- **Required front matter**:
  ```yaml
  ---
  layout: post
  title: "Post Title Here"
  date: YYYY-MM-DD HH:MM:SS +0600
  published: true
  tags:
    - tag-one
    - tag-two
  description: "One-sentence summary for SEO and blog listing excerpt."
  ---
  ```
- **Timezone**: Always `+0600` (Bangladesh Standard Time)
- Tags: lowercase kebab-case (`flutter`, `react-native`, `ci-cd`, `ml-kit`)
- Blog listing shows: title, date, tags (max 3), read time (words / 200), excerpt (30 words)
- Wrap JSX/template code in `{% raw %}` / `{% endraw %}` to prevent Liquid errors

### Writing Style (from existing posts)
- Professional but direct — no fluff or filler
- Technically confident — assumes reader is a developer
- First person sparingly — mostly instructional voice
- Structure: problem statement → solution overview → implementation → practical patterns
- Generous code blocks with language hints, tables for comparisons, ASCII diagrams for architecture
- Horizontal rules (`---`) between major sections
- Blockquotes (`>`) for important warnings/notes
- End posts with repo/resource links, not "thanks for reading"

### Homepage HTML Patterns
- Sections: Hero → Skills → Experience → Projects → Education → Contact
- Section structure:
  ```html
  <section class="section" id="sectionname">
    <div class="container">
      <div class="reveal">
        <div class="section-label">// label text</div>
        <h2 class="section-title">Title</h2>
        <p class="section-desc">Subtitle.</p>
      </div>
      <!-- content -->
    </div>
  </section>
  ```
- Alternate sections use `style="background:var(--bg-secondary);"`
- All new elements need `class="reveal"` for scroll animation

### Tech Tag Color Mapping (project cards)
| Tech           | Background                     | Color                     |
|----------------|--------------------------------|---------------------------|
| Flutter        | `rgba(59,130,246,0.1)`         | `var(--accent-bright)`    |
| React Native   | `rgba(59,130,246,0.1)`         | `var(--accent-bright)`    |
| Dart           | `rgba(34,211,238,0.1)`         | `var(--cyan)`             |
| Firebase       | `rgba(251,146,60,0.1)`         | `var(--orange)`           |
| REST API       | `rgba(34,211,238,0.1)`         | `var(--cyan)`             |
| GraphQL        | `rgba(167,139,250,0.1)`        | `var(--purple)`           |
| WebRTC         | `rgba(251,146,60,0.1)`         | `var(--orange)`           |
| ML Kit         | `rgba(52,211,153,0.1)`         | `var(--green)`            |
| Open Source    | `rgba(52,211,153,0.1)`         | `var(--green)`            |
| Social Login   | `rgba(52,211,153,0.1)`         | `var(--green)`            |
| Payments / IAP | `rgba(167,139,250,0.1)`        | `var(--purple)`           |
| Face Detection | `rgba(167,139,250,0.1)`        | `var(--purple)`           |
| Crashlytics    | `rgba(251,146,60,0.1)`         | `var(--orange)`           |

## Do's
- Use `{{ '/path/' | relative_url }}` for all internal links in templates
- Use HTML entities in HTML files: `&ndash;`, `&middot;`, `&amp;`
- Test locally with `bundle exec jekyll serve` before pushing
- Keep resume (`resume/index.html`) consistent with homepage experience/skills

## Don'ts
- Do NOT edit files in `_site/` — it's build output, gitignored
- Do NOT remove `future: true` from `_config.yml` unless intentional
- Do NOT use Liquid syntax inside fenced code blocks without `{% raw %}`/`{% endraw %}`
- Do NOT add `google_analytics` value in development

## Deployment
- **Dev branch**: `dev` (working branch)
- **Main branch**: `master` (push triggers GitHub Actions → GitHub Pages)
- GitHub Pages builds Jekyll server-side using the `github-pages` gem

## Author Info
- **Name**: Abdur Rahman
- **Role**: Lead Mobile App Developer (Team Lead at Potential Inc.)
- **Experience**: 6+ years, 10+ apps shipped, 3 countries served
- **Core stack**: Flutter, Dart, React Native, Bloc, Firebase, CI/CD
- **Also**: .NET Core, Node.js, Django, GraphQL, Python, Go
- **Education**: BSc CSE, United International University (2014–2019)
- **Location**: Bangladesh, open to EU relocation
- **Links**: github.com/arfuhad, linkedin.com/in/abdur-rahman-fuhad
