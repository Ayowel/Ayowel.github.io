# The coal mine

This repository hosts the sources for the personal static website available at https://ayowel.github.io/ .

It contains blog entries and links to personal projects.

## Development

### Pre-requisites

* [Ruby 3.X](https://www.ruby-lang.org/en/downloads/) with bundler installed

### Build the project

If so desired, create a custom `data/github.yml` file (this is required for the GitHub link to work):

```yml
build_revision: rkzehj
owner_url: https://github.com/Ayowel
owner_name: Ayowel
```

Then, build by following those steps:

* From the repository's root, run `bundle config set path '.bundle/vendor' --local`
* From the repository's root, run `bundle install`
* Run the server on port 4000 with `bundle exec jekyll serve`

### Lint

* Build the site: `bundle exec jekyll build`
* Run rubocop: `bundle exec rubocop -D --config .rubocop.yml`
