# gaps.wait.events Website Specification

## Overview
One-page website documenting the PostgreSQL Wait Events Coverage Gap Analysis project.

## Domain
gaps.wait.events (via Cloudflare Pages)

## Design
- **Theme**: Monospace web design (https://owickstrom.github.io/the-monospace-web/)
- **Typography**: Monospace font throughout
- **Layout**: Single column, content-width constrained
- **Style**: Minimalist, terminal-like aesthetic

## Content Sections

### Header
- Title: "PostgreSQL Wait Events Coverage Gaps"
- Subtitle: "Analysis and improvement proposals for PostgreSQL wait event instrumentation"
- Link to GitHub analysis

### The Problem
- Explain NULL wait_event being labeled as "CPU" in monitoring tools
- This is misleading - NULL means "not instrumented yet" not "CPU working"
- Tools like RDS Performance Insights, PASH Viewer show green/"CPU" for NULL

### Gap Categories
1. **I/O Operations** (Required)
   - COPY FROM/TO PROGRAM pipe operations
   - File operations (unlink, fsync signal files)
   - Dynamic shared memory fstat

2. **Authentication** (Critical - 32 locations)
   - LDAP: 12 operations (DNS SRV, TLS, bind, search)
   - Ident: 8 operations (DNS + TCP)
   - PAM: 2 operations (black box external auth)
   - RADIUS: 5 operations
   - GSSAPI/Kerberos: 1 operation
   - DNS lookups: 3 in auth + connection logging

3. **Compression** (Optional - observability)
   - Gzip, LZ4, Zstd during base backup
   - CPU work but useful to label

4. **Cryptography** (Optional - observability)
   - SCRAM authentication (PBKDF2 iterations)
   - SQL hash functions

### Recently Closed Gap
- **Hash table searches in locking**: Now has wait event
- Commit: https://github.com/postgres/postgres/commit/e05a24c2d4eab3dd76741dc6e6c18bb0584771c5
- Mark with ✅ or "COMPLETED"

### Observer Effect
- Mention that logging can cause observer effects
- Example: `log_min_duration_statement` with high TPS on non-fast disks
- Wait events show NULL during logging I/O - misleading "CPU" attribution

### GSoC 2026
- This project is proposed for Google Summer of Code 2026
- Link: https://wiki.postgresql.org/wiki/GSoC_2026#Wait_Event_Coverage_Improvements

### Resources
- GitHub analysis: https://github.com/NikolayS/postgres/blob/claude/cpu-asterisk-wait-events-01CyiYYMMcFMovuqPqLNcp8T/WAIT_EVENTS_ANALYSIS.md
- Mailing list thread: https://www.postgresql.org/message-id/flat/CAM527d9PkaSj-gNjLZqjJXnqaWTD8kHPtm2Yj8-1Gh_0pTRgDA%40mail.gmail.com

## Technical Requirements
- Single HTML file (index.html)
- Inline CSS (monospace grid-based design)
- No JavaScript required (optional: tiny bit for grid alignment)
- Responsive (shrinks in character-sized steps)
- Deploy to Cloudflare Pages

## Assets
- None required (text-only)
- Optional: ASCII art diagram showing the problem

## Deployment
- GitHub repo: gaps-wait-events
- Cloudflare Pages connected to GitHub
- Custom domain: gaps.wait.events
