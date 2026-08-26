# web-cordis

[English](README.md) | Español

Demostración autorreferencial de [`@deepseek-ai/dsh-tool-cordis`](../../packages/extensions/tool-cordis/README.es.md). El agent puede inspeccionar su proceso Cordis actual y montar o desmontar en memoria plugins escritos por el modelo. Los plugins temporales desaparecen cuando se desmontan o cuando el proceso sale, y pueden afectar a otras sesiones del mismo proceso.

## Ejecútalo

Arranca la interfaz de navegador:

```sh
pnpm run demo:cordis
```

Arranca en su lugar el servidor de automatización ACP:

```sh
pnpm run demo:cordis acp
```

Ambos comandos requieren `DEEPSEEK_API_KEY`. La [referencia de la herramienta Cordis](../../packages/extensions/tool-cordis/README.es.md) define los argumentos de la herramienta, su ciclo de vida, su limpieza y los contratos de seguridad.
