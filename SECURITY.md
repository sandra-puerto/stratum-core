# Security Policy

**Repository:** `stratum-core`
**Contact:** contacto@sandrapuerto.com
**Revision:** 1.0.0

---

## 1. Supported Versions

Security updates are provided for the following revisions:

| Version | Supported |
|---|---|
| 1.x | :white_check_mark: |
| < 1.0 | :x: |

The current revision of each component is declared in the header of its
respective `docker-compose.yml` and `README.md`. Consumers are expected
to track the current revision of the repository.

## 2. Reporting a Vulnerability

**Do not report security vulnerabilities through public GitHub issues,
discussions, or any other public channel.**

To report a vulnerability, send an email to:

> **contacto@sandrapuerto.com**

Include the following information in the report:

1. **Type of vulnerability.** A short classification (e.g., privilege
   escalation, information disclosure, denial of service, container
   escape, misconfiguration).
2. **Affected component.** Specify `dmz`, `gateway`, `database`, or the
   repository as a whole.
3. **Affected revision.** The revision number declared in the component's
   header, or the commit hash if the report concerns the repository
   directly.
4. **Description.** A precise description of the issue and the conditions
   under which it manifests.
5. **Reproduction steps.** A minimal, reproducible sequence of actions
   that demonstrates the issue.
6. **Impact assessment.** An evaluation of the potential consequences if
   the issue is left unaddressed.
7. **Suggested remediation.** Optional. Any proposed fix or mitigation.

If the report contains sensitive information, encrypt the email using
the recipient's public key, or arrange a secure channel through an
initial plain email that contains no sensitive details.

## 3. Response Timeline

The following timeline is committed on a best-effort basis:

| Stage | Target |
|---|---|
| Acknowledgment of receipt | Within 48 hours |
| Initial assessment and severity classification | Within 7 days |
| Remediation for critical issues | As soon as feasible |
| Remediation for high severity | Within 30 days |
| Remediation for medium severity | Within 90 days |
| Public disclosure | After a fix is released, coordinated with the reporter |

Timelines may be extended for issues that require coordination with
upstream projects. In such cases, the reporter is informed of the
expected delay and the rationale.

## 4. Disclosure Process

The following process is applied to every confirmed vulnerability:

1. **Report received.** The reporter receives an acknowledgment.
2. **Triage.** The issue is reproduced and its severity is classified.
3. **Remediation.** A fix is developed and validated in a staging
   environment.
4. **Release.** The fix is published as a new revision of the affected
   component.
5. **Disclosure.** A public advisory is issued after the fix is
   available, with credit to the reporter unless anonymity is requested.

Reporters who prefer coordinated disclosure receive advance notice of
the advisory before it is published.

## 5. Scope

### 5.1 In scope

- Configuration errors in the `docker-compose.yml` files of any
  component.
- Multi-organization tenant isolation weaknesses or cross-tunnel leakage
  in `gateway/`.
- Weaknesses in the network topology that allow a container to reach a
  network it is not attached to.
- Capability or permission configurations that exceed the minimum
  required by the upstream image.
- Credential exposure or leakage in the version-controlled
  configuration.
- Weaknesses in the boundary client configuration under `database/nginx/`.
- Vulnerabilities introduced by this repository into a deployment that
  would not exist with a default Docker installation.

### 5.2 Out of scope

- Vulnerabilities in upstream images (`nginx`, `mariadb`, `mongodb`,
  `redis`, `alpine`, `cloudflared`, `jc21/nginx-proxy-manager`). Report
  these to the corresponding upstream project.
- Vulnerabilities in Docker Engine or Docker Compose itself. Report
  these to Docker, Inc.
- Vulnerabilities in Cloudflare infrastructure or the Cloudflare Tunnel
  protocol.
- Social engineering attacks against operators.
- Denial-of-service attacks that require host-level access.
- Issues in host-level configuration (firewall, SSH, kernel) that fall
  outside the scope of this repository.

## 6. Safe Harbor

Security research conducted in good faith against a deployment of
`stratum-core` is considered authorized. The maintainer will not pursue
legal action against researchers who:

- Make a good-faith effort to avoid privacy violations, data
  destruction, and service interruption.
- Interact only with deployments they own, operate, or have explicit
  permission to test.
- Do not exploit a vulnerability beyond the minimum necessary to
  demonstrate its existence.
- Report the vulnerability promptly and privately in accordance with
  section 2.
- Do not disclose the vulnerability publicly before coordination with
  the maintainer.

This safe harbor applies only to research directed at the software
distributed in this repository. It does not extend to third-party
infrastructure, upstream projects, or deployments owned by parties other
than the researcher.

## 7. Recognition

Reporters who responsibly disclose a confirmed vulnerability are
acknowledged in the public advisory that accompanies the fix, unless
they request anonymity. There is no monetary compensation program
associated with this repository.

## 8. Security Best Practices for Operators

Operators deploying `stratum-core` are expected to:

1. Restrict `.env` file permissions to `600` on every component.
2. Never commit `.env` files to version control.
3. Rotate credentials periodically and immediately after any suspected
   compromise.
4. Keep Docker Engine and Docker Compose up to date with security
   patches.
5. Review the release notes of upstream images before applying updates
   in production.
6. Perform pre-update backups of every stateful service.
7. Enable host-level logging and forward it to a central collection
   point.
8. Restrict host-level access to the Docker socket and to the
   repository directory.
9. Monitor the repository for revisions that address security issues.

## 9. References

- `LICENSE` — license terms, including the security reporting
  requirement.
- `README.md` — repository overview.
- `dmz/README.md`, `gateway/README.md`, `database/README.md` — component
  documentation.

---

**Document control**

| Field | Value |
|---|---|
| Repository | `stratum-core` |
| Contact | contacto@sandrapuerto.com |
| Revision | 1.0.0 |
| Status | Production |
| Classification | Internal |