# AGENTS.md — GitHub Actions

[English](AGENTS.md) | Español

Ejecuta jobs en runners de Windows (etiquetas `windows-*`) bajo `pwsh` nativo.
El job `windows` de pull request es la excepción deliberada: ejecuta Windows Node bajo Wine en Linux hospedado y bloquea `all checks passed`; `windows-native` se ejecuta automáticamente en `windows-2025` (o en el pool self-hosted `[self-hosted, dsh-win-ci, windows]` bajo `DSH_CI_FAILOVER_WINDOWS=selfhosted`) pero informa de manera independiente.
`ci.yml` es solo para pull requests; el standby `serial-windows` de master, el standby Linux `serial-linux-selfhosted`, el seeder `wine-apt-cache` y los dos benchmarks manuales de runners viven en `ci-master.yml` (push a master + `workflow_dispatch`).
Como `ci-master.yml` no escucha `pull_request`, esos jobs solo-de-master nunca aparecen en los paneles de checks de los PRs (un job que un workflow define para un evento dado se lista y muestra `skipped` cuando su `if` es falso); mantenerlos en un workflow separado es lo que evita que los círculos de checks de PR muestren segmentos grises.
El standby `serial-windows` de master valida continuamente el objetivo self-hosted de failover — ver el [runbook de failover](../.agents/notes/implemented/process/2026-07-26-ci-failover-runbook.es.md).
