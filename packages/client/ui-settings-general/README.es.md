# @deepseek-ai/dsh-client-ui-settings-general

[English](README.md) | [中文](README.zh.md) | Español

Carcasa de ajustes, texto sin dueño y espacio de nombres duradero de onboarding del producto. Ocupa `sidebar.settings` con el chrome del disparador y el panel de ajustes modal, proyecta el ledger `settings.section` en la navegación y el ledger `settings.onboarding` en un paso montado a la vez, y registra en las páginas de Ajustes todo lo que no pertenece a ninguna funcionalidad concreta: el contenido de chrome del disparador/cabecera/cierre, la acción del archivo de configuración local, la sección General y su slot `settings.general.item`, y los diccionarios `settings`. Los tipos de slot en los que renderiza pertenecen a ui-settings, la base del dominio de ajustes; solo los tipos de contrato propios de la carcasa viven aquí, porque referencian el tipo de slot de ui-sidebar y la capa base no debe depender de ningún paquete `ui-*`. Las filas propiedad de funcionalidades (Permission, Language, Appearance), las secciones (Models) y los pasos de onboarding condicionales permanecen en sus paquetes de funcionalidad.

La carcasa no envía ningún texto de onboarding propio: todo el texto llega de los registrantes. Las etiquetas de navegación pueden ser thunks que siguen el locale, así que la proyección de navegación las resuelve a través de `resolveSlotLabel` y vuelve a renderizar con el incremento del ledger de secciones o la revisión del locale (una lectura opcional `ctx.get('locale')`; sin dependencia dura de locale). El ledger de onboarding proyecta en orden ascendente y monta exactamente un paso a la vez. Los pasos visibles son dueños de su chrome de diálogo y del ciclo de vida `inert` de la raíz de la app; un paso montado que aún resuelve hechos privados renderiza null, así que nada pinta ni bloquea mientras decide. El registrante activo recibe su id, `complete()` y una devolución de llamada `openSection(id)`; completar o saltar transfiere la propiedad a la siguiente entrada. Los registrantes son dueños de la finalización duradera, la disposición de la capacidad, el texto, las mutaciones y su envoltorio visible, de modo que los flujos registrados de forma independiente no pueden apilarse y la carcasa no se convierte en una segunda fuente de hechos de configuración.

Un navegador de bucle local carga la capacidad `hasDocument` del provider a través de `settings.describe` y renderiza **Open configuration file** solo cuando el Host confirma que puede prepararse un documento local propiedad del provider. La acción envía la solicitud `settings.openDocument` sin ruta y solo de bucle local; el Host resuelve de nuevo la ruta del provider, materializa un documento ausente y se lo entrega a un editor de texto nativo (`open -t` en macOS, evitando la asociación de archivos del navegador; la asociación de archivos de escritorio en Linux y Windows; la asociación de Windows tras la traducción `wslpath -w` en WSL). Los fallos de apertura mantienen la acción disponible y renderizan un error localizado. Reabrir el diálogo o reconectar refresca la disponibilidad tras un fallo de lectura transitorio o un cambio de topología del Host. Los navegadores remotos nunca registran la acción y nunca emiten la lectura privilegiada de ajustes.

La mitad de Host registra `ui-onboarding` en el seam de ajustes de usuario. El paso de bienvenida aportado por `ui-settings-models` lee y escribe su `welcomeNoticeVersion` a través del límite público de ajustes existente; la carcasa en sí permanece sin política.

## Experiencia de modelo

Ninguna, ya que el plugin renderiza la UI de ajustes del navegador; nada aquí llega a una solicitud de modelo.

#### Efecto en KV Cache

Ninguno; este paquete no ensambla ni envía ninguna solicitud de provider.

## Limitaciones conocidas y trabajo diferido

- La sección General no tiene filas integradas; cada fila aparece solo cuando su plugin de funcionalidad dueño está montado.
