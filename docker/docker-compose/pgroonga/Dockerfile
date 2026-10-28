# PostgreSQL with pgvector and pgroonga extensions.
#
# pgroonga is a multilingual full-text search extension built on Groonga.
# It works out of the box for CJK (Chinese, Japanese, Korean) and other
# non-whitespace-segmented languages via the TokenBigram tokenizer.
FROM groonga/pgroonga:4.0.8-debian-17

# Install pgvector on top of the pgroonga base image (which already provides
# pgroonga, the Groonga library, and the PostgreSQL PGDG package repository).
RUN sed -i \
    -e 's|http://deb.debian.org/|https://deb.debian.org/|g' \
    -e 's|http://apt.postgresql.org/|https://apt.postgresql.org/|g' \
    /etc/apt/sources.list.d/debian.sources /etc/apt/sources.list.d/pgdg.list \
    && apt-get update && apt-get install -y --no-install-recommends \
    postgresql-17-pgvector=0.8.6-1.pgdg13+1 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
