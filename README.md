# settings
https://geba-trans.github.io/settings/

<!-- GEBA hand-off runbook -->

## GEBA operational hand-off

### Wat zijn de veelvoorkomende fouten en storingen, en hoe worden die normaal opgelost?

This repository contains configuration/bootstrap settings. Failures usually come from wrong environment values, missing secret, permissions or applying settings in the wrong order. Compare a change against the target environment before applying it.

### Hoe en waar is de logging terug te vinden?

CI/CD output, deployment logs and the logs of the service consuming the settings; this repository has no standalone runtime log described.

### Hoe herstart of herstel je de applicatie?

There is no standalone application to start/stop. Apply settings through the consuming deployment, restart only affected services and validate configuration/health endpoints.

### Hoe worden updates doorgevoerd, en wat is er verder aan onderhoud nodig?

Review changes, protect secrets, test in staging and roll out atomically where possible. Maintain configuration ownership, secret rotation, audit history and rollback copies.
