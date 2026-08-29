# VortexPanel  PHP Webshell Scanner False Negative

**Target:** VortexPanel v3.4.9
**Type:** Self-hosted server control panel (Python/Flask, runs as root)
**Status:** Reported to maintainer, fixed in v3.4.11
**Date:** August 2026

## Summary

VortexPanel includes a PHP Webshell Scanner that explicitly advertises detection
of "known webshell patterns" including reverse-shell patterns, dynamic function
calls with user input, and obfuscation markers. Testing found that the scanner
fails to flag a PHP file containing a hardcoded reverse shell payload using
`exec()`, producing a false negative.

On a standard VortexPanel deployment which runs as root and uses Apache/Nginx
to serve panel-hosted websites successful exploitation of this gap results in
full system compromise. The scanner's false negative creates false confidence:
an administrator relying on the scanner to vet uploaded content would believe a
malicious file was clean.

## System Overview

VortexPanel is a free, open-source Linux server control panel supporting
Nginx/Apache/OpenLiteSpeed, WAF, Fail2Ban, load balancing, and a WordPress
toolkit. It is designed to run as root for system-level management and listens
on port 8888 by default.

Relevant data entry points:
- Website creation through the panel (file uploads to the hosted site's web root)
- User credentials (login)
- Server management tab

## Methodology

**Trust boundary identified:** the panel's PHP Webshell Scanner is trusted by
the administrator to certify that hosted PHP files are free of malicious code.
This is a signature-based scanner checking for known patterns (`eval(base64_decode())`,
`system($_GET)`, `preg_replace` with the `/e` modifier, among others).

**Hypothesis:** signature-based detection is inherently limited to its known
pattern list. A payload using a function present on the list (`exec()`) but in
a form not matching the specific tracked pattern (hardcoded arguments rather
than dynamic user input via `$_GET`) may bypass detection entirely.

**Test procedure:**
1. Created a website through the panel and hosted a simple PHP file upload page.
2. Simulated a live web server locally to serve the hosted directory.
3. Uploaded a PHP file containing a reverse shell payload using `exec()` with a
   hardcoded `/bin/bash` command over TCP (no `$_GET` or other dynamic input
   involved).
4. Ran the panel's PHP Webshell Scanner against the hosted directory.
5. Accessed the uploaded file through the web server while listening for an
   incoming connection.

**Result:** the scanner reported no findings. The payload executed successfully,
returning a working reverse shell.

## Root Cause

The scanner's pattern matching targets dynamic-input execution patterns
(e.g. `system($_GET)`) but does not flag `exec()`, `shell_exec()`, `passthru()`,
`popen()`, or `proc_open()` calls when the arguments are hardcoded rather than
sourced from user input. A hardcoded reverse shell requires no visible dynamic
input and does not match any pattern on the scanner's list.

## Impact

- **Severity:** Critical (as run on a standard root deployment)
- An attacker able to place a file in a panel-managed web directory (e.g. via
  an unrelated file upload vulnerability in a hosted application, or direct
  panel access) can bypass the panel's own security scanning.
- Given Vortex Panel's documented root execution model, a served web shell results
  in full system compromise.

## Suggested Fix

Flag any `exec()`, `shell_exec()`, `passthru()`, `popen()`, or `proc_open()`
call regardless of whether the arguments are dynamic or hardcoded, rather than
limiting detection to patterns involving direct user input.

## Disclosure Timeline

- Reported to maintainer via GitHub issue with private technical follow-up.
- Maintainer acknowledged and requested reproduction details.
- Fix released in v3.4.11; retested and confirmed the scanner now correctly
  flags the payload.

## Lessons

Signature-based security controls are only as strong as their pattern coverage.
A control that explicitly claims to detect a vulnerability class (reverse
shells) can still miss variants outside its tested pattern set, and the
presence of the control can create false confidence that is worse than having
no control at all.
