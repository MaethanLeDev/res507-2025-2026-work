# CI Failure Diagnosis and Recovery

- **Failed step:** `Run container smoke test`
- **Cause:** The health check URL in `.github/workflows/ci.yml` was changed to `http://localhost:3000/health-broken`, which does not exist in the Fastify application and returns HTTP 404 (`curl: (22) The requested URL returned error: 404`). After 30 attempts, the step exited with code 1 and triggered `Show container diagnostics`.
- **Fix:** Restored the health check URL to `http://localhost:3000/health`, which matches the route defined in `app.js` returning `{"ok": true}`.
