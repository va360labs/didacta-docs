# Licenses

Didacta is **fair-code, source-available**, aligned with the [faircode.io](https://faircode.io/) movement. It is not open source under the OSI definition.

## What you can do

- ✅ View, read, copy, modify and distribute the code.
- ✅ Run Didacta in production for your **organisation's internal purposes** (training your employees, an internal LMS…), at no cost and with no per-user licensing.
- ✅ **Non-commercial or personal** use: research, learning, evaluation, contribution.
- ✅ Distribute the software or your modifications **free of charge, for non-commercial purposes**.
- ✅ Proofs of concept, evaluations, audits and demos.

## What requires a commercial agreement

- ❌ Distributing Didacta or derivatives **for a fee**.
- ❌ Offering Didacta as **paid SaaS or managed hosting** for third parties.
- ❌ Selling Didacta **white-labelled** or rebranded for resale.
- ❌ Sublicensing Didacta as part of a commercial product.
- ❌ Removing or altering copyright, license or trademark notices.
- ❌ Using the Didacta trademarks beyond what applicable law permits.

For any of these cases: **licensing@didacta.io**, or use Didacta Cloud (`cloud.didacta.io`).

## Two licenses, one repository

| Code | License |
| --- | --- |
| Everything that is **not** `*.ee.*` and not inside `ee/` folders | [Didacta Sustainable Use License v1.0](https://github.com/va360labs/didacta-io/blob/main/LICENSE) — the fair-code sustainable use license |
| `*.ee.*` files, or files inside `ee/` folders (in the core only) | [Didacta Enterprise License](https://github.com/va360labs/didacta-io/blob/main/LICENSE_EE) |
| Third-party components | Their original upstream license |

Files under the Enterprise License require a **valid Enterprise license key** to be used in production; without one, the License SDK gates them at runtime (see [Enterprise](../enterprise/index.md)). **Modules are never** under the Enterprise License: all of them are Community.

## Reference documents

- [LICENSE_NOTICE.md](https://github.com/va360labs/didacta-io/blob/main/LICENSE_NOTICE.md) — a human-readable summary of the model.
- [COMMERCIAL_USE.md](https://github.com/va360labs/didacta-io/blob/main/COMMERCIAL_USE.md) — the commercial use policy.
- [TRADEMARKS.md](https://github.com/va360labs/didacta-io/blob/main/TRADEMARKS.md) — "Didacta" is a trademark of VA360 LABS S.L.; read it before using the name or the logo in forks or derivatives.
