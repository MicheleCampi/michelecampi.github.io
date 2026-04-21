source "https://rubygems.org"

# GitHub Pages gem — include Jekyll + plugin supportati
gem "github-pages", group: :jekyll_plugins

# Esplicita i plugin in uso
group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
end

# Fix Windows/WSL: richiede tzinfo-data su sistemi senza timezone DB nativo
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.1", :platforms => [:mingw, :x64_mingw, :mswin]
