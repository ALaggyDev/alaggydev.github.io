# Laggy's Blog

A blog for CTF write-ups, gists and random stuff I made on the Internet.

This blog uses the [jekyll-dash](https://github.com/bitbrain/jekyll-dash) theme for [Jekyll](https://jekyllrb.com/).

## Building

Install Ruby and Jekyll in [here](https://jekyllrb.com/docs/step-by-step/01-setup/).

Folders of interest: `/_posts`, `/assets`, `about.md`, and `projects.md`.

```shell
bundle install
bundle exec jekyll serve --livereload

bundle exec htmlproofer _site --disable-external --ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/"
```
