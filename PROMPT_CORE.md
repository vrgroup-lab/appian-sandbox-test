# Contexto
- Los wrappers `deploy-app.yml` y `deploy-package.yml` ya no generan la issue “Completar ICF_JSON_OVERRIDES” ni proveen un JSON para copiar; ahora se espera que los secretos `ICF_JSON_OVERRIDES_QA/PROD` contengan texto plano en formato `clave=valor`, una asignación por línea.
- `appian-build-icf@flow-b` (Core) sigue siendo el componente que lee `ICF_JSON_OVERRIDES` y construye `customization.properties`, pero hoy asume que el secreto es un JSON (lo parsea y reescribe), por lo que las ejecuciones fallarán al recibir el nuevo formato.

# Cambio mínimo propuesto
Actualizar la acción compuesta `appian-build-icf` para que:
1. trate `ICF_JSON_OVERRIDES` como blob de texto plano;
2. procese cada línea (`\n` o `\r\n`), ignorando vacías y comentarios que empiecen con `#`;
3. exija al menos un `=` por línea (usando la primera ocurrencia como separador clave/valor; el resto del string pertenece al valor);
4. escriba las líneas válidas (normalizadas a `\n`) en el archivo `customization.properties` que ya retorna `icf_path`;
5. registre únicamente el conteo de líneas válidas y el `sha256` del archivo final, sin exponer los valores ni ejecutar `set -x`;
6. falle de inmediato con mensaje claro si:
   - el secreto está vacío o indefinido,
   - alguna línea no contiene `=`,
   - después del filtrado no queda ninguna línea válida.

# Archivos Core implicados
- `.github/actions/appian-build-icf/action.yml` (para documentar el nuevo contrato, inputs/outputs).
- Script o helper invocado por esa acción (ej. `scripts/build_icf.py` o el shell equivalente que hoy parsea JSON).
- Documentación del Core (README o CHANGELOG) para comunicar el nuevo formato `key=value`.

# Pruebas mínimas en el Core
1. **Happy path**: secreto con 3 líneas válidas → genera archivo, loguea `count=3`, `sha256=…`, produce el mismo `icf_path` que consume `appian-promote`.
2. **Secret vacío/no definido**: error rápido antes de tocar disco (`::error::ICF_JSON_OVERRIDES vacío`).
3. **Línea sin `=`**: identificar número de línea y fallar con mensaje breve sin imprimir el contenido completo.
4. **Valor con `=`**: confirmar que sólo se divide por la primera `=`.
5. **CRLF**: entrada con `\r\n` debe generar archivo normalizado a `\n`.

# Compatibilidad / Rollback
- Idealmente aceptar durante una transición tanto JSON (formato anterior) como `key=value`, detectando `^\s*{` para enrutar al parser viejo y emitiendo `::notice::formato deprecated`. Si no es viable, documentar que el cambio es breaking y coordinar rollout con wrappers.
- Rollback: reinstalar la versión previa de `appian-build-icf` (o restaurar el parser JSON) y revertir la documentación. Los wrappers siguen apuntando al mismo `icf_path`, por lo que no requieren cambios adicionales.
