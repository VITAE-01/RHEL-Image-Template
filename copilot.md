# Copilot guidance

## Repository purpose and layout

This repository contains declarative RHEL 9.6 image blueprints for osbuild-composer. Profile definitions live under `profiles/proxmox/`, grouped by workload:

- `generic/` — baseline server
- `jenkins/` — Jenkins server
- `os-composer/` — osbuild-composer server

Each profile has two representations with the same blueprint name: an extensionless blueprint and a `.toml` file. Both describe the profile metadata and installer Kickstart customization. Keep them synchronized when changing a profile. `README.md` documents the current profiles, system requirements, and build workflow.

## Making changes

- Keep changes declarative and scoped to the requested profile or documentation.
- Read the existing profile and its paired representation before editing; follow their naming and formatting conventions.
- When changing profile behavior, update the extensionless blueprint file and use the command `composer-cli blueprints save <blueprint_filename>` to generate the `.toml`. Check both representations and ensure that their metadata and Kickstart contents agree.
- Update `README.md` when a change affects documented profile behavior, requirements, or workflow.
- Do not add credentials, password hashes, private keys, or other secrets. The existing profiles have security-sensitive installation settings; preserve the cautions in the README and avoid copying sensitive values into documentation.
- Treat Kickstart disk-clearing directives as destructive: profiles currently clear the selected disk and target `sda`. Do not change disk or networking assumptions without an explicit requirement.

## Build and verification workflow

The documented workflow uses a RHEL 9.6 Composer host with access to approved BaseOS and AppStream repositories. For internal mirrors, follow the README's repository setup and retain package-signature verification.

For a profile change, use the matching `.toml` file with Composer as documented in `README.md`:

1. Push the blueprint and inspect it.
2. Resolve dependencies before starting a compose.
3. Check supported compose types and monitor the compose through completion.
4. Boot the resulting artifact in a disposable VM; check installation, filesystems, and network connectivity before deployment.

For local checks, validate edited TOML syntax and compare each changed pair for equivalent metadata and Kickstart content. Use the README's Composer commands for environment-level validation; Composer builds require the documented RHEL host and repository access. Do not claim a profile was built or boot-tested unless that validation was actually performed.
