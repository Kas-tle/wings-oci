# Wings for Oracle Linux 8

## About

Builds of [Wings](https://github.com/pterodactyl/wings) compatible with Oracle Linux 8. This repository is neither endorsed by nor affiliated with the Pterodactyl project.

Please see the [releases](https://github.com/Kas-tle/wings-oci/releases) tab. All builds are done via Github Actions. The [action](https://github.com/Kas-tle/wings-oci/blob/main/.github/workflows/release.yml) is triggered when releases.json is modified with a new Wings release tag.

## Downloading

```bash
mkdir -p /etc/pterodactyl
curl -L -o /usr/local/bin/wings "https://github.com/kas-tle/wings-oci/releases/latest/download/wings_oracle_linux_$([[ "$(uname -m)" == "x86_64" ]] && echo "amd64" || echo "arm64")"
chmod u+x /usr/local/bin/wings
```

## Full Setup Guide

Please refer to [GUIDE.md](https://github.com/Kas-tle/wings-oci/blob/main/GUIDE.md) for a full walkthrough of setting up Wings on an ARM OCI machine.
