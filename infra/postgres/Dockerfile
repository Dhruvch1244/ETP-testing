# ==============================================================================
# Bakes the schema/seed SQL into the image instead of bind-mounting them at
# runtime. A CI agent that talks to Docker over a mounted socket (Jenkins
# running docker-outside-of-docker) has no real host path to bind-mount from
# — its workspace lives inside its own container — so runtime bind mounts of
# these files silently turn into empty directories. Build-time COPY sidesteps
# that entirely since the build context is streamed to the daemon, not path-
# resolved on the host.
# ==============================================================================
FROM postgres:16-alpine

COPY sprint8/db/schema.sql /docker-entrypoint-initdb.d/01-schema.sql
COPY sprint8/db/seed-data.sql /docker-entrypoint-initdb.d/02-seed-data.sql
