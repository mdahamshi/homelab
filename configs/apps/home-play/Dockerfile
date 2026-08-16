FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html

HEALTHCHECK --interval=30s --timeout=10s --retries=3 --start-period=10s \
  CMD wget -qO- http://127.0.0.1:80/ || exit 1