To run the site locally:

```bash
bundle exec jekyll serve
```

or, to delete the local build first, which seems to avoid weird issues where some templated files don't get rebuilt:

```bash
rm -rf ./_site/ && bundle exec jekyll serve
```
