# DeepSeek Harness

[English](README.md) | Español

DeepSeek Harness (`dsh`) es un agent harness (marco de trabajo para agentes) de código abierto desarrollado por [DeepSeek AI](https://deepseek.com).

Usa una arquitectura en la que **todo es un plugin**, y está impulsado por [Cordis](https://github.com/cordiverse/cordis), cuyo diseño se describe en [_A Programming Paradigm for Spatiotemporal Composability_](https://github.com/cordiverse/paper).

## Vista previa para desarrolladores

DeepSeek Harness se encuentra actualmente en _vista previa para desarrolladores_ y está iterando rápidamente. **HABRÁ CAMBIOS QUE ROMPEN LA COMPATIBILIDAD.**

## Ejecutar

### Ejecutar desde `npm`

Instala `Node.js` y, a continuación, ejecuta:

```sh
npx @deepseek-ai/dsh web
```

El comando inicia la Web UI en `http://127.0.0.1:3080` por defecto y la abre en el navegador predeterminado para un arranque local. Un arranque por SSH solo imprime la URL del host porque la dirección reenviada localmente pertenece al cliente SSH o al editor. Pasa `--no-open` para ejecutar el servidor sin abrir un navegador. Consulta la [guía de la Web UI](docs/user/guide/index.es.md).

### Ejecutar desde el código fuente

Para ejecutar desde un clon del repositorio:

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

`pnpm run build` prepara los artefactos del repositorio. `pnpm dsh web` usa esos artefactos compilados sin recompilar.

## Comunidad y soporte

- No dudes en enviar comentarios o informes de bugs a través de [GitHub Discussions](https://github.com/deepseek-ai/deepseek-harness/discussions).
- Añade el tema [`dsh-plugin`](https://github.com/topics/dsh-plugin) al repositorio de tu plugin para que sea fácil de descubrir.
- Únete a la <a href="https://discord.gg/Ycq5dCaS4">comunidad de Discord de DeepSeek Harness</a>.

## Contribuciones

Consulta [CONTRIBUTING.md](CONTRIBUTING.es.md).

## Desarrollo

Empieza con la [guía de desarrollo](docs/development.es.md) y la [documentación de arquitectura](docs/architecture.es.md).

Para agentes, sigue [AGENTS.md](AGENTS.es.md).

## Licencia

[MIT](LICENSE)

Las dependencias de terceros y sus licencias se detallan en [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
