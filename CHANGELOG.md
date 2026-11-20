# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [v1.0.0] - 2026-11-20

Initial release.

### Added

- Declarative UFW rule management: rules are declared per service in
  `ufw_common`, `ufw_group` and `ufw_host` (merged per service), or in
  `ufw` (complete rule set for a host).
- Support for every `community.general.ufw` rule parameter. List-valued
  fields expand into one rule per combination.
- Reconciliation: missing, changed and undeclared rules are detected, and
  the rule set is rebuilt only when needed.
- Check mode support.
- Validation of rule comments and detection of duplicate rule definitions.
- Molecule tests for Ubuntu 24.04 and 26.04, and GitHub Actions workflows
  for CI and releases.

[v1.0.0]: https://github.com/Maksym-Ototiuk/ansible-role-ufw/releases/tag/v1.0.0
