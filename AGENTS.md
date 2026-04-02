# codedb Agent Guidelines

## Review guidelines

- Flag any security issues: injection, file traversal, untrusted input, secret exposure
- Verify that sensitive files (.env, .pem, .key, credentials) are excluded from indexing AND search
- Check that telemetry behavior matches documentation claims
- Flag any regression in benchmark-critical paths (threshold: 10%)
- Treat P1 issues as merge-blocking
- Verify new language parsers handle malformed input gracefully (braces in strings, unterminated comments)
- Check that installer scripts don't execute untrusted code or skip verification

## Security-sensitive areas

- `src/watcher.zig` — file indexing skip lists (secrets must be excluded)
- `src/mcp.zig` — file read/search (path traversal, scope boundaries)
- `src/telemetry.zig` — data collection and transmission (must match docs)
- `src/snapshot.zig` — sensitive file filtering
- `install/install.sh` — binary download and config modification
