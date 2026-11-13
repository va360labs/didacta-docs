# Telemetry

Every installation sends **one anonymous daily heartbeat** to `registry.didacta.io` so the project knows how many Didacta installations exist. The full payload is this and nothing else:

| Field | Content |
| --- | --- |
| `instanceId` | A **random** UUID generated on first run (the `.didacta-instance-id` file in the data volume). It identifies no person and no organization. |
| `version` / `edition` | The Didacta version and edition (`community` or the Enterprise plan). |
| `node` / `os` | Node version and platform (`linux/x64`…). |
| `sentAt` | The heartbeat date. |

No PII, no business data (no users, no courses, no domains) and nothing blocking: if there is no internet access, the heartbeat fails silently and the platform carries on.

## Disabling it

```bash
# In your .env
DIDACTA_TELEMETRY_DISABLED=true
```

## Opt-in registration (optional)

Beyond the anonymous heartbeat there is a **voluntary registration** (**Administration → Settings → Registration**) where the operator can identify themselves with an email address and organization in exchange for a direct channel to the team. That level sends aggregate metrics and offers **opt-out and GDPR erasure** from the panel itself.
