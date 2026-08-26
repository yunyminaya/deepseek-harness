# e2e de navegador de apps/web

[English](README.md) | Español

Estas pruebas arrancan la composición web real en proceso y la manejan con un Chromium real sobre HTTP real. La mecánica del carril — modos, fixtures, goldens y las divergencias deliberadas de composición respecto a `dsh web` — está documentada en [`scaffold.ts`](scaffold.ts) y en la [Agent Note de browser e2e](../../../.agents/notes/implemented/testing/2026-07-24-web-gui-browser-e2e-lane.es.md).

## Estas son pruebas de cara al Host

Verifican tipos en el `tsconfig.host.json` raíz, no en el agregado Client, porque leen servicios del Host directamente: `ctx.apiProxy`, el `SessionStore` del Host, `ctx.sessionProjectionCache`. Manejar un navegador en runtime no convierte un archivo en parte del programa Client — las dos caras fusionan el `Context` de cordis bajo las mismas claves con servicios distintos, así que un programa no puede ver ambas. Mover estos archivos al agregado Client hace que todo acceso a servicios del Host falle al compilar.

## No importes `@deepseek-ai/dsh-client-*` aquí

Importar un paquete Client — un valor o un tipo — arrastra todo su proyecto TypeScript, y todos los proyectos a los que referencia, al **grafo de compilación del Host**. Eso ya mordió a este carril una vez: cuatro paquetes de consumidor Client referencian la cara Client de `api/remotes`, que no puede compilar hasta que el tsdown del Host haya generado `@deepseek-ai/dsh-goal/remote`, así que la fase de compilación del Host terminó esperando un artefacto que produce ella misma.

Cuando un escenario necesita una constante o función pura propiedad del Client, refléjala aquí en su lugar, junto al import comentado que nombra el módulo fuente. Una deriva aparece entonces como un selector fallido o un valor reflejado obsoleto — un fallo ruidoso, nunca un pase silencioso. `scaffold.ts` sigue esta regla para el espacio de nombres del aviso de bienvenida, el campo de acuse, la versión y el texto chino verificado.

Dos tipos de import Client se mantienen. `assembled-boot.ts` maneja el propio shell, así que importa `AppWebEntry` de `@deepseek-ai/dsh-client-web` y el tipo del manifest de arranque de `@deepseek-ai/dsh-client-modules/client`: arrancar el shell real es para lo que existe ese harness, y ambos paquetes ya están en el grafo del Host. Aparte, los escenarios de chat importan `conversationContextKey` de `@deepseek-ai/dsh-client-runtime/client` porque `client/runtime` es alcanzable a través de los paquetes `directory-picker` sin dividir y no arrastra nada más. Esa alcanzabilidad es incidental, no una garantía — si algún día sale del grafo, refleja el helper como el resto.

Nada hace cumplir esta regla mecánicamente; mantenla presente en las revisiones.
