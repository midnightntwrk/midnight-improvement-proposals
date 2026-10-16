source "https://rubygems.org"

# Jekyll static-site generator for the Midnight Improvement Proposals site.
# See: https://jekyllrb.com/
gem "jekyll", "~> 4.3"

# Theme: just-the-docs — sidebar navigation + built-in full-text search.
# https://just-the-docs.com/
gem "just-the-docs", "~> 0.10"

group :jekyll_plugins do
  # Rewrites in-document relative .md links to their built page URLs.
  gem "jekyll-relative-links", "~> 0.7"
end

# Lock the platform so `bundle install` in CI resolves the same gems.
gem "webrick", "~> 1.8"
