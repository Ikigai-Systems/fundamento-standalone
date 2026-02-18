# Fundamento - Strong foundation for your internal collaboration

<p align="center">
  <a href="https://fundamento.it" target="_blank" align="center">
    <img src="https://res.cloudinary.com/fundamento/image/upload/v1734469016/fundamento_banner.webp" width="900" alt="Fundamento Banner">
  </a>
  <br>
</p>

---

## Quick Start

1. Clone this repository:
   ```
   git clone https://github.com/Ikigai-Systems/fundamento-standalone.git
   cd fundamento-standalone
   ```

2. (Optional) Edit `env.standalone` to customize the initial admin account.
   Defaults are `john@fundamento.it` / `secret!`.

3. Start Fundamento:
   ```
   docker compose up
   ```

4. Open `http://localhost:3333` and log in.

That's it — credentials are generated automatically on first boot.

---

## Architecture

Fundamento runs as a set of Docker containers:

| Service | Purpose |
|---------|---------|
| **website** | The main web application (Rails) |
| **jobs** | Background job worker (GoodJob) |
| **postgresql** | Database (PostgreSQL 16) |
| **redis** | Caching and real-time features |

---

## Configuration

### Admin account (`env.standalone`)

Edit before first start to set your admin credentials:

| Variable | Default |
|----------|---------|
| `FUNDAMENTO_ORGANIZATION` | Fundamento |
| `FUNDAMENTO_ADMIN_EMAIL` | john@fundamento.it |
| `FUNDAMENTO_ADMIN_FIRST_NAME` | John |
| `FUNDAMENTO_ADMIN_LAST_NAME` | Doe |
| `FUNDAMENTO_ADMIN_PASSWORD` | secret! |

### Environment variables (`.env`)

Copy `.env.example` to `.env` to customize:

| Variable | Default | Description |
|----------|---------|-------------|
| `RAILS_PORT` | 3333 | Web server port |
| `HTTP_HOST` | localhost:3333 | Public hostname for links and emails |
| `RAILS_LOG_LEVEL` | info | Log verbosity (debug/info/warn/error) |
| `FUNDAMENTO_VERSION` | latest | Pin a specific image version |

---

## Updating

Pull the latest images and restart:

```
docker compose pull
docker compose up -d
```

Database migrations run automatically on startup.

To pin a specific version, set `FUNDAMENTO_VERSION` in your `.env` file.

---

## Email (SMTP)

By default, Fundamento does not send emails. To enable email delivery,
create a credentials file with your SMTP settings:

```
docker compose run --rm website bin/rails credentials:edit -e standalone
```

This opens the Nano editor. Add an `smtp` block:

```yaml
smtp:
  user_name: your_username
  password: your_password
  address: smtp.example.com
  port: 587
  authentication: login
```

Press `Ctrl-X`, then `Y` to save and exit. Restart to apply:

```
docker compose restart website jobs
```

---

## Troubleshooting

### Port 3333 already in use

Set a different port in `.env`:
```
RAILS_PORT=3334
HTTP_HOST=localhost:3334
```

### Viewing logs

```
docker compose logs -f           # all services
docker compose logs -f website   # web application only
```

### Resetting everything

To start fresh (this destroys all data):
```
docker compose down -v
docker compose up
```

### Re-seeding admin account

After changing `env.standalone`, re-seed without losing other data:
```
docker compose exec website bin/rails db:seed:replant
```
**Warning:** This resets all data in the database.

---

## Documentation

https://docs.fundamento.it

---

<p align="center">
<a href="https://fundamento.it">Fundamento</a> &bull;
<a href="https://docs.fundamento.it">Docs</a> &bull;
<a href="https://fundamento.it/pricing">Pricing</a> &bull;
<a href="https://fundamento.it/terms">Terms</a> &bull;
<a href="https://fundamento.it/privacy">Privacy</a>
</p>
