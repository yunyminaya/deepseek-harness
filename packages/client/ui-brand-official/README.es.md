# @deepseek-ai/dsh-client-ui-brand-official

[English](README.md) | [中文](README.zh.md) | Español

Este paquete ocupa `sidebar.brand.mark`, `sidebar.brand.name` y `conversation.hero.brand.mark` solo cuando `DSH_CLIENT_BUILD_PROFILE` es `official`. Otros builds cargan el plugin pero no registran ocupantes, dejando visibles los fallbacks del shell.

Los tres ocupantes se instalan como un conjunto de registros consciente de declaraciones mediante llamadas `slots.inject()` anidadas. El paquete funciona así tanto si su fila se activa antes como después de los declarantes de la barra lateral y de la conversación, retira todos los ocupantes cuando cualquiera de las dos declaraciones colapsa y no deja mezcla de marca parcial durante HMR (hot module replacement). No conserva estado de runtime. La mitad node es un asiento de Loader vacío, y el título del navegador sigue siendo una preocupación del entorno de build fuera de este paquete.

## Experiencia de modelo

Ninguna, porque el paquete aporta solo presentación de navegador; nada de esto llega a una petición de modelo.

#### Efecto en la caché KV

Ninguno; este paquete ni ensambla ni envía una petición al provider.

## Limitaciones conocidas y trabajo pendiente

- **El paquete suministra un único conjunto de ocupantes** — la presentación alternativa pertenece a otro paquete Cordis que ocupe los mismos slots.
- **El título del navegador es independiente** — `DSH_CLIENT_TITLE` selecciona el texto del título en tiempo de build en lugar de hacerlo a través de un slot de UI.
