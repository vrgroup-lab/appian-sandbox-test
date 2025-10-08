## ✅ Acción requerida

1. Ve a **Settings → Secrets and variables → Actions → Repository secrets**.
2. Edita el secreto `ICF_JSON_OVERRIDES` y agrega/actualiza las claves necesarias para esta promoción.
3. Usa la plantilla `provisioning/icf-template.properties` como referencia para los valores obligatorios.
4. Una vez actualizada la configuración, guarda el secreto y continúa con la aprobación en GitHub Actions.

### Contexto de la ejecución
- Plan seleccionado: `{{PLAN}}`
- Dry run: `{{DRY_RUN}}`
- Directorio de artefacto: `{{ARTIFACT_DIR}}`
- Metadata export: `{{METADATA_PATH}}`
- Ejecución: {{RUN_URL}}

### Entornos objetivo detectados
{{TARGETS_LIST}}

> Cierra esta issue cuando los overrides estén listos. Si la promoción se repite, se generará una issue nueva.

{{TEMPLATE_SECTION}}

### Plantilla sugerida para `ICF_JSON_OVERRIDES`
```json
{{OVERRIDES_JSON}}
```
