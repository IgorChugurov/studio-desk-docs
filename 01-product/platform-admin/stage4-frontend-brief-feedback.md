# Stage 4 feedback — Platform Admin and Organization Admin shell

`pnpm lint`, `pnpm format:check`, and `pnpm test` in `studio-desk-frontend` passed. Lint exits 0 with the existing `next/no-img-element` warning in `src/studio/catalogs/hall-gallery.tsx`. Tests: 30 files, 167 passed.

Internal folders `src/app/studio`, `src/studio`, and `src/platform/studios` stayed. Public platform routes are `/` (organizations list), `/organizations/new`, `/organizations/[id]`, `/presets`, `/presets/new`, and `/presets/[id]`. The old `/studios` pages were removed. Organization admin browser URLs are unchanged.

The platform client calls `/organizations` and `/presets`. The organization client derives `/api/organization`. `next.config.ts` rewrites `/api/organization/:path*`. `NEXT_PUBLIC_STUDIO_ADMIN_URL` and `studioAdminUrl` are unchanged; that value is still the impersonation handoff URL.

Preset resource-type `key` is generated and not shown (`place`, then `resource`, `resource-2`, …). The brief has no key field; the API requires one. A new preset starts with every section enabled and one place card. Place cannot be removed. Geography, duration, person, and offering `conflictMode` are locked in the editor to the role rules. `Add resource type` adds a card whose role can be `resource` or `offering`.

Organization settings send only changed fields. A new resource type omits `id`. After a successful `PATCH /settings`, the signed-in session `config` is replaced from the response. `409 RESOURCE_TYPE_IN_USE` has no fixed screen text in the canon, so the form shows `Something went wrong. Try again`.

The modular editor labels (sections, cubes, settlement, resource-type cards) are the English `CONFIG_EDITOR_TEXTS` in every interface language. Navigation, staff, and `Organization settings saved` come from the `en` / `sk` / `uk` dictionaries. The fixed specialist and organization error sentences stay English in Slovak and Ukrainian.

Staff loads person resources with `GET /resources?resourceTypeId=` for types where `person` is true. One such type uses its localized singular label. Several use Specialist / Špecialista / Спеціаліст. None leaves only administrator and accountant. A specialist save sends `resourceId` only when the link changes. Leaving the specialist role sends `role` only.

`src/platform/api/schema.d.ts` still describes `/api/platform/studios`. The screens do not use those paths. Catalogs and Schedule still call the trainer, hall, and class-type endpoints. They compile and their tests pass; Stage 5 rewrites those screens.

The running `npm run dev` proxies `/api/organization` to `localhost:3000`, and that connection is refused. The new screens were not clicked against a live API.
