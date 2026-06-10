# How to generate a bcrypt hash

AWG 2.0 Easy uses bcrypt hashes for the `PASSWORD_HASH` and `PROMETHEUS_METRICS_PASSWORD` environment variables.

## Generate a hash

Run this single command — it uses `node:18-alpine` which is already pulled during image builds:

```sh
docker run --rm node:18-alpine sh -c \
  "npm install bcryptjs --silent && \
   node -e \"require('bcryptjs').hash('YOUR_PASSWORD',12).then(h=>console.log(h))\""
```

Example output:
```
$2b$12$coPqCsPtcFO.Ab99xylBNOW4.Iu7OOA2/ZIboHN6/oyxca3MWo7fW
```

## Using the hash in docker-compose.yml

Dollar signs in bcrypt hashes (`$2b$12$...`) must be escaped as `$$` when set inline in `docker-compose.yml`, because Docker Compose interprets `$` as a variable prefix:

```yaml
# Wrong — Docker Compose will strip the $ signs
- PASSWORD_HASH=$2b$12$coPqCsPtcFO.Ab99xylBNOW4.Iu7OOA2/ZIboHN6/oyxca3MWo7fW

# Correct — each $ escaped as $$
- PASSWORD_HASH=$$2b$$12$$coPqCsPtcFO.Ab99xylBNOW4.Iu7OOA2/ZIboHN6/oyxca3MWo7fW
```

Alternatively, put `PASSWORD_HASH` in a `.env` file — values there are **not** interpolated:

```
# .env
PASSWORD_HASH=$2b$12$coPqCsPtcFO.Ab99xylBNOW4.Iu7OOA2/ZIboHN6/oyxca3MWo7fW
```

Then reference it in `docker-compose.yml`:

```yaml
env_file:
  - .env
```
