# ADR-0008: Localization model and the `LocalizedText` value object

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/10-localization-and-currency.md](../gaps/resolved/10-localization-and-currency.md)
- **Builds on**: [ADR-0005](0005-catalog-and-product-modeling.md), [ADR-0007](0007-pricing-tax-and-vouchers.md) (currency is already settled there)
- **Introduces**: the `LocalizedText` value object; `id` as the platform default locale; `Store.supported_locales`.

## Context

Currency policy was settled by [ADR-0007](0007-pricing-tax-and-vouchers.md). Gap 10's remaining concern is **language / locale** for buyer-facing descriptive content.

The platform's target market is multilingual (Indonesia: Bahasa Indonesia is the lingua franca but English and regional languages are common in commerce). Multi-language content is therefore not optional — the model supports stores publishing in multiple languages from day one, with `id` as the platform default.

Decisions made here propagate into every Catalog string field (name, description, attribute labels, category labels, voucher description, media alt-text) and into the Beckn projection logic in the Bridge.

## Options considered

### Per-field locale model

**A. Single-string fields with a `Store.language` attribute.** Rejected — does not support multilingual stores, which is a v1 requirement.

**B. `LocalizedText` value object: map of locale → string, with a required default-locale entry** — chosen.

**C. Single canonical + automatic translation pipeline.** Rejected — machine translation quality, cost, and brittleness; auto-translation is out of scope for v1.

### Default locale

**A. Platform-wide default = `id` (Bahasa Indonesia)** — chosen. Indonesian is the platform's primary language; matches ion-spec convention. The default is enforced as an invariant on every `LocalizedText`.

**B. Per-store default.** Rejected for v1 simplicity. Can be added later if stores operating in other primary languages become a meaningful fraction.

### Store-level locale declaration

**A. `Store.supported_locales` is an explicit attribute** — chosen. Declares which locales the store actively publishes in. Used by the storefront language switcher, by Beckn projection, and by owner UX. Always includes the default; default at creation is `[id]`.

**B. Supported locales derived from authored content.** Rejected — loses the distinction between "I haven't translated yet" and "I don't support this language."

### PlatformCategory localization

**A. Single-language category labels.** Rejected by user — would force category names to appear in the platform language regardless of store language.

**B. Multi-locale labels on PlatformCategory via `LocalizedText`** — chosen.

## Decision

### 1. `LocalizedText` value object

`LocalizedText` is a domain value object representing a string that may have translations.

Structure:
- `entries`: `Map<bcp47_tag, string>` — locale-to-text mapping.

Invariants:
- `entries` is non-empty.
- `entries` always contains an entry for the **platform default locale** (`id`).
- Keys are valid BCP 47 language tags.

Operations:
- `get(locale)` — returns `entries[locale]` if present; else `entries[default]` (fallback to platform default).
- `get_strict(locale)` — returns `entries[locale]` if present; else null.
- `with(locale, text)` — returns a new `LocalizedText` with that entry added or replaced.
- `without(locale)` — returns a new `LocalizedText` with that entry removed; **cannot remove the default**.
- `available_locales()` — returns the set of locales present.

### 2. Platform default locale

The platform default locale is **`id`** (Bahasa Indonesia, ISO 639-1). Configured at platform setup; not per-store. Every `LocalizedText` must have an entry for `id`.

### 3. Locale tag format

**BCP 47 language tags** throughout. Examples: `id`, `en`, `en-US`, `en-ID`, `ms`, `jv`, `su`, `zh-Hans`.

### 4. Translatable fields

Fields that become `LocalizedText`:

- **Store** — name, description, public contact display text.
- **Product** — name, description.
- **ProductAttribute** (Matrix mode) — name (the attribute label, e.g., "Size" / "Ukuran"). Allowed-value labels may also be `LocalizedText` when display labels need translation; the **value identity** itself remains a locale-neutral string.
- **Media** — alt_text.
- **Voucher** — description (this ADR adds `description: LocalizedText` to the `Voucher` entity defined in [ADR-0007](0007-pricing-tax-and-vouchers.md); it was not previously specified).
- **PlatformCategory** — name.

Locale-neutral (single string or non-string, regardless of language):
- All internal identifiers, opaque IDs, cross-context references.
- Slugs, SKUs, voucher codes.
- `Money` amounts, ISO 4217 currency codes, timestamps, dates.
- Boolean flags, enum values, status fields.

