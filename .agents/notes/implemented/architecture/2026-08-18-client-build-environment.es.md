# Agent Note: Variables de entorno públicas en tiempo de compilación para el código de negocio del cliente

Status: implemented

[English](2026-08-18-client-build-environment.md) | Español

## Problema

Los paquetes de negocio de navegador necesitan que las compilaciones de despliegue seleccionen comportamiento estático, pero el cliente web tiene dos rutas de artefacto que no se contienen mutuamente: Vite compila el shell estático, mientras que el preset compartido de tsdown compila los plugins cargados dinámicamente. Reemplazar una expresión de entorno en una sola ruta daría a la misma expresión de negocio resultados distintos según el tipo de paquete.

Los navegadores no tienen `process` de Node, e incrustar el objeto de entorno completo del proceso de compilación expondría valores ajenos al frontend. La configuración de runtime tampoco representa con precisión una variante de compilación porque esta elección debe permanecer fija después de publicar un artefacto.

## Decisión

`DSH_CLIENT_*` es el namespace en tiempo de compilación para los valores que pueden exponerse al código de negocio de navegador. El código de negocio puede usar una lectura de propiedad estática como `process.env.DSH_CLIENT_NAME` para seleccionar comportamiento. Los valores provienen solo del entorno del proceso de compilación, no de archivos `.env*` de Vite. Los valores establecidos se incrustan como cadenas, y los no establecidos evalúan a `undefined`.

La configuración de Vite y el preset compartido de tsdown para bundles de cliente dinámicos usan un único generador de defines. El generador crea sustituciones exactas solo para `DSH_CLIENT_*` y reduce todas las lecturas restantes de `process.env` a un objeto vacío. El navegador no recibe ningún `process` global, ni búsqueda por clave dinámica, ni capacidad de enumeración del entorno.

El propio prefijo `DSH_CLIENT_*` declara que un valor es público. Las credenciales, rutas y demás valores solo de Host o CI no deben usarlo.

El envoltorio de compilación raíz suministra un único entorno público exacto a ambos bundlers. Deriva `DSH_CLIENT_COMMIT_HASH` como el prefijo de siete caracteres del HEAD Git de la fuente para toda compilación completa; un valor explícito da soporte a entornos de compilación sin metadatos de repositorio. `pnpm run build` hereda por lo demás los valores `DSH_CLIENT_*` del llamador, mientras que `pnpm run build:official` selecciona el perfil de artefacto oficial del repositorio sin sintaxis de entorno específica del shell y fija `DSH_CLIENT_BUILD_PROFILE=official` para los registros de negocio específicos de despliegue. Una compilación completa exitosa escribe el entorno público exacto y un digest que cubre la salida de Vite y cada bundle de cliente dinámico. Los comandos de compilación parcial no reemplazan ese registro.

## Alternativas consideradas

**Reemplazar valores solo en Vite.** El `lib/client.js` de un plugin dinámico se carga como script independiente y nunca entra en el grafo de módulos de Vite, así que la expresión permanecería en un navegador que no tiene `process`.

**Exponer cada valor `DSH_*`.** Las variables de Host, prueba y CI ya usan ese prefijo y pueden contener credenciales o rutas locales. El prefijo más estrecho `DSH_CLIENT_*` hace auditable la intención de exposición.

**Proporcionar un objeto `process.env` completo en el navegador.** Esto permitiría enumerar el entorno de compilación y convertiría un shim de compatibilidad de Node en una API de runtime. Las sustituciones estáticas exactas bastan para las elecciones de compilación.

**Estandarizar en `import.meta.env`.** Los plugins dinámicos se emiten como fábricas CommonJS independientes y no pueden conservar `import.meta`. El código de negocio seguiría necesitando dos interfaces según la ruta del artefacto.

## Consecuencias

El shell estático de Vite y los bundles dinámicos del preset compartido de tsdown reciben la misma cadena para una variable de compilación `DSH_CLIENT_*` dada. Una lectura de propiedad estática no establecida evalúa a `undefined`; los valores no-`DSH_CLIENT_*` no pueden entrar en los artefactos de navegador por este mecanismo, y el código de negocio no puede enumerar el entorno del proceso de compilación. Cada compilación completa lleva su revisión corta de fuente como metadato de visualización público. Los gates de compilación de CI seleccionan el perfil oficial sin exponer sus valores públicos a las pruebas de fuente ni a pasos de workflow no relacionados. El empaquetado npm y las pruebas web compiladas verifican el entorno registrado y el digest actual del artefacto, de modo que una compilación por defecto seguida de una petición de paquete oficial, una recompilación parcial o una salida modificada falla antes del consumo.

Todo valor `DSH_CLIENT_*` referenciado por el código de negocio se convierte en contenido público del artefacto, así que un valor mal nombrado puede divulgar información. Las elecciones de compilación quedan fijas al generarse el artefacto; un ajuste que deba cambiar después del despliegue requiere un mecanismo de configuración de runtime validado, transportado y documentado.
