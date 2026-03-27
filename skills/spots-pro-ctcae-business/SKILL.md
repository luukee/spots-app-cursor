---
name: spots-pro-ctcae-business
description: >-
  Guides PRO-CTCAE-aligned symptom business logic for SPOTS: question
  progression, response validation (0–4 scales), cart and reporting flows, and
  types under app/_lib. Use when changing symptom questions, cart behavior,
  Problem Station, search-to-report flows, or REDCap field mapping for
  frequency/severity/interference. Triggers: PRO-CTCAE, symptom cart, question
  flow, validation, SymptomBusiness, prior_problems, body_parts, feelings.
---

# SPOTS PRO-CTCAE business logic

Implement and change **rules and data shapes** here; keep UI thin (hooks/components call services). Legacy **SpotSymptoms (C#)** and JSON under SpotSymptoms are **requirements references only** — do not port legacy code verbatim.

## Where code lives

| Area | Location |
|------|----------|
| PRO-CTCAE question generation, progression, validation | `app/_lib/services/pro-ctcae/question-service.ts` |
| Symptom cart add/remove/update/submit | `app/_lib/services/symptoms/symptom-cart-service.ts` |
| Business types (questions, symptoms, cart items) | `app/_lib/types/business/pro-ctcae.ts`, `app/_lib/types/business/symptoms.ts` |
| REDCap-facing symptom/report shapes & 0–4 scales | `app/_lib/types/redcap/redcap-business.ts` |
| React integration | `app/_lib/hooks/symptoms/` (e.g. `useSymptomCart.ts`, `useSymptoms.ts`, `useSymptomQuestions.ts`) |
| Unit tests (patterns for question logic) | `app/_lib/services/pro-ctcae/__tests__/question-business.test.ts` |

Extend existing modules before adding parallel “second” implementations.

## PRO-CTCAE scales and consistency

- Responses for standard domains use **0–4** ordinals; model them with the project’s union types (e.g. `PROCTCAEFrequency`, `PROCTCAESeverity`, `PROCTCAEInterference` in `redcap-business.ts`), not plain `number`, when wiring business logic.
- **Progression** (which domains apply: frequency, severity, interference, presence) is **category- and subcategory-dependent** — follow the rules encoded in `question-service.ts` (`QUESTION_PROGRESSION_RULES`, subcategory maps). When adding a category or subcategory, update those maps and tests together.

## Question text and options (REDCap / PPC)

- **All user-visible question and option strings must come from PPC / REDCap-backed data** for real runs. The service uses explicit sentinels (e.g. `MISSING_QUESTION_FROM_REDCAP`) and guards against showing raw REDCap variable names — preserve that behavior when touching formatting.
- **Do not** add runtime “fallback” copy that masks missing REDCap labels; missing data should be visible as a data/integration issue so it gets fixed upstream (aligns with `spots-api-redcap-integration`).

## Symptom cart and reporting

- Cart mutations and validation belong in **services**; hooks orchestrate React state and call services.
- When mapping cart items to API payloads, keep field names aligned with backend/REDCap expectations (symptom identifiers, repeat instances, parent vs child fields — follow existing `symptom-cart-service.ts` and API docs).
- **Soft delete** and list filtering rules (e.g. `symptom_deleted`) are behavioral requirements: follow `.cursor/rules` and feature docs; do not reintroduce deleted items in lists.

## Validation and edge cases

- Validate at the **boundary** of the service (incoming symptom, cart item, or API DTO) with clear, typed results; avoid `any`.
- Respect **parent vs child** question variants when types or REDCap fields differ (see types under `symptoms.ts` / `question-service`).

## Testing

- Add or update **unit tests** alongside `question-service` when changing progression, validation, or label-sanitization behavior.
- Integration tests that need real symptom payloads should go through **Gateway APIs**, not static JSON substitutes in production code paths (tests may use fixtures).

## Related skills

- **`spots-api-redcap-integration`** — proxy patterns and no mock fallbacks for live routes.
- **`spots-security-privacy`** — no PHI in logs when debugging question/cart flows.

## Reference

- PRO-CTCAE overview: [https://healthcaredelivery.cancer.gov/pro-ctcae/](https://healthcaredelivery.cancer.gov/pro-ctcae/)
