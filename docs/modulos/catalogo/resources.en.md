# mod.resources — Resource library

<span class="didacta-chip didacta-chip--community">Community</span> · **Engagement** category (can be disabled)

## What it does

A library of community resources, organised into **collections** and searchable: workflows from the classes, skills, an AI tool directory and templates. Each resource is either a file uploaded to the tenant's storage (`FILE`) or an external link (`LINK`), can be tied to the live class that produced it, and carries a download counter.

## How it works

- The first visit seeds **6 default collections**; staff can create more, each with a cover image.
- Any member can **share resources**; deleting is available to the author (for their own) or to staff (for any).
- Deleting a collection that still holds resources is **blocked on purpose** (`Restrict`, not cascade): you have to empty it first — content is never lost silently.
- `POST /:id/download` records the download/open and returns the URL: the counter feeds the popularity ordering.

## Dependencies

None. The reference to the class (`zoomSessionId`) is a logical ID with no foreign key.

## Data model

`mod_resources_collection` (title unique per tenant, ordering, default flag) · `mod_resources_resource` (kind FILE/LINK, URL, author, originating class, downloads).

## API

Prefix `/modules/resources`. Details in [Reference → Community and people](../../api/referencia/comunidad.md#resources-modulesresources).

## Events

**Emits**: `resources.resource.created`, `resources.resource.deleted`, `resources.collection.created` (the first two score points in [gamification](gamification.md)). It consumes none.

## Configuration

No configuration: only the module's activation toggle.
