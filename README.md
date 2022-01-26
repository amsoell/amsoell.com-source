# amsoell.com

Personal web site source, built on [Hugo](https://github.com/gohugoio/hugo).

## Local development

This site can be generated and hosted locally via Docker:

```
docker run --rm -it \
  -v $(pwd):/src \
  -p 1313:1313 \
  klakegg/hugo:0.83.1-alpine \
  server
```