### 5. `Store.supported_locales`

Each Store carries `supported_locales` — an ordered list of BCP 47 tags representing locales the store actively publishes in.

Rules:
- Always includes the platform default (`id`); cannot be removed.
- Default at store creation: `[id]`.
- Owner can add more locales as they author content.
- Adding a locale **does not** require backfilling translations across every field — `LocalizedText.get(locale)` falls back to the default for unauthored locales.

Consumers:
- **Storefront** uses it for the language switcher.
- **Beckn Bridge** declares supported languages in the provider descriptor.
- **Admin UI** uses it to surface which language fields to offer the owner.

### 6. Uniqueness with `LocalizedText`

Where uniqueness applies to a translatable field (e.g., `Product.name` is unique within store), the check uses the **default-locale value** (`id`). Cross-locale collisions are not checked at v1. This keeps the rule simple and decidable.

### 7. Beckn locale handling

When the Bridge projects content into Beckn responses:

- The BAP's request includes a language preference via `context.language` (or equivalent for the protocol version).
- For each `LocalizedText` field in the projected entity, the Bridge calls `get(requested_locale)`. If the locale is present, that text is used; otherwise the default-locale value is used.
- The Bridge declares in the response which locale was used per descriptor block.
- **No automatic translation.** If the BAP requested `ja` and the store only has `id` + `en`, the BAP receives the `id` text.
- The Bridge declares `Store.supported_locales` in the provider descriptor so BAPs know what's available.

### 8. PlatformCategory localization

System Admins author `PlatformCategory.name` as `LocalizedText`. The platform default (`id`) is required at creation; other locales may be added.

If a store operates in a locale that the System Admin has not yet authored for a category, the storefront displays the default-locale label (graceful degradation). Notifications to System Admins about missing translations are operational, not architectural.

## Consequences

What this commits to:

- **`LocalizedText` is a first-class domain value object** used wherever buyer-facing descriptive text appears across contexts.
- **Every translatable field** named in §4 above is `LocalizedText` in its respective entity. [`design/catalog.md`](../design/catalog.md) is updated to reflect this for Product, ProductAttribute, Media, and PlatformCategory.
- **`Store.supported_locales` is a new attribute** on the Store entity, owned by the Tenancy context.
- **`Voucher.description: LocalizedText`** is added to the Voucher entity from [ADR-0007](0007-pricing-tax-and-vouchers.md).
- **BCP 47** is the canonical tag format throughout the system.
- **The Bridge** must accept a per-request locale preference, resolve each LocalizedText accordingly, and declare the chosen locale in its response.
- **Uniqueness checks** on translatable fields apply to the default-locale value.
- **Audit and event payloads** that carry translatable content should record the full `LocalizedText` (all locales), not just one rendering, so history is locale-agnostic.

What this defers:

- **Automatic / machine translation** — out of scope; owners author translations manually.
- **Per-store override of the platform default locale** — `id` is the default for all stores in v1.
- **Translation memory / suggested translations** — UX feature.
- **Right-to-left layout** — display concern at the Interface Layer.
- **Date and number formatting per locale** — display concern.
- **Locale-specific search and indexing** — search-layer concern.
- **Notifications to System Admins about missing category translations** — operational.

What this makes harder:

- **Migrating to a different platform default locale**. The `id` requirement is baked into every `LocalizedText`. Changing it later would require a system-wide migration.
- **Single-language stores**. A store that only ever operates in English must still author content in `id` (the default). If they don't, they either populate the `id` slot with English text (labeled as `id`) or leave a gap that breaks the invariant.
- **Cross-locale uniqueness checks**. If two products have different default-locale names but identical English names, the system permits it. Acceptable trade-off for simplicity.

## References

- [gaps/resolved/10-localization-and-currency.md](../gaps/resolved/10-localization-and-currency.md)
- [ADR-0005](0005-catalog-and-product-modeling.md) (translatable entities introduced)
- [ADR-0007](0007-pricing-tax-and-vouchers.md) (currency, the sibling concern; also adds `Voucher` which gains a `description` here)
- [design/catalog.md](../design/catalog.md) (updated for LocalizedText)
- `ion-specs` — language tag conventions; `id` as default
- BCP 47 — language tag standard
