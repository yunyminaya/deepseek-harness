# Flujos de trabajo del colaborador de Python

[English](development.md) | [中文](development.zh.md) | Español

Sigue el flujo de trabajo según el resultado de colaborador que necesites: construir artefactos del runtime, validar el SDK, ejecutar contra la fuente o construir distribuciones. El comportamiento de los paquetes pertenece a la [referencia del SDK](sdk/README.es.md) y a la [referencia del portador del runtime](sdk-runtime/README.es.md).

## Construir artefactos del runtime

Los ejecutables por plataforma son artefactos de construcción y no se consignan en git. Ejecuta la construcción desde la raíz del repositorio:

```sh
pnpm install
pnpm exec tsx scripts/build-exe-for-python-sdk.ts
```

Usa `--skip-build` cuando los artefactos `lib/` requeridos ya existan, o `--targets=node24-linux-x64,node24-linux-arm64,node24-macos-arm64` para seleccionar plataformas. Los productos caen en `dist-exe/` y el script sincroniza los portadores seleccionados en `python/sdk-runtime/`. Las construcciones de macOS también sincronizan el auxiliar de spawn requerido por `node-pty`.

## Validar el SDK

Mantén el entorno virtual fuera de `python/`, instala el grupo de pruebas y ejecuta la suite de Python:

```sh
export UV_PROJECT_ENVIRONMENT="$PWD/tmp/py-sdk-venv"
uv sync --project python/sdk --group test
uv run --project python/sdk pytest
```

`python/sdk/tests/test_bundled_runtime.py` ejercita los portadores incluidos disponibles y salta un portador cuyo artefacto no se ha construido. Para la política de pruebas de todo el repositorio, consulta [Testing](../docs/testing.es.md).

Esa suite conduce pares de runtime falsos. `scripts/smoke-python-runtime.py` conduce en cambio el runtime empaquetado real, y el trabajo de CI `python-runtime`, obligatorio, ejecuta cada escenario contra un ejecutable recién construido:

```sh
uv run --project python/sdk python scripts/smoke-python-runtime.py \
  --scenario sdk-minimal --exe dist-exe/dsh-jsonrpc-agent-pkg-macos-arm64
```

Dos escenarios comparan la salida esperada consignada bajo `scripts/snapshots/python-sdk-single-exe/`. `minimal/model-visible.json` fija los system prompts ensamblados de la composición mínima consignada, los esquemas de herramientas anunciados y los mensajes visibles para el modelo, así que un plugin que aporta una sección de sistema o un mensaje de usuario no intencional hace fallar el trabajo; descarta la instantánea dinámica de contexto de runtime, que la misma composición emite en macOS y no en Linux ([#2488](https://github.com/deepseek-harness/deepseek-harness/issues/2488)). `advanced/` fija el resultado del SDK y los registros de sesión persistidos. Reejecuta el escenario propietario con `--update-snapshots` y revisa ese diff antes de consignarlo.

Un smoke interactivo necesita `DEEPSEEK_API_KEY` en el entorno o en el `.env` de la raíz del repositorio:

```python
from deepseek_harness import DeepSeekHarness

with DeepSeekHarness() as harness:
    print(harness.run("say hi").final_response)
```

## Ejecutar contra la fuente de Node

Los colaboradores del repositorio pueden elegir cualquiera de los dos portadores de desarrollo:

- Fija `DSH_RUNTIME_MODE=node` para usar el portador de Node construido sobre el Node del sistema `>=22.19`. El script de construcción refresca este portador, pero las distribuciones nunca lo incluyen ni lo seleccionan automáticamente.
- Fija `launch_args_override=("./node_modules/.bin/tsx", "packages/examples/jsonrpc-demo/src/bin.ts")` con la raíz del repositorio como `cwd` para ejecutar la fuente TypeScript sin construir. Suministra `cordis=...` cuando la configuración por defecto no convenga.

Consulta `python/sdk/tests/manual_sdk_agent_smoke.py` para una invocación completa en modo fuente.

## Construir distribuciones

La versión del `package.json` raíz es la autoritativa para ambas distribuciones de Python. El script de preparación inyecta esa versión en ambos wheels y fija el SDK a la misma versión de `deepseek-harness-runtime-bin`.

Construye una vez el wheel puro del SDK y un wheel del runtime en cada plataforma nativa:

```sh
version="$(python - <<'PY'
import runpy

release = runpy.run_path("scripts/build-python-release.py")
print(release["pep440_version"](release["repository_version"]()))
PY
)"
python scripts/build-python-release.py --package sdk --output-dir dist-python
python scripts/build-python-release.py --package runtime --platform macos-arm64 --runtime-exe dist-exe/dsh-jsonrpc-agent-pkg-macos-arm64 --output-dir dist-python
pip install \
  "dist-python/deepseek_harness_sdk-$version-py3-none-any.whl" \
  "dist-python/deepseek_harness_runtime_bin-$version-py3-none-macosx_14_0_arm64.whl"
```

La distribución del runtime es solo wheel. El pipeline de publicación publica tres wheels de plataforma junto con el wheel puro del SDK: Linux x64, Linux arm64 y macOS 14 o superior en arm64. Una etiqueta `python-v<repository-version>` solo se acepta cuando coincide con la versión del repositorio; las versiones de repositorio en prerrelease tales como `0.0.1-rc.1` usan su grafía normalizada PEP 440, como `0.0.1rc1`, dentro de los nombres de archivo y metadatos de los wheels.

## Validar un candidato de publicación

Etiqueta un pull request con `python-release-dry-run`, o ejecuta manualmente el flujo `Release (Python)` de GitHub con `publish=false`, para construir los cuatro wheels, instalar el conjunto de publicación de Linux en Python 3.10 y 3.14, comprobar nombres de archivo y metadatos exactos, imponer el límite de tamaño por archivo de PyPI y retener un artefacto agregado con hashes SHA-256. Ambas vías carecen de credenciales de registro; una ejecución desde un pull request no puede entrar en ninguno de los dos trabajos de publicación.

La publicación pública corre desde el repositorio privado de automatización; los metadatos del paquete apuntan al espejo público de solo lectura y separado del código, que no ejecuta Actions de publicación. El repositorio privado define la variable de repositorio `PYPI_PUBLISHER_REPOSITORY` como su propio `owner/name` y mantiene `PUBLIC_PYPI_RELEASE_ENABLED=false` salvo durante una publicación intencional.

Unos trabajos separados de runtime y SDK permiten que un fallo de subida del SDK se reanude sin reenviar los archivos inmutables del runtime. Aceptan `publish=true` solo cuando el flujo corre desde el repositorio editor configurado en la etiqueta `python-v*` correspondiente y los entornos protegidos `pypi-runtime` y `pypi` aprueban los trabajos del runtime y del SDK, respectivamente. El Trusted Publishing de PyPI sigue suministrando credenciales OIDC de corta duración, pero las atestaciones públicas están deshabilitadas porque revelarían la identidad del editor privado.
