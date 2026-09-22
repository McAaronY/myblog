# frozen_string_literal: true

source "https://rubygems.org"

gem "jekyll-theme-chirpy", "~> 7.6"

gem "html-proofer", "~> 5.0", group: :test

# _config.yml 里的 `timezone` 需要 tzinfo；Linux/macOS 直接读系统 zoneinfo，
# Windows / JRuby 没有系统时区数据库，因此额外依赖 tzinfo-data。
gem "tzinfo", ">= 1", "< 3"

platforms :windows, :jruby do
  gem "tzinfo-data"
end

gem "wdm", "~> 0.2.0", :platforms => [:windows]
