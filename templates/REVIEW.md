# Review Policy

Report defects that can change runtime behavior, security, data integrity, deployment, public API behavior, or tests.

Treat these as blocking:

- Auth, authorization, or tenant isolation regressions.
- Secrets, tokens, request bodies, or personal data written to logs.
- Database migrations that can lose data or break rollback.
- Public API changes without compatibility handling.

Treat style comments as summary-only unless the repository has a written rule for them.

After the first review, suppress new nits and report only defects that matter before merge.
