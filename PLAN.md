# PLAN

## Qué hay hoy
- `.github/workflows/deploy-app.yml:317-334` y `.github/workflows/deploy-package.yml:325-343` invocan `python3 .github/scripts/create_icf_issue.py` dentro del job `archive_export`, generando una issue por ejecución para pedir que el promotor copie un JSON en los secretos `ICF_JSON_OVERRIDES_*`.
- Los jobs `promote_*` de ambos workflows ejecutan `vrgroup-lab/appian-cicd-core/.github/actions/appian-build-icf@flow-b` (`deploy-app.yml:436-445`, `deploy-package.yml:467-476`, `deploy-app.yml:586-595`, etc.) que lee el secreto `ICF_JSON_OVERRIDES_*` en formato JSON, combina los valores (según la plantilla descargada) y emite un `customization.properties` temporal.
- El archivo resultante se pasa al Core sin más transformación mediante la entrada `icf_path` de `appian-promote@flow-b` (`deploy-app.yml:447-456`, `deploy-package.yml:497-505`, `deploy-app.yml:597-605`, etc.).

## Qué cambia
- Eliminar los pasos “Crear issue con guía para ICF” de ambos workflows y dejar de usar `create_icf_issue.py`/`.github/templates/icf-issue.md`, manteniendo el resto del job `archive_export` intacto.
- Mantener los pasos `appian-build-icf@flow-b` dentro de `promote_*`, pero documentar que el Core debe dejar de esperar JSON y pasar a parsear texto plano `clave=valor`. Mientras tanto el wrapper seguirá proveyendo el secreto como hoy (no añadimos script intermediario).
- Preparar `PROMPT_CORE.md` con la información que necesita el equipo Core para:
  1. cambiar el parser de `appian-build-icf` a formato `key=value`,
  2. aceptar `ICF_JSON_OVERRIDES_*` como texto plano desde los secretos,
  3. conservar salidas (`icf_path`) para `appian-promote`.
- Añadir README/CHANGELOG breve explicando que la issue automática desaparece y que los secretos ahora deben guardarse como `clave=valor`. Incluir un snippet de ejemplo en el README.

## Riesgos y compatibilidad
- **Secretos obsoletos**: una vez que el Core cambie el parser, los secretos deben haberse actualizado al nuevo formato o el build fallará; necesitamos coordinar el rollout.
- **Formato de líneas**: el Core debe validar líneas sin `=` y reportar errores amigables sin mostrar valores completos.
- **Protección de secretos**: asegurarnos vía prompt que el Core registra solo cantidades/hash y nunca los valores literales.
- **Paridad con el Core**: `appian-promote` seguirá recibiendo la misma ruta; el Core debe garantizar que el `customization.properties` final mantenga el formato esperado.

## Pruebas mínimas
1. **Caso feliz**: secret con 2–3 líneas válidas → script genera archivo, reporta `count` y hash, y `appian-promote` lo consume sin errores.
2. **Secret vacío o no definido**: el paso falla inmediatamente con mensaje `::error::Secret ICF_JSON_OVERRIDES_* vacío` (sin revelar contenido).
3. **Línea inválida**: incluir una línea sin `=` y verificar que el script corta la ejecución con un mensaje breve que incluya número de línea.
4. **Valores con `=` y CRLF**: secret con `foo=bar=baz` y terminaciones Windows debe mantener `value` completo (`bar=baz`) y normalizar saltos de línea a `\n`.

## Rollback simple
- Revertir el commit que elimina la issue y las notas del README. En paralelo, pedir al Core que mantenga compatibilidad dual (JSON/`key=value`) o volver a la versión anterior de `appian-build-icf` mientras se migran los secretos.
