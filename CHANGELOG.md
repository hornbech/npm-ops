# Changelog

## [Unreleased]

### Pending
- Lock port back to `127.0.0.1:7880` once DNS for `stats.hhornbech.dk` propagates
- Create NPM proxy host with SSL + basic auth access list

---

## [0.1.0] - 2026-04-18

### Added
- GoAccess container (`xavierh/goaccess-for-nginxproxymanager`) for monitoring Nginx Proxy Manager request logs
- `docker-compose.yml` with read-only mount of NPM log directory (`/home/jhh/npm/data/logs`)
- `PROCESSING_DATES=true` for per-day click-to-filter in the dashboard
- `KEEP_LAST=30` retaining 30 days of log history
- Design document and implementation plan in `docs/plans/`

### Configuration
- Port `7880` bound to all interfaces (`0.0.0.0`) for temporary LAN access during DNS propagation
- Timezone set to `Europe/Copenhagen`
- Dashboard auto-refreshes every 10 seconds
