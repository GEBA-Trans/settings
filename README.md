# settings
https://geba-trans.github.io/settings/

<!-- GEBA hand-off runbook -->

## GEBA operational hand-off

### Wat zijn de veelvoorkomende fouten en storingen, en hoe worden die normaal opgelost?

This repository contains configuration/bootstrap settings. Failures usually come from wrong environment values, missing secret, permissions or applying settings in the wrong order. Compare a change against the target environment before applying it.


<!-- GEBA git history bugfix review -->


#### Git history: bugfixes and incident-related changes
The following entries are taken from the available Git history. Where a commit references an issue or pull request, its title was looked up and included as the one-line error/change description.
- `d5025cc` (2025-03-25) — fix: update branding color variables in branding.css for consistency — **Description:** Commit description: fix: update branding color variables in branding.css for consistency
- `9d1ff67` (2025-03-24) — fix: define warning color variable in branding.css — **Description:** Commit description: fix: define warning color variable in branding.css

### Hoe en waar is de logging terug te vinden?

CI/CD output, deployment logs and the logs of the service consuming the settings; this repository has no standalone runtime log described.

### Hoe herstart of herstel je de applicatie?

There is no standalone application to start/stop. Apply settings through the consuming deployment, restart only affected services and validate configuration/health endpoints.

### Hoe worden updates doorgevoerd, en wat is er verder aan onderhoud nodig?

Review changes, protect secrets, test in staging and roll out atomically where possible. Maintain configuration ownership, secret rotation, audit history and rollback copies.
