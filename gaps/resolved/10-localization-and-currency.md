# Gap 10 — Localization & currency

> **RESOLVED on 2026-05-28**. Currency policy resolved by [ADR-0007](../../decisions/0007-pricing-tax-and-vouchers.md); localization model resolved by [ADR-0008](../../decisions/0008-localization-and-localizedtext.md). Active rules in `CLAUDE.md` §5.12 (currency) and §5.14 (localization). This file is preserved for historical context.

## Statement

Beckn supports localized descriptors and per-context currency. The charter mentions `Money` as a value object (§3.2) but takes no stance on whether the system is single-locale, multi-locale, single-currency, or multi-currency. Because this decision is binary at every modeled attribute (one description vs. localized; one price vs. multi-currency), retrofitting it later is expensive.

## Current charter coverage

* §3.2 — `Money` is named as a value object.
* No mention of localization, language, or i18n.

## Open questions

1. **Multi-locale descriptors.** Are product descriptions / titles single-string, or a map of `locale → string`? Same question for category names, store names, attribute labels.
2. **Translation source.** Are translations entered by store owners, by the platform, or via automated translation?
3. **Locale negotiation.** When responding to a Beckn `on_search`, is the response in the store's locale, the buyer's requested locale (from `context.location.country` / `context.language`), or both?
4. **Currency cardinality.** One currency per store (likely), one per product, or full multi-currency catalogs?
5. **Currency conversion.** If multi-currency: are prices stored in one base currency and converted at display, or stored per-currency with no conversion?
6. **Currency on orders.** Locked at order creation, or recomputed on every read?
7. **Locale-aware identifiers.** Are slugs locale-specific? Search-indexable per locale?

## Implications

* Affects every Catalog string attribute.
* Influences the shape of the `Money` value object (currency-tagged amount, or amount-only with implicit currency).
* Determines whether the Bridge needs locale-aware projection logic.

## Dependencies

* Influences: 07 (catalog string fields), 09 (pricing storage).
* Worth resolving before deep Catalog work.
