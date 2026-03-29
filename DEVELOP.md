# Building PostgREST and Creating a Docker Image

## Prerequisites

- Docker installed and running
- Git

## 1. Clone the Repository

```bash
git clone https://github.com/PostgREST/postgrest.git
cd postgrest
```

## 2. Build a Static Binary via Nix

We use a Nix Docker container to build a fully static binary (no dependencies, smaller image).

### Start the Nix container with your source mounted

```bash
docker run -it -v $(pwd):/workdir nixos/nix
```

### Inside the container

```bash
# Enable Nix flakes
echo "experimental-features = nix-flakes nix-command" >> /etc/nix/nix.conf

# Set up PostgREST binary cache (speeds up build significantly)
nix-env -iA cachix -f https://cachix.org/api/v1/install
cachix use postgrest

# Build static binary
cd /workdir
nix-build --attr postgrestStatic

# Copy binary to workdir
cp result/bin/postgrest /workdir/postgrest

# Verify it is static
ldd /workdir/postgrest
# expected output: not a dynamic executable

exit
```

## 3. Verify the Binary

```bash
ls -lh postgrest       # should be ~21MB
ldd postgrest          # should say: not a dynamic executable
./postgrest --help     # sanity check
```

## 4. Build the Docker Image

```dockerfile
# Dockerfile-Scratch
FROM scratch

COPY postgrest /usr/bin/postgrest

EXPOSE 3000

USER 1000

CMD ["postgrest"]
```

```bash
docker build -t codeflu/postgrest-custom -f Dockerfile-Scratch .
```

## 5. Verify the Image

```bash
docker images codeflu/postgrest-custom
# should be ~21MB
```

## 6. Run the Image

```bash
docker run --rm \
  -e PGRST_DB_URI="postgres://user:pass@host:5432/dbname" \
  -e PGRST_DB_SCHEMAS="public" \
  -e PGRST_DB_ANON_ROLE="anon" \
  -p 3000:3000 \
  codeflu/postgrest-custom
```

## Notes

- The Nix build uses `musl` libc for full static linking — no OS or shared libs needed at runtime
- `cachix use postgrest` pulls prebuilt dependencies, reducing build time from hours to ~15-30 minutes
- The `result` directory created by `nix-build` is a symlink to `/nix/store/` — always copy the binary out before exiting the container
- The `FROM scratch` base image is safe here because the binary is fully static
