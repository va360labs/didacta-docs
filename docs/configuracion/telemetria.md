# Telemetría

Cada instalación envía **un latido diario anónimo** a `registry.didacta.io` para que el proyecto sepa cuántas instalaciones de Didacta existen. El payload completo es este y nada más:

| Campo | Contenido |
| --- | --- |
| `instanceId` | UUID **aleatorio** generado en la primera ejecución (fichero `.didacta-instance-id` en el volumen de datos). No identifica a ninguna persona ni organización. |
| `version` / `edition` | Versión de Didacta y edición (`community` o el plan Enterprise). |
| `node` / `os` | Versión de Node y plataforma (`linux/x64`…). |
| `sentAt` | Fecha del latido. |

Sin PII, sin datos de negocio (ni usuarios, ni cursos, ni dominios) y sin bloquear nada: si no hay salida a internet, el latido falla en silencio y la plataforma funciona igual.

## Desactivarla

```bash
# En tu .env
DIDACTA_TELEMETRY_DISABLED=true
```

## Registro opt-in (opcional)

Aparte del latido anónimo existe un **registro voluntario** (**Administración → Registro**) donde el operador puede identificarse con email y organización a cambio de un canal directo con el equipo. Ese nivel envía métricas agregadas y tiene **opt-out y borrado RGPD** desde el propio panel.
