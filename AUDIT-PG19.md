# PostgreSQL 19 wait-event coverage audit

Inspected 2026-11-18. PostgreSQL REL_19_STABLE commit [`882f6b32f4ae42d32d63730985658040e88e7dc0`](https://github.com/postgres/postgres/commit/882f6b32f4ae42d32d63730985658040e88e7dc0). This is a source audit, not fault-injection or latency benchmarking.

## Version and coverage

Upstream has the REL_19_0 tag (`5c839a2a08b9b87a349be97371fac6c5a2873b10`, committed 2026-11-16) and REL_19_STABLE; master identifies as 20devel. The pinned SHA is REL_19_0 plus three commits (JIT hashing, Incremental Sort, `.abi-compliance-history`), none of which touch the files cited here, so every line number below is also valid at REL_19_0. A tag is not a public GA announcement. As checked on 2026-11-18: the latest postgresql.org announcement is "19 RC 2 Released" (2026-11-12), the current docs are labeled "19rc2", the PG19 release notes still say "Release date: 2026-??-??", and the roadmap page says the 19 release "is planned for November 19, 2026". PostgreSQL 19 is therefore not GA yet; this audit uses an exact PG19 stable SHA rather than claiming a release date.

Every category on the site was reviewed: all 9 historical I/O rows, all 32 historical authentication rows (below), both rejected CPU-only categories, logging, and the listed recently-added event names. This is not an exhaustive search for every possible gap in PostgreSQL. The 15 historical compression/cryptographic CPU candidates remain excluded: CPU time is not waiting; their execution time was not benchmarked.

## Count reconciliation

- Historical inventory: 41 blocking candidates (9 I/O + 32 auth) and 15 CPU-only candidates = 56. Do not present 56 as current open gaps.
- Auth: 32 historical rows = 12 potentially blocking uninstrumented rows + 5 false DNS claims + 5 removed RADIUS rows + 10 requiring narrower library/platform validation. The visible reconciliation separates these four statuses; rejected claims and removed features are not presented as current open gaps.
- I/O: 9 historical sites minus 2 COPY sites now instrumented = 7 retained; plus one newly documented relation-creation site = 8. Relation creation and removal apply more broadly than temp tables. This is a call-site inventory, not eight new event names.

## I/O evidence

All paths below are under [the pinned source](https://github.com/postgres/postgres/tree/882f6b32f4ae42d32d63730985658040e88e7dc0).

| Historical item | PG19 evidence | Status |
|---|---|---|
| Standby signal fsync | xlogrecovery.c:1027, in readRecoverySignalFile() | Open; direct pg_fsync, no wait wrapper |
| Recovery signal fsync | xlogrecovery.c:1041 | Open; same |
| Relation first-file unlink | md.c:406 | Open; direct unlink |
| Relation extra-segment unlink | md.c:465 | Open; direct unlink |
| Deferred relation unlink | md.c:1966; sync.c:231 caller | Open; no wrapper at callee or caller |
| POSIX DSM metadata | dsm_impl.c:279 | Open; fstat outside DSM_ALLOCATE wrapper |
| mmap DSM metadata | dsm_impl.c:850 | Open; fstat outside DSM_FILL_ZERO_WRITE wrapper |
| COPY FROM fread | copyfromparse.c:256 | Fixed; WAIT_EVENT_COPY_FROM_READ |
| COPY TO fwrite | copyto.c:627 | Fixed; WAIT_EVENT_COPY_TO_WRITE |
| NEW relation creation | mdcreate() md.c:249 -> PathNameOpenFile() fd.c:1568 -> PathNameOpenFilePerm() fd.c:1579, call at :1611 -> BasicOpenFilePerm() fd.c:1115, open() at :1121/:1123 | Open; O_CREAT/O_EXCL reaches uninstrumented open |

COPY fixed by [e05a24c2](https://github.com/postgres/postgres/commit/e05a24c2d4eab3dd76741dc6e6c18bb0584771c5), committed 2026-02-03. This retires the original two read/write rows, not all possible open/close/process-wait work.

### Temporary tables: precise proposal

Repeated CREATE TEMP TABLE and DROP TABLE can expose uninstrumented relation-file creation/removal, including filesystem metadata stalls. Temp relations are unlinked immediately in mdunlinkfork() (md.c:406 for the first segment, md.c:465 for extra segments) rather than through the deferred permanent-main-fork checkpoint path (md.c:1966, called from sync.c:231). Not counted in the historical inventory, but on the same removal path: for permanent relations mdunlinkfork() first calls do_truncate() (md.c:393, :420, :455), which calls pg_truncate() (fd.c:724) and a bare truncate() with no wait event. Temp relations skip this truncate. Unlike FileTruncate() (DATA_FILE_TRUNCATE), pg_truncate() is not instrumented. Existing DataFileRead/Write/Sync and FileTruncate wrappers are separate; do not claim all temporary-table work is invisible. Catalog modifications, shared invalidation, lock waits and CPU work are also distinct.

Propose narrowly scoped relation-file open/create and removal wait instrumentation after measurements. Consider common file wrappers, retries, errno preservation, failure paths, platform variations, and the difference between instrumented elapsed syscall time and actual blocked time. Do not put one I/O event around an entire DDL statement. No PostgreSQL code patch is included.

## Recently-added section audit

COPY and CommitDelay are present. WAIT_FOR_WAL_WRITE was added and flush waiting extended to standbys by [7a39f43d](https://github.com/postgres/postgres/commit/7a39f43d885b44a78f366aa50535944c4e38bdbb); WRITE/FLUSH/REPLAY moved to IPC by [56fa044d](https://github.com/postgres/postgres/commit/56fa044d15e8b06d84889f06f5def21a62fb86a3). AioUringCompletion, WaitLSN and CheckpointDelayStart/Complete remain present. Correct the displayed WAL summarizer event to WALSummarizer (not WALSummarizerLock). The historical 2025-11-05 LSN date is the committer date (author date Nov3), so it is retained.

GSoC: "Wait Event Coverage Improvements" was on the PostgreSQL GSoC 2026 ideas list (still present in wiki revision [43116](https://wiki.postgresql.org/index.php?title=GSoC_2026&oldid=43116#Wait_Event_Coverage_Improvements), 2026-03-31). The 2026-05-04 revision ([43378](https://wiki.postgresql.org/index.php?title=GSoC_2026&oldid=43378)) adds a Results block ("Projects submitted: 14 ... Google approved: 7") and keeps 7 project sections; the wait-event project is not among them, so the current-page anchor `GSoC_2026#Wait_Event_Coverage_Improvements` no longer resolves. Wiki `index.php?oldid=` URLs return 403 "Resource available only to logged in users" to anonymous readers, so the site links a public Wayback Machine copy of the ideas list from 2026-03-12 (https://web.archive.org/web/20260312164905/https://wiki.postgresql.org/wiki/GSoC_2026#Wait_Event_Coverage_Improvements) instead, and says the idea is not among the listed approved projects.

# PG19 authentication and logging audit

Baseline: PostgreSQL REL_19_STABLE `882f6b32f4ae42d32d63730985658040e88e7dc0`, inspected 2026-11-18. All line references below use this SHA: https://github.com/postgres/postgres/blob/882f6b32f4ae42d32d63730985658040e88e7dc0/ . Source inspection, not runtime fault injection. `auth.c` has no `pgstat_report_wait_start/end` calls; this does not make every call a legitimate wait candidate.

## Main corrections
- Five RADIUS rows are **retired by feature removal**, not fixed by new events. Commit `a1643d40b308911cc725e62d3c5f7904b426aa09` (2026-04-08), https://www.postgresql.org/message-id/E1wAQPK-003XpL-19@gemulon.postgresql.org and https://www.postgresql.org/docs/19/auth-radius.html . Authentication via PAM can still invoke RADIUS externally.
- Five alleged DNS rows actually request numeric addresses. These are false-positive DNS claims, not newly fixed gaps.
- PAM hostname resolution was mislabeled SSPI. It does DNS only with `pam_use_hostname` enabled.
- Connection-startup hostname lookup happens before `InitProcess()` and the shared PGPROC. Do **not** say these backends necessarily appear as active CPU rows in pg_stat_activity. This requires early-process observability design, not merely a new event name.
- The historical total 32 is a count of old table rows, not 32 verified blocking gaps. TLS has two platform-specific code sites in one row; the Ident reverse conversion row actually encompasses two calls. Avoid implying exact physical call-site counts.

## Every historical authentication table row

Here `auth.c` means `src/backend/libpq/auth.c`. Status P = demonstrated potentially blocking operation without dedicated instrumentation (not proof it blocked in a measured workload); X = incorrect DNS claim; R = removed feature; U = needs narrower/library/platform validation, not a confirmed wait gap.

| Old row | PG19 file:line | Status | Assessment |
|---|---|---|---|
| LDAP sslinit Windows 2220 | auth.c:2275 | U | Library/platform-dependent initialization; Microsoft describes connection-block setup and server discovery. Do not assume synchronous TLS here without checking implementation. |
| LDAP init Windows 2222 | auth.c:2277 | U | Initialization, not enough evidence to label mandatory network wait. |
| LDAP domain2hostlist 2268 | auth.c:2323 | P | DNS SRV discovery when server is derived from base DN. |
| LDAP initialize 2320 | auth.c:2375 | U | OpenLDAP explicitly defers connection until first operation; historical network-connect claim incorrect for OpenLDAP. Not a confirmed wait candidate. |
| LDAP init fallback 2339 | auth.c:2394 | U | OpenLDAP init likewise does not open initial connection; other supported libraries require verification. |
| LDAP set_option 2350 | auth.c:2405 | U | Sets LDAP_OPT_PROTOCOL_VERSION; generic “may perform network” is unsubstantiated. |
| LDAP start_tls_s 2363/2365 | auth.c:2418/2420 | P | Synchronous StartTLS on respective platforms. |
| LDAP search bind 2526 | auth.c:2581 | P | Synchronous bind. |
| LDAP search_s 2551 | auth.c:2606 | P | Synchronous search. |
| LDAP get_option 2602 | auth.c:2657 | U | Reads LDAP_OPT_ERROR_NUMBER, no demonstrated network operation. |
| LDAP user bind 2626 | auth.c:2681 | P | Synchronous bind. |
| LDAP get_option 2660 | auth.c:2715 | U | Reads LDAP_OPT_DIAGNOSTIC_MESSAGE; no demonstrated network operation. |
| Ident reverse lookup 1686–1689 | auth.c:1741–1748 | X | NI_NUMERICHOST and NI_NUMERICSERV on remote AND local conversions; not reverse DNS. |
| Ident remote forward lookup 1704 | auth.c:1750–1759 | X | AI_NUMERICHOST, numeric ident port. |
| Ident local forward lookup 1720 | auth.c:1767–1775 | X | AI_NUMERICHOST and no service. |
| Ident socket 1728–1729 | auth.c:1783 | U | Local socket creation; no evidence of remote wait. Do not count as established external-network gap. |
| Ident bind 1744 | auth.c:1799 | U | Local socket bind; no evidence of remote wait. |
| Ident connect 1755–1756 | auth.c:1810 | P | Blocking connect, no dedicated wrapper. |
| Ident send 1776 | auth.c:1831 | P | Blocking send can stall. |
| Ident recv 1793 | auth.c:1848 | P | Blocking receive. |
| RADIUS getaddrinfo 2971 | removed | R | RADIUS removed in PG19. |
| RADIUS bind 3066 | removed | R | RADIUS removed in PG19. |
| RADIUS sendto 3075–3076 | removed | R | RADIUS removed in PG19. |
| RADIUS select 3124 | removed | R | RADIUS removed in PG19. |
| RADIUS recvfrom 3157 | removed | R | RADIUS removed in PG19. |
| Generic DNS 432–435 | auth.c:446–449 | X | NI_NUMERICHOST; rejected-HBA diagnostic, not reverse DNS. |
| Generic DNS 478 | auth.c:492–495 | X | NI_NUMERICHOST; rejected-HBA diagnostic, not pg_ident reverse lookup. |
| Generic DNS / claimed SSPI 2081 | auth.c:2131–2138 | P | Actually PAM; DNS conditional on pam_use_hostname, numeric otherwise. |
| PAM authenticate 2115 | auth.c:2170 | P | External PAM module operation can block. |
| PAM account 2128 | auth.c:2183 | P | External PAM module account operation can block. |
| GSS accept 996 | auth.c:1042–1052 | U | Uninstrumented opaque library call, but routine Kerberos acceptor ticket validation need not contact KDC. Requires mechanism/configuration proof; don't present every GSS login as KDC round-trip. Existing GSS_OPEN_SERVER covers client session setup read, not this call. |
| Connection hostname 206–209 | src/backend/tcop/backend_startup.c:220–223 | P | Reverse DNS when log_hostname=true, but before shared-memory PGPROC setup (BackendInitialize called at :123, InitProcess at :129). |

Reconciled historical rows: **12 P, 5 X, 5 R, 10 U = 32**. P is only potentially blocking, not a runtime duration measurement. Within U, at least OpenLDAP init/initialize and option operations have flawed claimed network justifications. Recommended site: remove old absolute 32-gaps badge, show category status with audit details rather than relabel 12 as an exhaustive new total.

Primary library evidence: OpenLDAP upstream manual https://github.com/openldap/openldap/blob/master/doc/man/man3/ldap_open.3 lines 68–71 and 103–108 explicitly say init/initialize allocate a structure and defer initial connection. Windows source documentation https://learn.microsoft.com/en-us/windows/win32/api/winldap/nf-winldap-ldap_sslinit describes session initialization/discovery, so a Windows-specific investigation remains useful. No library-wide runtime tests were performed.

## Logging observer effect

Still a valid instrumentation concern, but narrow wording:
- Backend `write_pipe_chunks()` writes collector pipe in `src/backend/utils/error/elog.c:3962,3972` without a dedicated event.
- Backend direct-stderr path `write_console()` calls `write(fileno(stderr),...)` at `elog.c:3115` without dedicated event.
- `elog.c:3886–3906` explicitly chooses pipe when logging collector redirection is active, direct stderr otherwise.
- Collector writes actual logfile via `fwrite()` in `src/backend/postmaster/syslogger.c:1137`, also without a dedicated event. Collector `SysLoggerMain` is its idle main-loop wait, not write instrumentation.
- Therefore don't claim each backend writes disk for every query when collector is active. Backends normally write pipe; slow sink can cause pipe backpressure. Heavy logging also consumes legitimate formatting/CPU time; NULL does not establish logging I/O by itself.
- Syslog calls at `elog.c:2916,2918,2928,2930` are similarly opaque and not wrapped. This is additional pathway context, not extra counted historical rows.

No source modifications or external writes were made. Scope was all 32 historical authentication rows and the site's logging claim, not all possible authentication or startup blocking sites (for example HBA hostname resolution outside those old rows is not an exhaustive audited extension).
