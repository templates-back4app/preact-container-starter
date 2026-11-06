# Preact (Vite) Container Starter

The official Preact (Vite) starter, unchanged, plus a production-ready multi-stage Dockerfile
(Node 22 build stage, nginx 1.27 serving the build on port 8080 with SPA fallback,
gzip, and immutable caching for hashed assets).

Run it locally:

    docker build -t preact-container-starter .
    docker run --rm -p 8080:8080 preact-container-starter

Then open http://localhost:8080.

Deploy guide: (coming) on the Back4app blog.
