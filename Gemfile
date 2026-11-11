source "https://rubygems.org"

# Static site generator that turns the Markdown in this repo into HTML.
gem "jekyll", "~> 4.4.1"

# Documentation theme: sidebar navigation, search, Mermaid support.
# Pinned so a new theme release can't change the site without a deliberate bump.
gem "just-the-docs", "0.12.0"

group :jekyll_plugins do
  # Rewrites links to .md files (e.g. questions/caching/x.md) into links to the generated pages.
  # GitHub's built-in Pages build turns this on silently; our own build has to ask for it.
  gem "jekyll-relative-links"
end
