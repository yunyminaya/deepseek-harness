# @deepseek-ai/dsh-sandbox-local

[English](README.md) | Español

Implementación local del seam [`dsh-sandbox`](../sandbox/). Selecciona y cachea un runner de plataforma: en Linux prefiere un `bwrap` funcional y después Landlock; en macOS usa Seatbelt; en Windows usa el runner de token restringido por ACL. Varios candidatos se prueban en orden, mientras que un candidato único se selecciona directamente.

La raíz del paquete exporta el plugin `LocalSandboxProvider` por defecto y con nombre, y `Config`; los constructores de profiles de plataforma permanecen internos.

Las plataformas no soportadas y los runners inutilizables fallan en fail-closed con `SANDBOX_UNAVAILABLE`; la ejecución nunca cae silenciosamente sin confinar. Cada wrap lleva reglas estructuradas de fallo de runner para que los consumers puedan distinguir un sandbox roto de un fallo de comando.

La política es por llamada; el provider solo guarda el mecanismo y el veredicto cacheado del runner. Cada wrap informa de la completitud del enforcement más las firmas de denegación específicas del backend y las reglas de fallo de runner. Landlock exige el exit 125 y una línea fatal `landlock-run:` tras excluir únicamente el aviso exacto de enforcement parcial; un aviso con exit de hijo 1, 2 o 125 sigue siendo un resultado del hijo. Bubblewrap y Seatbelt siguen siendo solo de firma porque ninguno de los dos contratos públicos reserva un estado de fallo de launcher. Los consumers hacen spawn directamente del argv devuelto, así que un runner ausente o no ejecutable es un fallo de spawn fuera de banda, mientras que un hijo lanzado con éxito y exit 126 o 127 sigue siendo ordinario. `runnerCommand` se salta las pruebas y exige una o más entradas `runnerFailureSignatures` no vacías, de una sola línea e insensibles a mayúsculas para el dialecto fatal propio del runner personalizado. Como su mecanismo es desconocido, lleva ambos dialectos de denegación de Linux. `probeTimeoutMs` acota las pruebas funcionales. La [Agent Note de sandbox](../../../.agents/notes/implemented/feature/2026-07-06-sandbox.es.md) es dueña de la selección y la semántica de fallo.

El profile de bwrap combina una raíz de host de solo lectura, un `/dev` fresco y un `/proc` de PID privado. Los comandos gestionan descendientes pero no pueden ver los procesos del host; ocultar las entradas `/proc/<pid>` del host impide que los magic links como `root` y `fd` se salten sus mounts. `workspace-write` añade un `/tmp` efímero y un bind de workspace escribible. La [Agent Note de PID privado](../../../.agents/notes/implemented/bug-fix/2026-08-06-bwrap-private-pid-namespace.es.md) registra el límite.

El profile de Seatbelt es allow-default con `(deny file-write*)` más allow-lists de escritura, así que se gobiernan exactamente los efectos de archivo que promete el modo: `read-only` concede solo el literal `/dev/null`; `workspace-write` añade la raíz del workspace, `/tmp` y el directorio temp de darwin por usuario (`os.tmpdir()` — el área temp real de la plataforma para las herramientas de la familia mkstemp), con cada raíz canonizada porque Seatbelt compara rutas resueltas (`/tmp` ES `/private/tmp`). Apple marca la CLI `sandbox-exec` como obsoleta pero la incluye en todos los macOS; la prueba funcional es lo que falla en fail-closed si eso cambia alguna vez.

El rung de Windows mantiene un SID de escritura determinista y un ACE standing por workspace, pero da a cada par sesión/workspace vivo un directorio temp privado aleatorio con un SID distinto y un ACE revocable. Las sesiones que comparten un workspace comparten, por tanto, la autoridad de escritura prevista sin heredar la autoridad temp unas de otras. Un provider nuevo siempre elige una ruta temp y un SID nuevos, así que los residuos de un crash no pueden bloquear ni autorizar una sesión reanudada; las llamadas sin agent reciben del runner el mismo aislamiento por invocación. Un workspace igual a la raíz temp de la plataforma o que la contenga falla antes de cualquier mutación de ACL porque su ACE de workspace heredable alcanzaría de otro modo a cada hijo temp privado.

[`@deepseek-ai/node-addon-landlock-run`](https://www.npmjs.com/package/@deepseek-ai/node-addon-landlock-run) aporta el launcher de plataforma, la prueba funcional y el vocabulario de argumentos de CLI. Este provider solo es dueño del mapeo de modo a concesión y de la selección de runner. Mantener la resolución de rutas y el análisis de la prueba junto al binario versionado evita la deriva del contrato.

```yaml
- id: sandbox
  name: '@deepseek-ai/dsh-sandbox-local'
```

Consumers: [`@deepseek-ai/dsh-bash-sandbox`](../../shell/bash-sandbox/); consulta [el ejemplo acp-agent](../../../examples/acp-agent/) para la composición por defecto ejecutable.

## Model Experience

Indirectamente, a través de [`dsh-bash-sandbox`](../../shell/bash-sandbox/README.es.md) y [`dsh-tool-bash`](../../shell/tool-bash/README.es.md), que renderizan los hechos de enforcement y denegación de este provider, mientras que el seam [`dsh-sandbox`](../sandbox/README.es.md) es dueño del texto `SANDBOX_UNAVAILABLE`, y la selección de runner y los profiles permanecen fuera del contexto.

#### Efecto de KV Cache

Sin invalidación directa; el consumer nombrado es dueño de cualquier cambio en el prefijo de petición.

## Limitaciones conocidas y trabajo aplazado

- **El enforcement de ACL de Windows es parcial** — el token restringido debe conservar Everyone para la inicialización del proceso, así que los objetos externos que conceden acceso de escritura a Everyone siguen siendo escribibles; los hard links NTFS también aliasan un único objeto de archivo entre rutas del workspace y externas. El provider informa de `enforcement: 'partial'` en lugar de exagerar ese límite como completo.
- **Landlock puede ser parcial** — las ABIs de kernel soportadas más antiguas confinan solo las clases de acceso que exponen, informado como `enforcement: 'partial'` en lugar de exagerarlo como completo.
- **Seatbelt depende de `sandbox-exec`, obsoleto** — macOS todavía lo incluye, pero este provider no puede reemplazar ni probar ese motor de política privado si Apple lo elimina.
- **La selección de runner se cachea durante toda la vida del provider** — instalar, eliminar o reparar un runner exige recargar el plugin antes de que cambie la selección.
- **`runnerCommand` es una afirmación del operador** — un runner personalizado configurado se salta las pruebas funcionales y se asume que implementa honestamente el profile compatible con bwrap; si es en sí mismo un script de Bash, el arranque de su intérprete se ejecuta antes de que ese script aplique el confinamiento.
