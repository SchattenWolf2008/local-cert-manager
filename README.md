# local-cert-manager
A vibe-coded utility to easily create LOCAL certificates. These certificates are intended for localhost setups only and are not usable (unless trusted) over the internet. For that, use certbot. This utility is aimed for local domains that dont exist on the internet; or to hijack existing domains for localhost purposes. (In my case local AI)

KEEP IM MIND: This script was generated using AI and specifically crafted prompts. I have tested it a bit and let an AI agent test it, but obviously mistakes can (and are) there. A list of known issues are written below; otherwise USE THIS AT YOUR OWN RISK.
Feel free to copy or do whatever with this, I dont really care.


## Security model

This tool creates private certificate authorities and server certificates for local HTTPS development. It must be run as root because it manages certificate files under `/etc/nginx`, modifies `/etc/hosts`, updates the Debian/Ubuntu system trust store, and reloads Nginx.

Newly created bundle CAs are automatically installed into the system trust store. Anyone who obtains a bundle's `ca.key` can issue certificates trusted by systems that trust that CA. The CA private key must therefore never be shared, copied to clients, or committed to source control.

Clients only need the public `ca.crt` file. They must never receive `ca.key` or `server.key`.

This tool is intended for root-controlled personal or development systems. It assumes that `/etc/nginx/certificates` and its existing bundle directories cannot be modified by untrusted users.

## Known limitations and quirks

### Certificate behavior

* Each bundle has its own private root CA.
* New bundle CAs are trusted system-wide automatically.
* CA certificates are valid for 20 years, and server certificates are valid for 10 years.
* Private keys are stored unencrypted but protected with root-only filesystem permissions.
* Every entered DNS name receives both an exact SAN and a wildcard SAN. For example, `example.test` produces `example.test` and `*.example.test`.
* Wildcards only match one additional label. `*.example.test` matches `app.example.test`, but not `deep.app.example.test`.
* Only DNS names are supported. IP-address SANs are not supported.
* Internationalized domain names are not converted automatically. Punycode must be entered manually.
* No certificate revocation list or OCSP service is provided. If a CA key is compromised, remove the CA from trust and recreate the bundle.
* `server-fullchain.crt` contains the leaf certificate followed by the bundle root CA. Sending the root CA from a TLS server is normally unnecessary, but is retained for convenient local configuration.

### System trust

* `update-ca-certificates` updates the Debian/Ubuntu system trust store.
* Some browsers, Java applications, containers, and other software may use separate trust stores and may require the CA to be imported separately.
* Trusting a CA on the host does not automatically trust it on other computers, virtual machines, containers, phones, or browsers with independent certificate stores.
* Trust and untrust actions outside bundle deletion are not fully transactional. If `update-ca-certificates` fails after the CA file has been copied or removed, manual cleanup may be necessary.

### `/etc/hosts` management

* `/etc/hosts` supports exact names only. Certificate wildcards cannot be represented there.
* Managed sections use exact `BEGIN` and `END` marker lines. These marker lines should not be manually renamed, indented, duplicated, or partially removed.
* The script refuses to modify `/etc/hosts` if its managed markers are malformed, nested, duplicated, or incomplete.
* A backup is created before every actual `/etc/hosts` write. Backups are not pruned automatically and can accumulate over time.
* Hosts backups may contain private local hostnames and IP addresses.
* If multiple bundles select the same hostname, the first bundle alphabetically owns the rendered hosts entry.
* If the owning bundle is deleted, another bundle selecting the same hostname automatically takes ownership.
* If a hostname already exists outside the managed blocks, the external entry is left unchanged and takes precedence over the managed selection.
* When managed blocks exist, trailing blank lines in the unmanaged part of `/etc/hosts` are normalized to produce stable reconciliation results.
* The script has no global lock. Running multiple instances simultaneously can cause races between certificate, trust-store, and hosts-file operations.

### IP-address validation

* When Python 3 is installed, IP addresses are validated using Python's `ipaddress` module.
* Without Python 3, the IPv4 fallback is strict, but the simplified IPv6 fallback may accept malformed IPv6-like strings. This is a known limitation.

### Nginx integration

* Nginx is reloaded directly with `systemctl reload nginx` after certificate updates.
* The script intentionally does not run `nginx -t` before reloading.
* A failed reload does not undo the newly written certificate files.
* Bundle deletion does not modify Nginx configuration files.
* Bundle deletion does not reload Nginx, because an existing configuration may still reference the deleted certificate paths.
* The script searches `/etc/nginx` and warns when configuration files appear to reference a bundle being deleted.
* If the script itself is stored under `/etc/nginx` under a filename other than `local-cert-manager.sh`, it may appear in the reference warning if it contains a matching bundle path.

### Deletion and rollback

* Bundle deletion stages `/etc/hosts`, the trusted CA, and the bundle directory before committing deletion.
* Most failures restore the previous hosts file and trusted CA.
* A very narrow multi-failure case remains: if staging the bundle directory fails and the subsequent trust-store rollback also fails, the script may exit before restoring the previous hosts file. The bundle directory and trusted CA file remain recoverable, but manual hosts restoration may be required from the backup shown by the script.
* If cleanup of the private deletion staging directory fails after the commit point, the bundle remains deleted but recoverable staged data may remain under a hidden `.delete-*` directory.

### Filesystem assumptions

* Existing bundles are assumed to be root-owned and protected from modification by untrusted users.
* The script does not comprehensively reject symbolic links or verify the ownership and permissions of every existing bundle file.
* Certificate files are replaced individually rather than as one atomic directory transaction. A rare disk or filesystem failure during installation could leave certificate output files from different issuance attempts.
* Do not use the tool where untrusted users can modify `/etc/nginx/certificates`.

### Platform requirements

* Intended for Debian or Ubuntu systems.
* Requires Bash, OpenSSL, `update-ca-certificates`, systemd, Nginx, GNU core utilities, GNU `find`, `awk`, `sed`, and related standard tools.
* `nano` is optional and is only required for the manual hosts-file editor option.
* The automatic privilege escalation expects the script to be executable. Alternatively, run it explicitly with `sudo bash local-cert-manager.sh`.
* Systems without systemd or `update-ca-certificates` are not supported.

### What are bundles?
"Bundles" describe a folder containing a certificate / ca file(s). It may contain multiple domain names in the same certificate file, meaning you dont have to use different files in nginx configuration for multiple domain names. So bundle is just a different name for directory / the certificate file.
