source "https://rubygems.org"

gem "jekyll", "~> 4.3.0"
gem "minima", "~> 2.5"
gem "sass-embedded", "~> 1.69.5"
gem "csv"
gem "logger"
gem "base64"
gem "bigdecimal"

# Security floors for transitive dependencies (Gemfile.lock is not tracked,
# so these constraints are what guarantee the patched versions get resolved).
gem "addressable", ">= 2.9.0"        # CVE-2026-35611 (ReDoS in URI templates)
gem "concurrent-ruby", ">= 1.3.7"    # CVE-2026-54904/54905/54906 (lock correctness)
gem "rexml", ">= 3.4.2"              # CVE-2025-58767 (DoS on malformed XML)

group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.12"
  gem "jekyll-sitemap"
  gem "jekyll-seo-tag"
  gem "jekyll-paginate"
  gem "kramdown-parser-gfm"
end

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]