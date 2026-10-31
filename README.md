# RHEL Image Templates

This repository contains declarative RHEL 9.6 blueprints for building server
images with **osbuild-composer**. Profiles describe the software blueprint and
include Kickstart instructions for installing RHEL and laying out the disk.
The resulting image can then be tested and deployed as a VM in Proxmox.

The repository provides the blueprints and example Composer commands. 

## Profiles and architecture

```text
profiles/
└── proxmox/
    ├── generic/
    │   ├── rhel96-generic
    │   └── rhel96-generic.toml
    ├── jenkins/
    │   ├── rhel96-jenkins
    │   └── rhel96-jenkins.toml
    └── os-composer/
        ├── rhel96-os-composer
        └── rhel96-os-composer.toml
```

Each role has two blueprint representations: a blueprint file with multiline Kickstart content and a `.toml` file with that content encoded as a
TOML string which describe the profile metadata and installer customization.

The extensionless blueprint is used to generate the `.toml` with `composer-cli blueprints save <blueprint-name>`.

### Common settings

All profiles share the following configuration:

- Hybrid bootloader (`[customizations.bootloader] hybrid = true`), so the
  image boots under both BIOS and UEFI firmware.
- `gb` keyboard layout and `Europe/London` timezone. Can be customized to other timezone if required.
- Non-interactive text install that ejects the installer media on reboot
  (`reboot --eject`).
- A `root` password and a `support` user in the `wheel` group with password encrypted hashes using `python3 -c 'import crypt,getpass;pw=getpass.getpass();print(crypt.crypt(pw) if (pw==getpass.getpass("Confirm: ")) else exit())'`.
- GPT partitioning restricted to `sda` (`ignoredisk --only-use=sda`), a 1 GiB
  XFS `/boot`, and an LVM volume group `rhel` holding XFS logical volumes.
- DHCP and IPv6 auto-configuration on `ens18`.

### Storage layout

Separate `/tmp`, `/var`, `/var/tmp`, `/var/log` and `/var/log/audit` volumes
are defined in every profile. Logical volume sizes (GiB): This is a very minimal setup so adjust sizes to fit your system workload.

| Mount point | `rhel96-generic` | `rhel96-jenkins` | `rhel96-os-composer` |
| --- | --- | --- | --- |
| `/` | 4 | 5 | 5 |
| `/tmp` | 1.5 | 4 | 4 |
| `/var` | 4 | 4 | 4 |
| `/var/tmp` | 2 | 2 | 3 |
| `/var/log` | 2 | 4 | 3 |
| `/var/log/audit` | 1 | 4 | 4 |
| `/home` | 2 | 4 | 2 |
| `/opt` | 2 | 5 | 5 |
| `/var/lib/jenkins` | – | 14 | – |
| `/var/cache/osbuild-worker` | – | – | 20 |
| `/var/lib/osbuild-composer` | – | – | 10 |
| swap | 2 | 2 | 2 |
| **Total LVM** | **20.5** (24GB) | **48** (60GB) | **62** (64GB) |

### Generic profile hardening

`rhel96-generic` targets the **CIS Red Hat Enterprise Linux 9 Benchmark,
Level 2 – Server**:

- OpenSCAP remediation is applied at build time with the profile
  `xccdf_org.ssgproject.content_profile_cis` from the `scap-security-guide`
  data stream. To use a different data stream, set
  `[customizations.openscap] datastream` to its absolute path in the image.
- To find the openscap profile ID for image hardening - install the packages
  `dnf install scap-security-guide openscap-scanner openscap`
- Finf the profile that fits your system compliance
  `oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml`
- Pre generate profile ID blueprint template and modify to fit your requirements using
  `oscap xccdf generate fix --profile=cis --fix-type=blueprint /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml`

The Jenkins and OS Composer profiles do not include the OpenSCAP profile.

## System requirements
* System type: A dedicated RHEL host machine
* Processor: 2 cores
* Memory: 4 GB RAM
* Disk size: 64 GB

## Image build workflow

### 1. Prepare a RHEL Composer host

Configure the RHEL host or VM with at least
2 CPU cores, 4 GiB of memory, and a 64 GiB disk. Provide access to the RHEL
9.6 BaseOS and AppStream repositories, either through subscription-managed
repositories or an approved internal mirror.

If you use an internal mirror, configure its HTTPS URL and trusted repository
signing key in `/etc/yum.repos.d/`. Keep package signature verification
enabled. For example, configure each repository with settings like:

