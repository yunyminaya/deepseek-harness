# Agent Note: Informes de API extractor

Status: proposed

[English](2026-06-11-api-extractor-reports.md) | Español

> Las partes de tipado de bloques de documentación y de taxonomía de eventos ya se enviaron ([aplicación de doc-sync](../../archived/process/2026-06-11-doc-sync-enforcement.md)); esta parte restante de informe de API queda aplazada como propuesta independiente.

## Problema

Los cambios en la API pública son invisibles: nada convierte «este commit cambió la API pública» en un hecho explícito y revisable. Un revisor que lee un diff puede pasar por alto que un tipo exportado ganó un campo o que una firma de método cambió.

## Propuesta

api-extractor (o `tsc --emitDeclarationOnly` + un volcado normalizado de la API pública) que genere un `etc/<pkg>.api.md` versionado por paquete; CI falla si la regeneración difiere. Cada cambio de API pública se convierte en una línea de diff que un revisor (o un agent de revisión) debe ver.

## Alternativas consideradas

**`tsc --emitDeclarationOnly` más un volcado normalizado de la API pública** — el mecanismo más ligero si api-extractor resulta demasiado pesado; cualquiera de los dos satisface la forma de informe versionado y diffable que la propuesta necesita.

## Criterios de aceptación

- Cada paquete tiene un `etc/<pkg>.api.md` versionado; CI falla cuando la regeneración difiere del informe commiteado.
- Un cambio de API pública (una exportación nueva, un campo ampliado, una firma desplazada) es visible como línea de diff del informe en la revisión.

## Riesgos

La dependencia es pesada y quisquillosa — la razón por la que esto se aplazó — y el formato del informe cambia con cada actualización del compilador, añadiendo una carga de mantenimiento que compra poco mientras los paquetes sigan sin publicarse.

## Por qué se aplazó

Se aplazó cuando aterrizó doc-sync: poco valor para un monorepo interno donde los revisores ya ven el diff del código fuente, y una dependencia pesada y quisquillosa. Revisítalo si los paquetes se publican algún día externamente — en ese punto una API pública estable y diffable se paga sola.
