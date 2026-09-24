# Guía: versionamiento manual y empaquetado automático con GitHub Actions

Esta guía explica cómo funciona el flujo de este repositorio: tú editas **manualmente**
la versión en `version.txt`, haces push, y GitHub Actions valida la versión, genera un
ZIP del código fuente y publica un **Release** (estable o prerelease) automáticamente.

---

## 1. Flujo general

```
Editas version.txt   (0.1.0 -> 0.1.1)
        |
        v
git push origin main
        |
        v
GitHub Actions (.github/workflows/version.yml)
        |
        |-- ¿version.txt cambió?         --no--> no se dispara (filtro paths)
        |-- ¿Es SemVer válida?           --no--> falla con error claro
        |-- ¿Es distinta a la anterior?  --no--> falla (evita releases duplicados)
        |-- ¿El tag ya existe?           --si--> falla (evita reutilizar tags)
        |
        |-- ¿Contiene sufijo "-"?  --si--> Prerelease (ej: 0.1.2-beta.1)
        |                          --no--> Release estable (ej: 0.1.1)
        v
Tag vX.Y.Z + Release + source-vX.Y.Z.zip
```

---

## 2. Archivos que participan

| Archivo | Rol |
|---|---|
| `version.txt` | Fuente única de verdad. Contiene solo la versión, ej: `0.1.1` |
| `.github/workflows/version.yml` | Workflow que valida, empaqueta y publica |
| `.gitattributes` | Excluye `guia.md`, `.github/` y sí mismo del ZIP (`export-ignore`) |

---

## 3. Regla de estable vs prerelease (SemVer)

