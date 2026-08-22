# The coal mine

This repository hosts the sources for the personal static website available at https://ayowel.github.io/ .

It contains blog entries and links to personal projects.

## Development

### Pre-requisites

* [Ruby 3.X](https://www.ruby-lang.org/en/downloads/) with bundler installed

### Build the project

* From the repository's root, run `bundle config set path '.bundle/vendor' --local`
* From the repository's root, run `bundle install`
* Run the server on port 4000 with `bundle exec jekyll serve`

### Lint

* Build the site: `bundle exec jekyll build`
* Run rubocop: `bundle exec rubocop -D --config .rubocop.yml`
