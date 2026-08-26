# @deepseek-ai/dsh-user-approval

[English](README.md) | Español

Seam de aprobación de un solo uso neutral al canal. `ctx.approval.request(req)` devuelve `allowed-once`, `rejected`, `cancelled` o `unavailable`; los answerers ausentes o que fallan hacen fallar en cerrado, y una concesión se aplica solo a la acción solicitada. Las firmas exactas de los eventos viven en la región generada de [approval.md](../../../docs/subsystems/approval.es.md#cordis-surface).

Cada solicitud debe pertenecer a un turno de agent abierto. El servicio anexa un registro de auditoría emparejado `approval/asked` y `approval/decided`, mientras que el modelo ve solo el resultado de herramienta registrado resultante. Una solicitud abortada se resuelve como `cancelled`; una anexión de auditoría que falla antes del commit rechaza en lugar de devolver una decisión no registrada.

Los answerers son listeners de waterfall de `approval/request`. Devuelve un resultado para responder por un agent propio o llama a `next()` para delegar. Los listeners con ámbito de agent reciben solo las solicitudes de ese agent; compón un answerer terminal por despliegue porque el orden de los listeners hermanos no es un mecanismo de prioridad de política. El puente de automatización ACP aporta decisiones de máquina de un solo uso para las sesiones que posee.

`ApprovalPolicy` es `'ask'` o `'never'`. El valor efectivo es el último evento `approval/policy`, con recurso a la config; `setApprovalPolicy()` es la vía de escritura. `'never'` rechaza antes del despacho interactivo. Ambas políticas aportan su significado actual completo a la instantánea de contexto de runtime segura para la caché.

El pipeline de herramientas enruta las decisiones `ask` a través de este seam y falla en cerrado cuando está ausente; la herramienta bash en sandbox también lo usa para reintentos escalados. El puente de automatización ACP responde las llamadas de sus propios agents a través de la política de máquina del cliente. Los eventos de auditoría siguen siendo solo de log, así que el modelo ve solo el resultado del consumer que pregunta. Ver la [Agent Note del approval seam](../../../.agents/notes/implemented/feature/2026-07-06-approval-seam.es.md) y la [Agent Note de sandbox](../../../.agents/notes/implemented/feature/2026-07-06-sandbox.es.md).

## Experiencia del modelo

### Contexto actual de la política de aprobación

#### Qué ve el modelo

La primera solicitud y cada cambio efectivo de política anexan una instantánea completa de contexto de runtime después del historial retenido. Con `ask`, la aportación de aprobación declara que se puede consultar a los answerers configurados y que su ausencia falla en cerrado. Con `never`, declara la consecuencia determinista de rechazo y no escalada. Las solicitudes sin cambios conservan la instantánea anterior sin añadir otro mensaje.

##### Aportación de la política ask

```markdown
Approval policy: ask. Operations that require approval may ask through the configured answerers; without an available answerer, the request fails closed.
```

##### Aportación de la política never

```markdown
Approval prompts are disabled in this session: actions that require approval are rejected automatically — do not request sandbox escalation (do not set `sandbox_permissions`).
```

#### Efecto de tokens

Un mensaje de contexto conciso en la primera solicitud y en cada cambio efectivo; las solicitudes sin cambios no añaden tokens de política duplicados.

#### Efecto en la caché KV

Solo añadidura después del historial retenido. Una conmutación `ask`/`never` conserva el prefijo estable de sistema y de conversación en lugar de reescribir el primer mensaje de wire.

### Resultado de herramienta

#### Qué ve el modelo

`approval/asked` y `approval/decided` son solo de log. El modelo ve solo el resultado de herramienta final —permitido, rechazado, cancelado o no disponible— del consumer que pregunta; la UI de permisos humana no es contexto.

#### Efecto de tokens

Cero tokens de auditoría duplicados. Un rechazo puede reemplazar un resultado de herramienta normal por un error retenido pequeño, mientras que una autorización deja el resultado ordinario del consumer.

#### Efecto en la caché KV

Solo añadidura; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Las solicitudes solo valen dentro de un turno abierto** — un llamador inactivo o entre turnos lanza una excepción antes de auditar; un flujo de trabajo de aprobación fuera de turno y duradero queda diferido.
- **Solo existen concesiones de un solo uso** — el vocabulario de resultados tiene `allowed-once` pero no `allow-always`, regla recordada, revocación ni almacén de concesiones; la política de sesión es solo `ask` / `never`.
- **La solicitud no lleva argumentos de herramienta** — un answerer ve el nombre de la herramienta, la razón y un id de llamada opcional; el canal de máquina ACP exige un id de llamada y delega las solicitudes sin él.
- **Sin answerer integrado** — los despliegues sin cabeza o compuestos de forma incompleta se resuelven como `unavailable` y fallan en cerrado; el servicio en sí nunca pregunta a una persona.
