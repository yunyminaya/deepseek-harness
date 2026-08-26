# extensions/ — el agent modifica su propio runtime

[English](README.md) | Español

Herramientas orientadas al modelo sobre el runtime de cordis en vivo dentro del que corre el propio agent (agente): inspeccionar los plugins cargados y la API de servicios, definir y ejecutar paquetes dinámicos escritos por el modelo, y retractarlos de nuevo — además del runtime de Plugin de repositorio restringido. Ambos paquetes de mitad de navegador viven aquí en lugar de bajo `packages/client/` porque son mitades de los paquetes de dos mitades de este subsistema; el agregado de host los excluye para que cada cara conserve su propio programa de compilación. Hogar de diseño: [la Agent Note del toolset](../../.agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.md).

| Paquete | Rol | clave ctx |
|---|---|---|
| [`tool-cordis/`](tool-cordis/README.es.md) | Herramientas de inspección de runtime orientadas al modelo y de paquetes dinámicos | se registra en `ctx.tools` |
| [`cordis-host-runner/`](cordis-host-runner/README.es.md) | Registro de definiciones, el sandbox `node:vm` para las mitades de host y el viaje de ida y vuelta de solicitud de ejecución | aporta `ctx.dynamicCordisRunner` |
| [`cordis-client-runner/`](cordis-client-runner/README.es.md) | Mitad de navegador de un paquete de dos mitades: evalúa la definición convirtiéndola en un plugin de navegador en vivo y responde a la solicitud de ejecución | cara de cliente; aporta el `ctx.dynamicCordisRunner` del navegador |
| [`ui-cordis/`](ui-cordis/README.es.md) | Superficies de navegador: el panel de ancho de frame que opera cada definición, y la tarjeta de definición de solo lectura | cara de cliente; registra slots |
