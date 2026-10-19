# Ubuntu repositories on Red Hat Satellite 6.18 and 6.19

Use this procedure to sync Ubuntu package repositories into Red Hat Satellite and install packages from that content on an existing Ubuntu host.

It was exercised on Satellite 6.19.5 with Ubuntu 26.04 (`resolute`), `amd64`. The same repository fields and client flow apply to Satellite 6.18.5. Differences between the two Satellite versions are called out where they change a command.

Example names used below:

| Placeholder | Example |
|---|---|
| Satellite FQDN | `satellite.example.com` |
| Organization | `Acme` |
| Ubuntu host | `ubuntu01.example.com` |
| Content view | `CV-Ubuntu2604` |
| Activation key | `ak-ubuntu2604` |

Substitute your own names. The organization value passed to `subscription-manager` is the organization **label**. In a default install the label matches the name.

## Support boundary

Satellite can store and serve `.deb` repositories. The plugin that does this, `pulp_deb`, is shipped in the Satellite repository and is turned off by default.

Red Hat supports Satellite content management, host registration, and provisioning for Red Hat Enterprise Linux. The [Satellite 6.18 planning guide](https://docs.redhat.com/en/documentation/red_hat_satellite/6.18/html/overview_concepts_and_deployment_considerations/supported-usage-and-versions-of-project-components_planning) and the [Satellite 6.19 planning guide](https://docs.redhat.com/en/documentation/red_hat_satellite/6.19/html/overview_concepts_and_deployment_considerations/supported-usage-and-versions-of-project-components_planning) state that Satellite includes components to provision and configure other operating systems, and that Red Hat supports that usage for Red Hat Enterprise Linux. The content-management tables list RHEL only.

Treat Ubuntu sync, `apt`, and the Ubuntu client packages as outside the Satellite support matrix. A failure in those areas is not an SLA-backed Satellite case. The client package is the community `subscription-manager` build published by ATIX for Foreman and Katello. It is not the Red Hat Satellite Client repository.

Do not install `pulp_deb` from PyPI or a third-party RPM. Use the package in the Satellite repository, installed by `satellite-installer`.

## What gets synced

Ubuntu publishes several pockets. Each pocket is a separate Satellite repository. One repository must contain one suite. Do not put `resolute` and `resolute-updates` in the same repository.

| Pocket | Upstream URL | Releases / Distributions | What it is |
|---|---|---|---|
| Release | `https://archive.ubuntu.com/ubuntu/` | `resolute`, `noble`, or `jammy` | Packages from the day that Ubuntu release was published |
| Updates | `https://archive.ubuntu.com/ubuntu/` | `resolute-updates`, `noble-updates`, or `jammy-updates` | Bug fixes, and security fixes after they are copied here |
| Security | `https://security.ubuntu.com/ubuntu/` | `resolute-security`, `noble-security`, or `jammy-security` | Security fixes. This is a different server from the archive |

The release pocket alone does not deliver ongoing security updates. A host that should track Ubuntu security updates needs the release, updates, and security repositories.

**Components** select the archive area:

| Component | When to sync it |
|---|---|
| `main` | Always. Officially supported free software. |
| `universe` | When hosts install community packages. A default Ubuntu image enables this. |
| `restricted` | When hosts need proprietary firmware or drivers. |
| `multiverse` | Rarely. Licensing and support differ from `main`. |

**Architecture** `amd64` is 64-bit x86. Set this when the Ubuntu hosts are x86_64. Satellite then also syncs architecture-independent (`all`) packages. **No restriction** downloads every architecture Ubuntu publishes, including `arm64` and `i386`.

Prefer **Download policy: On demand**. Satellite stores the repository indexes during the sync and downloads a `.deb` when a host requests it. **Immediate** downloads every package during the sync. One `main` pocket is thousands of packages and can consume tens of gigabytes on `/var/lib/pulp`.

Do not add a content-view package filter to these repositories for the first deployment. A filter can republish a structured Ubuntu repository as a flat repository and break clients and Capsules.

## Requirements

- Red Hat Satellite 6.18.5 or 6.19.5 on RHEL 9.
- The Satellite server repository enabled on the Satellite host. The installer uses that repository to install `python3.12-pulp-deb`. That RPM is built for the Python 3.12 interpreter Satellite already uses. Do not install Pulp's Python packages from AppStream or PyPI.

  ```bash
  subscription-manager refresh
  subscription-manager repos --enable satellite-6.19-for-rhel-9-x86_64-rpms
  ```

  On Satellite 6.18, enable `satellite-6.18-for-rhel-9-x86_64-rpms` instead.

  If `subscription-manager` reports that the repository ID is not valid, the account this host is registered to does not provide **Red Hat Satellite 6.18** or **6.19 for RHEL 9 x86_64 (RPMs)**. Maintenance, Utils, and Client repositories are not a substitute. Refresh cannot create a repository the account does not own.

- Free space on the Pulp volume, normally `/var/lib/pulp`.
- DNS from the Ubuntu host to the Satellite FQDN.
- TCP 443 from the Ubuntu host to Satellite.
- A maintenance window. `satellite-installer` reconfigures Satellite and restarts its services.

On a Capsule that must serve this content, repeat the installer change in [Enable Debian content](#enable-debian-content) on that Capsule. This procedure was validated on a Satellite server with no external Capsule.

## Enable Debian content

Satellite's scenario file forces Debian content off. That setting is in `/usr/share/foreman-installer/config/foreman.hiera/scenario/satellite/scenario-overrides.yaml`:

```yaml
foreman_proxy_content::enable_deb: false
```

That file is higher priority than the installer answers file. Passing only `--foreman-proxy-content-enable-deb true` does not win. Put the override in custom Hiera, which is read first.

### CLI

```bash
grep -q '^foreman_proxy_content::enable_deb:' /etc/foreman-installer/custom-hiera.yaml \
  || printf '\nforeman_proxy_content::enable_deb: true\n' >> /etc/foreman-installer/custom-hiera.yaml

satellite-installer --help | grep enable-deb
satellite-installer --foreman-proxy-content-enable-deb true
```

If the help text also lists `--katello-enable-deb`, pass it as well:

```bash
satellite-installer \
  --foreman-proxy-content-enable-deb true \
  --katello-enable-deb true
```

Satellite 6.19.5 has the content-proxy flag. The Katello flag is not present on that version. The custom Hiera line is required on both 6.18 and 6.19.

The installer runs for several minutes. Wait until it prints `Success!` and the Satellite URL.

Confirm the plugin RPM is installed:

```bash
rpm -q python3.12-pulp-deb
```

The version differs by Satellite release and errata. The package must be present. `rubygem-pulp_deb_client` alone is not enough. That gem can be installed while the Pulp plugin is still absent.

There is no web UI switch for this. It is an installer setting.

## Create the product and repositories

Create one custom product, then one repository per pocket.

### Web UI

1. Go to **Content > Products > Create Product**.
2. Enter a name, for example `Ubuntu 26.04`, and save.
3. Open the product and click **New Repository**.
4. Use the values in the table for that Ubuntu release. Set **Type** to `deb` and **Download Policy** to `On demand`.
5. Repeat **New Repository** for the updates pocket and the security pocket.
6. Select each repository and click **Sync Now**. Wait for each task under **Monitor > Tasks**.

Ubuntu 26.04 (`resolute`):

| Name | Upstream URL | Releases / Distributions | Components | Architectures |
|---|---|---|---|---|
| `Ubuntu 26.04 resolute main amd64` | `https://archive.ubuntu.com/ubuntu/` | `resolute` | `main` | `amd64` |
| `Ubuntu 26.04 resolute-updates main amd64` | `https://archive.ubuntu.com/ubuntu/` | `resolute-updates` | `main` | `amd64` |
| `Ubuntu 26.04 resolute-security main amd64` | `https://security.ubuntu.com/ubuntu/` | `resolute-security` | `main` | `amd64` |

Ubuntu 24.04 (`noble`):

| Name | Upstream URL | Releases / Distributions | Components | Architectures |
|---|---|---|---|---|
| `Ubuntu 24.04 noble main amd64` | `https://archive.ubuntu.com/ubuntu/` | `noble` | `main` | `amd64` |
| `Ubuntu 24.04 noble-updates main amd64` | `https://archive.ubuntu.com/ubuntu/` | `noble-updates` | `main` | `amd64` |
| `Ubuntu 24.04 noble-security main amd64` | `https://security.ubuntu.com/ubuntu/` | `noble-security` | `main` | `amd64` |

Ubuntu 22.04 (`jammy`):

| Name | Upstream URL | Releases / Distributions | Components | Architectures |
|---|---|---|---|---|
| `Ubuntu 22.04 jammy main amd64` | `https://archive.ubuntu.com/ubuntu/` | `jammy` | `main` | `amd64` |
| `Ubuntu 22.04 jammy-updates main amd64` | `https://archive.ubuntu.com/ubuntu/` | `jammy-updates` | `main` | `amd64` |
| `Ubuntu 22.04 jammy-security main amd64` | `https://security.ubuntu.com/ubuntu/` | `jammy-security` | `main` | `amd64` |

A sync of `main` for one current release is on the order of 6,000 packages. **On demand** keeps the package files off disk until a host requests them.

A `404` for `dists/<suite>/main/binary-amd64/Packages` in the Pulp worker log is expected. Ubuntu publishes `Packages.gz` and `Packages.xz`, not an uncompressed `Packages` file. Pulp logs the miss and continues.

### CLI

```bash
hammer product create \
  --name "Ubuntu 26.04" \
  --organization "Acme"

hammer repository create \
  --name "Ubuntu 26.04 resolute main amd64" \
  --organization "Acme" \
  --product "Ubuntu 26.04" \
  --content-type deb \
  --url "https://archive.ubuntu.com/ubuntu/" \
  --deb-releases resolute \
  --deb-components main \
  --deb-architectures amd64 \
  --download-policy on_demand

hammer repository create \
  --name "Ubuntu 26.04 resolute-updates main amd64" \
  --organization "Acme" \
  --product "Ubuntu 26.04" \
  --content-type deb \
  --url "https://archive.ubuntu.com/ubuntu/" \
  --deb-releases resolute-updates \
  --deb-components main \
  --deb-architectures amd64 \
  --download-policy on_demand

hammer repository create \
  --name "Ubuntu 26.04 resolute-security main amd64" \
  --organization "Acme" \
  --product "Ubuntu 26.04" \
  --content-type deb \
  --url "https://security.ubuntu.com/ubuntu/" \
  --deb-releases resolute-security \
  --deb-components main \
  --deb-architectures amd64 \
  --download-policy on_demand

hammer repository synchronize \
  --name "Ubuntu 26.04 resolute main amd64" \
  --organization "Acme" \
  --product "Ubuntu 26.04"

hammer repository synchronize \
  --name "Ubuntu 26.04 resolute-updates main amd64" \
  --organization "Acme" \
  --product "Ubuntu 26.04"

hammer repository synchronize \
  --name "Ubuntu 26.04 resolute-security main amd64" \
  --organization "Acme" \
  --product "Ubuntu 26.04"
```

Change `--deb-releases` and the names for `noble` or `jammy`. Confirm each sync before you publish a content view:

```bash
hammer repository info \
  --name "Ubuntu 26.04 resolute main amd64" \
  --organization "Acme" \
  --product "Ubuntu 26.04"
```

`Sync Status` must be `Success`, and **Debian Packages** must be greater than zero.

## Content view

Hosts only see Debian repositories that are in the content view they are registered to. The Library copy of a product is not enough.

### Web UI

1. Go to **Content > Lifecycle > Content Views > Create content view**.
2. Name it `CV-Ubuntu2604`.
3. Add the three Ubuntu 26.04 repositories.
4. Click **Publish** and wait until the task succeeds. Version 1.0 is in the **Library** environment.

### CLI

```bash
hammer content-view create \
  --name "CV-Ubuntu2604" \
  --organization "Acme"

hammer content-view add-repository \
  --name "CV-Ubuntu2604" \
  --organization "Acme" \
  --product "Ubuntu 26.04" \
  --repository "Ubuntu 26.04 resolute main amd64"

hammer content-view add-repository \
  --name "CV-Ubuntu2604" \
  --organization "Acme" \
  --product "Ubuntu 26.04" \
  --repository "Ubuntu 26.04 resolute-updates main amd64"

hammer content-view add-repository \
  --name "CV-Ubuntu2604" \
  --organization "Acme" \
  --product "Ubuntu 26.04" \
  --repository "Ubuntu 26.04 resolute-security main amd64"

hammer content-view publish \
  --name "CV-Ubuntu2604" \
  --organization "Acme"
```

When you add pockets later, add the repositories to this same content view and publish again. Hosts registered to **Library / CV-Ubuntu2604** receive the new version. Do not create a second content view for updates and security.

## Activation key

### Web UI

1. Go to **Content > Lifecycle > Activation Keys > Create Activation Key**.
2. Name it `ak-ubuntu2604`.
3. Set **Lifecycle environment** to `Library` and **Content view** to `CV-Ubuntu2604`.
4. Save.
5. If the key has a **Subscriptions** tab and the Ubuntu product is listed, add that subscription. Satellite 6.19 in Simple Content Access does not attach subscriptions on the key. The content view is what grants the repositories.
6. If the key has a **Repository sets** tab, override each Ubuntu repository to **Enabled**. Custom Debian repositories are disabled on the host until something enables them.

### CLI

```bash
hammer activation-key create \
  --name "ak-ubuntu2604" \
  --organization "Acme" \
  --lifecycle-environment "Library" \
  --content-view "CV-Ubuntu2604"
```

On Satellite 6.19.5, `hammer activation-key` has no `add-subscription` or `subscriptions` subcommand. Enabling is done on the host, in the next section, unless the web UI still shows repository-set overrides.

## Prepare the Ubuntu host

### Python on the Ubuntu host

`subscription-manager` is built against the system `python3` of one Ubuntu release. It will not install on a newer or older release. Do not add a second Python, a `deadsnakes` PPA, or `python3` from another Ubuntu suite to satisfy the dependency. Use the client repository that matches `python3 --version`.

| Ubuntu release | System Python | `python3-subscription-manager` requires |
|---|---|---|
| 22.04 (`jammy`) | 3.10 | `python3 (>= 3.10)` and `python3 (<< 3.11)` |
| 24.04 (`noble`) | 3.12 | `python3 (>= 3.12)` and `python3 (<< 3.13)` |
| 26.04 (`resolute`) | 3.14 | `python3 (>= 3.14)` and `python3 (<< 3.15)` |

Check the host before you add a client repository:

```bash
python3 --version
lsb_release -rs
```

Ubuntu 26.04 reports `Python 3.14.x`. The 24.04 client then fails with:

```text
python3-subscription-manager : Depends: python3 (< 3.13) but it is not installable
```

The same mismatch happens if a 22.04 client is used on 24.04, or a 26.04 client is used on 24.04. `apt-get install subscription-manager` also pulls `apt-transport-go-katello` and `python3-subscription-manager` from that same client repository. Those packages are not in the Ubuntu archive.

### Client repository

| Ubuntu release | Client repository |
|---|---|
| 26.04 (`resolute`) | `https://oss.atix.de/Ubuntu26LTS/` |
| 24.04 (`noble`) | `https://oss.atix.de/Ubuntu24LTS/` |
| 22.04 (`jammy`) | `https://oss.atix.de/Ubuntu22LTS/` |

The ATIX signing key fingerprint is `CA56F29E00060B63D0890D0F5BD96651DF50FFFB`. The key is published at `https://oss.atix.de/atix_gpg.pub`.

On the Ubuntu host, as root, for 26.04:

```bash
curl --silent --show-error \
  --output /etc/apt/trusted.gpg.d/atix-client.asc \
  https://oss.atix.de/atix_gpg.pub

cat > /etc/apt/sources.list.d/foreman-client.sources << 'EOF'
Types: deb
URIs: https://oss.atix.de/Ubuntu26LTS/
Suites: stable
Components: main
Architectures: amd64
Signed-By: /etc/apt/trusted.gpg.d/atix-client.asc
EOF

apt-get update
apt-get install -y subscription-manager
```

Change the `URIs` line for 24.04 or 22.04. Confirm the host can resolve `satellite.example.com` before you register it.

## Register the host

### Web UI

1. Go to **Hosts > Register Host**.
2. On the **General** tab, select the activation key `ak-ubuntu2604`.
3. If **Insecure** is available, enable it for the first registration. The host does not yet trust Satellite's Katello CA, so the first `curl` fails with `SSL certificate problem: self-signed certificate in certificate chain` until you do this.
4. On the **Advanced** tab, set **Setup Red Hat Lightspeed** to **No**. The registration script would otherwise try to install `insights-client`, which is not an Ubuntu package.
5. Leave **Install packages** empty. `subscription-manager` is already installed.
6. Leave **Update packages** unchecked.
7. Leave **Repositories** at **0 set**. Those entries are extra repositories for the script itself. The activation key supplies the Ubuntu content.
8. Leave **Force** and **Ignore errors** unchecked on a first registration.
9. Click **Generate** and run the printed command on the Ubuntu host.

If you already copied a command that fails the TLS check, add `--insecure` to `curl` and `&insecure=true` to the query string. Keep the original URL and bearer token. A command that still contains `...` is not valid and returns `ERROR: unauthorized`. Registration tokens expire after the lifetime shown on the form, four hours by default. Generate a new command if the token is rejected.

A successful run registers the host and lists the Ubuntu repositories in `/etc/apt/sources.list.d/rhsm.sources`. New custom repositories are usually `Enabled: 0`.

### Enable the repositories on the host

```bash
subscription-manager repos --list
subscription-manager repos \
  --enable REPO_ID_FROM_THE_LIST
```

Repeat `--enable` for each Ubuntu repository ID. Then confirm:

```bash
subscription-manager repos --list
```

Every Ubuntu repository the host should use shows `Enabled: 1`.

## Point apt at Satellite

The host still has Ubuntu's own archive entries. Until those are moved aside, `apt` keeps using `archive.ubuntu.com` as well as Satellite.

```bash
mv /etc/apt/sources.list /etc/apt/sources.list.upstream
mkdir -p /etc/apt/sources.list.d/disabled
mv /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/disabled/ 2>/dev/null || true

cat > /etc/apt/apt.conf.d/98phased-updates << 'EOF'
APT::Get::Always-Include-Phased-Updates "true";
EOF

apt update
```

Leave `foreman-client.sources` in place. That repository is only the client tools.

Ubuntu phases some updates so that not every host sees them on the same day. The `apt` setting above turns that off so Satellite's view of the archive is the view the host installs.

## Verify

`apt-cache policy` for a package in `main` must show a `katello://satellite.example.com/...` URL whose path contains the content view name and the suite:

```bash
apt-cache policy bash
```

Install a small package and confirm the `Get:` line uses `katello://`:

```bash
apt install net-tools
```

`apt update` fetches three suites when release, updates, and security are enabled. Ignored `InRelease` and `Release.gpg` lines mean the publication is not signed yet. `apt` still uses the unsigned `Release` file. See [Sign the published repositories](#sign-the-published-repositories) if you want those lines to go away.

The host is listed in **Hosts > All Hosts**.

## Add universe later

Add three more repositories, with component `universe` and the same URLs and suites as `main`. Use **On demand**. Add them to `CV-Ubuntu2604`, publish, then on the host:

```bash
subscription-manager refresh
subscription-manager repos --list
```

Enable any new ID that shows `Enabled: 0`.

Skip `restricted`, `multiverse`, and `*-backports` unless you have a package that comes from those areas.

## Sign the published repositories

Assigning a GPG key on the repository form in the web UI does not sign the `Release` file Satellite publishes. That field is for verifying an upstream repository during sync. Ubuntu 24.04 and 26.04 then print:

```text
Notice: Missing Signed-By in the sources.list(5) entry for 'katello://...'
```

`apt` also ignores `Release.gpg` because the file is not there.

Katello signs every Debian publication on the server when Pulp has a signing service named exactly `katello_deb_sign`. This uses `pulpcore-manager` on the Satellite host. It is part of making Ubuntu content verifiable, and it is outside Red Hat's supported Satellite operations. One service signs all `deb` repositories on that Satellite, not a single product.

The lab procedure uses a key with no passphrase so the Pulp workers can sign without a prompt. Use a passphrase and a signing script that unlocks it for any deployment you consider production.

On the Satellite host, as root:

```bash
sudo -u pulp gpg --batch --pinentry-mode loopback \
  --quick-generate-key "Satellite APT <apt@satellite.example.com>" \
  default default never

sudo -u pulp gpg --list-keys --with-fingerprint "Satellite APT"
```

Copy the 40-character fingerprint, then install the script. Replace `THE_40_CHARACTER_FINGERPRINT` before you run it.

```bash
cat > /usr/local/bin/sign_deb_release.sh << 'EOF'
#!/bin/bash
set -euo pipefail
RELEASE_FILE="$(/usr/bin/readlink -f "$1")"
OUTPUT_DIR="${PULP_TEMP_WORKING_DIR}"
DETACHED_SIGNATURE_PATH="${OUTPUT_DIR}/Release.gpg"
INLINE_SIGNATURE_PATH="${OUTPUT_DIR}/InRelease"
GPG_KEY_ID="THE_40_CHARACTER_FINGERPRINT"
COMMON_GPG_OPTS="--batch --armor --digest-algo SHA256"
/usr/bin/gpg ${COMMON_GPG_OPTS} \
  --detach-sign \
  --output "${DETACHED_SIGNATURE_PATH}" \
  --local-user "${GPG_KEY_ID}" \
  "${RELEASE_FILE}"
/usr/bin/gpg ${COMMON_GPG_OPTS} \
  --clearsign \
  --output "${INLINE_SIGNATURE_PATH}" \
  --local-user "${GPG_KEY_ID}" \
  "${RELEASE_FILE}"
printf '{"signatures":{"inline":"%s","detached":"%s"}}\n' \
  "${INLINE_SIGNATURE_PATH}" "${DETACHED_SIGNATURE_PATH}"
EOF

chmod 755 /usr/local/bin/sign_deb_release.sh
chown root:pulp /usr/local/bin/sign_deb_release.sh

sudo -u pulp PULP_SETTINGS=/etc/pulp/settings.py \
  pulpcore-manager add-signing-service \
  --class deb:AptReleaseSigningService \
  katello_deb_sign \
  /usr/local/bin/sign_deb_release.sh \
  THE_40_CHARACTER_FINGERPRINT
```

Republish the content view so existing versions get signed metadata:

```bash
hammer content-view version republish-repositories \
  --organization "Acme" \
  --content-view "CV-Ubuntu2604" \
  --version "1.0"
```

Publish the public key where the Ubuntu host can read it:

```bash
sudo -u pulp gpg --armor --export "Satellite APT" > /var/www/html/pub/satellite-apt.asc
restorecon -v /var/www/html/pub/satellite-apt.asc
```

On the Ubuntu host, the file name must start with `client`, `pulp_`, or `orcharhino_` and end with `.asc` or `.gpg`. `subscription-manager` only then writes `Signed-By` into `rhsm.sources`. `atix-client.asc` does not match.

```bash
curl --insecure \
  --output /etc/apt/trusted.gpg.d/client.asc \
  https://satellite.example.com/pub/satellite-apt.asc
subscription-manager refresh
apt update
```

`apt update` should retrieve `InRelease` rather than ignoring `Release.gpg`. If a publish fails after the signing service is added, read the Pulp worker log for that task before you register more hosts. A failing script fails the publish.

## Remove the content and free disk space

Deleting a content view removes a reference to the packages. The files stay on `/var/lib/pulp` until nothing else references them and orphan cleanup runs.

1. Delete the content view, including its published versions. In the web UI this is **Content > Lifecycle > Content Views**.
2. Delete the repositories, or delete the product. In the web UI this is **Content > Products**.
3. Wait until those delete tasks succeed under **Monitor > Tasks**.
4. Remove orphans:

```bash
df -h /var/lib/pulp
foreman-rake katello:delete_orphaned_content
```

The rake command only queues a background task. `df` does not change until that task finishes. If you start it while a repository delete is still running, it can find nothing to delete. Run it again after the deletes have succeeded, then check `df -h /var/lib/pulp`.

## Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `satellite-6.19-for-rhel-9-x86_64-rpms` is not a valid repository ID | The registered account does not include the Satellite server product | Add that subscription to the account, then `subscription-manager refresh` and enable the repository |
| Installer reports success but `python3.12-pulp-deb` is absent | Scenario override left Debian content off, or the Satellite repository was unavailable | Set custom Hiera and run the installer again with the repository enabled |
| Repository type `deb` is missing in the UI | The plugin package is not installed | Repeat [Enable Debian content](#enable-debian-content) |
| Sync stays in **Immediate** for a long time | Every `.deb` is being downloaded | Cancel the task, set **On demand**, and sync again |
| `python3-subscription-manager` depends on `python3 (< 3.13)` or another Python bound that is not installable | Client repository does not match `python3 --version` on that host | Use the ATIX repository for that Ubuntu release. Do not install another Python |
| `curl: (60)` self-signed certificate | The host does not trust the Katello CA yet | Regenerate the registration command with **Insecure**, or add `--insecure` and `&insecure=true` |
| `ERROR: unauthorized` | The bearer token was replaced with a placeholder, or it expired | Generate a new registration command and run that output unchanged |
| Repositories listed with `Enabled: 0` | Custom Debian repositories are disabled by default | `subscription-manager repos --enable REPO_ID` |
| `apt` still contacts `archive.ubuntu.com` | The stock source files are still active | Move `sources.list` and `ubuntu.sources` aside |
| `Missing Signed-By` and ignored `Release.gpg` | The publication is unsigned | Optional [signing service](#sign-the-published-repositories) |
| Content view deleted but Pulp usage is unchanged | Orphan cleanup has not finished, or it ran before the delete finished | Run `foreman-rake katello:delete_orphaned_content` again and wait for the task |
