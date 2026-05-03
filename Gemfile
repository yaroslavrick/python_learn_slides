source "https://rubygems.org"

gem "jekyll", "~> 4.4"

# Lock tzinfo for non-Linux platforms that lack zoneinfo files.
platforms :windows, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.1", :platforms => [:windows]
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]
