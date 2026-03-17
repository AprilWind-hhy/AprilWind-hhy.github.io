# frozen_string_literal: true

source "https://rubygems.org"

# 核心主题依赖（保留你的原有版本）
gem "jekyll-theme-chirpy", "~> 7.5"
# 补充缺失的核心依赖（解决 404/构建失败的关键）
gem "jekyll-remote-theme", "~> 0.4.3"
# 明确 Jekyll 版本，避免版本冲突
gem "jekyll", "~> 4.4.0"

# 测试环境依赖（保留原有）
gem "html-proofer", "~> 5.0", group: :test

# Windows/JRuby 平台适配（保留原有）
platforms :windows, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Windows 平台依赖（保留原有）
gem "wdm", "~> 0.2.0", :platforms => [:windows]