```ini
[repo id]
name=repo name
baseurl=https://<repo-host>/<repository-path>/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/<trusted-repository-key>
```

Replace the placeholders with your organization's repository URL and
provisioned signing key. Refresh DNF's metadata after configuring the sources:

```bash
sudo dnf clean all
sudo dnf makecache
```

### 2. Install and start image composition services

Install osbuild-composer, then enable the Composer socket. 
Cockpit's Composer interface is optional; install and enable it only if the web interface is needed.

```bash
sudo dnf install -y osbuild-composer composer-cli
sudo systemctl enable --now osbuild-composer.socket
composer-cli status show
```

For the optional Cockpit interface:

```bash
sudo dnf install -y cockpit-composer
sudo systemctl enable --now cockpit.socket
```

For command completion in the current shell, load the Composer completion
script:

```bash
source /etc/bash_completion.d/composer-cli
```

To use Composer without logging in as root, add the intended account to the
`weldr` group, then start a new login session for the group membership to
take effect:

```bash
sudo usermod -aG weldr "$USER"
```

### 3. Configure package sources for the image

Composer needs package repositories from which to resolve and install the
blueprint's packages. 
If your build uses an internal mirror instead of the default RHEL sources, configure the supported repository override under
`/etc/osbuild-composer/repositories/` which take precedence over those in the `/usr/share/osbuild-composer/repositories/`.
Below is an example

```ini
mkdir -p /etc/osbuild-composer/repositories
cd /etc/osbuild-composer/repositories

cat > rhel-9.6.json <<'EOF'
{
  "x86_64": [
    {
      "name": "reponame",
      "baseurl": "https://<repo-host>/<repository-path>/",
      "gpgkey"="file:///etc/pki/rpm-gpg/<trusted-repository-key>",
      "rhsm": false,
      "check_gpg": true
    }
  ]
}
EOF
```

Validate the JSON before restarting Composer, then confirm the configured
sources are available:

```bash
json_verify < /etc/osbuild-composer/repositories/rhel-9.6.json
sudo systemctl restart osbuild-composer.service
composer-cli sources list
composer-cli sources info <source-id>
```

### 4. Push and inspect a blueprint

Choose a profile that matches the intended workload. Push the `.toml`
blueprint to the Composer host, then inspect it and resolve package
dependencies before starting a build:

```bash
composer-cli blueprints push profiles/proxmox/generic/rhel96-generic.toml
composer-cli blueprints list
composer-cli blueprints show rhel96-generic
composer-cli blueprints depsolve rhel96-generic
```

Replace the path and blueprint name to build the Jenkins or OS Composer
profile. `composer-cli blueprints save <blueprint-name>` can save a copy of a
blueprint from the Composer service.

### 5. Compose and monitor the image

List the image types supported by the configured Composer environment, then
choose the type that matches your deployment target:

```bash
composer-cli compose types
composer-cli compose start rhel96-generic image-installer
composer-cli compose status
```

The example starts an `image-installer` compose. Check the available types
and resulting artifact for your workflow rather than
assuming every environment produces the same format. A completed compose is
reported with status `FINISHED`; use its compose UUID to inspect the result:

```bash
composer-cli compose info <compose-uuid>
composer-cli compose logs <compose-uuid>
composer-cli compose metadata <compose-uuid>
composer-cli compose image <compose-uuid>
```

The image command downloads the built artifact. The metadata and compose logs can be useful for troubleshoot later.

### 6. Test and deploy

Boot the downloaded artifact in a disposable VM before promoting it. Check
that installation completes, the expected filesystems are present, and the
VM receives network connectivity. For `rhel96-generic`, also run an OpenSCAP
scan against the CIS profile to confirm compliance:

```bash
sudo oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis \
  --report /root/rhel9-cis-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

For further remediation plans, then use:
```bash
sudo oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis \
  --remediate /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml \
  --report /root/rhel9-cis-report.html \
  --results /root/rhel9-cis-results.xml  \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

## Important installation and credential cautions

- Kickstart uses `clearpart --all` and `ignoredisk --only-use=sda`. This
  erases `sda` during installation and ignores any other disk; boot the VM
  with only the intended disk attached, presented as `sda`.
- Network configuration expects the interface name `ens18` and DHCP. Adapt
  the profile if the target VM uses a different interface or addressing
  policy.
- All profiles currently use the same encrypted password hash for both
  `root` and the `support` user, and `support` has `sudo` rights through
  `wheel`. Use unique credentials per environment and secure secret handling.