Se usa [Versionamiento Semántico](https://semver.org/lang/es/): `MAJOR.MINOR.PATCH`.
Si la versión lleva **sufijo de prerelease** (empieza con `-`), se publica como
*Pre-release*; si no, como release **estable**.

| Valor en `version.txt` | Resultado |
|---|---|
| `0.1.0` | Release estable `v0.1.0` |
| `0.1.1` | Release estable `v0.1.1` |
| `1.0.0` | Release estable `v1.0.0` |
| `0.1.2-beta.1` | **Prerelease** `v0.1.2-beta.1` |
| `0.1.2-rc.1` | **Prerelease** `v0.1.2-rc.1` |
| `0.1.1+build.5` | Release estable `v0.1.1+build.5` (metadata no cuenta) |
| `0.1` o `v0.1.1` | Error: el workflow falla, formato inválido |

> `version.txt` debe contener **solo** el número (`0.1.1`), sin `v` ni texto adicional.

---

## 4. Explicación del workflow, paso a paso

### 4.1 Disparador

```yaml
on:
  push:
    branches: [main]
    paths: [version.txt]
```

Solo se ejecuta al hacer push a `main` cuando el commit **modifica `version.txt`**.
Un commit que no toque ese archivo no dispara nada.

### 4.2 Permisos

```yaml
permissions:
  contents: write
```

Necesario para que el `GITHUB_TOKEN` pueda crear tags y Releases.
Si al publicar ves un error `403 Resource not accessible`, revisa
**Settings → Actions → General → Workflow permissions** y activa
*Read and write permissions*. No hace falta crear ningún secreto.

### 4.3 Leer la versión

```yaml
VERSION=$(tr -d '[:space:]' < version.txt)
echo "version=$VERSION" >> "$GITHUB_OUTPUT"
```

Elimina saltos de línea/espacios y expone el valor como salida del paso,
para reutilizarlo en los pasos siguientes con `${{ steps.version.outputs.version }}`.

### 4.4 Validar SemVer y el cambio real

- Valida con una expresión regular el formato `MAJOR.MINOR.PATCH` con sufijos opcionales.
- Compara contra el `version.txt` del commit anterior (`github.event.before`) y
  **falla si no cambió**, evitando releases repetidos por accidente.
- Verifica que el tag `vX.Y.Z` no exista, para no reutilizar versiones publicadas.

En el primer push del repositorio no hay commit anterior, por lo que la comparación
se omite y se publica la versión inicial tal cual.

### 4.5 Determinar estable o prerelease

```yaml
if [[ "$VERSION" == *-* ]]; then
  TIPO="prerelease"; PRE="true"
else
  TIPO="estable"; PRE="false"
fi
```

Si la cadena contiene `-`, hay sufijo de prerelease. Ese valor alimenta la entrada
`prerelease` de la acción de release.

### 4.6 Empaquetar el código fuente

```bash
git archive --format=zip --output="source-v0.1.1.zip" HEAD
```

`git archive` crea el ZIP **solo con archivos versionados** (sin `.git/`) del commit
actual. Gracias a `export-ignore` en `.gitattributes`, la guía y el propio `.github/`
no se incluyen en el paquete.

### 4.7 Publicar tag y Release

```yaml
uses: softprops/action-gh-release@v3
with:
  tag_name: v${{ steps.version.outputs.version }}
  prerelease: ${{ steps.tipo.outputs.prerelease }}
  files: source-v${{ steps.version.outputs.version }}.zip
  generate_release_notes: true
```

Crea el tag `vX.Y.Z` (si no existe), publica el Release con el ZIP adjunto y añade
notas generadas automáticamente con los commits desde el release anterior.
Si `prerelease` es `true`, GitHub lo marca como *Pre-release*.

---

## 5. Cómo probarlo

Requisitos: tener el repositorio remoto configurado y permisos de push a `main`.

```bash
# 1) Primer push (publica la versión inicial 0.1.0)
git add .
git commit -m "chore: version inicial 0.1.0"
git push origin main

# 2) Release ESTABLE: 0.1.0 -> 0.1.1
echo "0.1.1" > version.txt
git commit -am "chore: subir version a 0.1.1"
git push origin main

# 3) PRERELEASE: 0.1.1 -> 0.1.2-beta.1
echo "0.1.2-beta.1" > version.txt
git commit -am "chore: version 0.1.2-beta.1"
git push origin main
```

Después de cada push revisa:

1. **Pestaña Actions** → run `Empaquetado por version` (verde = correcto).
2. **Releases** → nueva entrada `vX.Y.Z` con el asset `source-vX.Y.Z.zip`.
3. La prerelease aparece con la etiqueta **Pre-release**.

### Prueba local del empaquetado (sin GitHub)

```bash
git archive --format=zip --output=source-test.zip HEAD
unzip -l source-test.zip     # comprueba qué incluye el ZIP
```

---

## 6. Preguntas frecuentes / errores comunes

| Síntoma | Causa y solución |
|---|---|
| El workflow no se ejecuta | El commit no tocó `version.txt`, o el push no fue a `main`. Revisa el filtro `paths` |
| `La version 'X' no es SemVer valida` | Formato incorrecto (`0.1`, `v0.1.1`, texto). Corrige `version.txt` |
| `version.txt sigue en X` | Hiciste push sin cambiar la versión. Edítala y vuelve a hacer push |
| `El tag vX.Y.Z ya existe` | Esa versión ya se publicó. Sube el número (protocolo: nunca se reutiliza) |
| `403 Resource not accessible` | Faltan permisos de escritura: Settings → Actions → General → Workflow permissions |
| El ZIP incluye archivos que no quiero | Agrega la ruta con `export-ignore` en `.gitattributes` |
| Re-ejecuté el workflow y falló | El tag ya existe por la ejecución anterior; es el comportamiento esperado |

---

## 7. Personalización

- **Otra rama:** cambia `branches: [main]` por la que uses.
- **Más artefactos:** agrega archivos con `git archive` o pasos de build y lístalos en
  `files:` (acepta múltiples líneas).
- **Build real (Node, Python, Go…):** reemplaza el paso *Crear ZIP del código fuente*
  por los pasos de build y empaqueta la carpeta de salida (ej: `dist/`).
- **Disparo manual adicional:** agrega `workflow_dispatch:` bajo `on:` para poder
  lanzarlo desde la pestaña Actions.
- **Notas de release propias:** el `body:` del workflow ya muestra el cambio de versión;
  puedes ampliarlo o quitar `generate_release_notes`.

---

## 8. Resumen

1. Editas **solo** `version.txt` con una versión SemVer.
2. Push a `main` → el workflow detecta el cambio automáticamente.
3. `0.1.1` sin sufijo → **Release estable**; `0.1.2-beta.1` → **Prerelease**.
4. Siempre se adjunta `source-vX.Y.Z.zip` con el código fuente.
