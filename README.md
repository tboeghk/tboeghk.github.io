# The souce of my thiswayup.de website

These are the Jekyll sources for [thiswayup.de]. Feel free to
use this for your personal website but be aware that the layout
is not free. A usage license must be purchased by

## Running locally

```
docker run -it -v $(pwd):/srv/jekyll \
    -p 4000:4000 jekyll/jekyll:3.8.5 \
    jekyll serve --watch
```