# Multi-Storefront Advisor — Decision Logic

**Version:** 1.0  
**Last Updated:** December 5, 2024  
**Live App:** https://annafy2024.github.io/multi-storefront-advisor/

---

## Overview

This document explains the recommendation algorithm used by the Multi-Storefront Architecture Advisor. The tool guides users through 10 questions and outputs a recommended Shopify architecture based on weighted scoring and hard constraints.

---

## Architecture Options

The tool evaluates 5 architecture patterns:

| Option | Name | Description |
|--------|------|-------------|
| **A** | Single Store + Sections | One Shopify store with different sections/collections per brand |
| **B** | Single Store + Alias Domains | One store with multiple domains, conditional Liquid rendering |
| **C** | Multiple Stores (Headed) | Separate Shopify stores with native Liquid themes |
| **D** | Single Store + Headless | One store backend with multiple custom frontends (Hydrogen, etc.) |
| **E** | Multiple Stores + Headless | Separate stores, each with custom headless frontends |

### Feature Matrix

| Capability | A | B | C | D | E |
|------------|:-:|:-:|:-:|:-:|:-:|
| Unique domains per brand | ❌ | ✅ | ✅ | ✅ | ✅ |
| Brand-specific checkout | ❌ | ❌ | ✅ | ❌ | ✅ |
| Separated customer accounts | ❌ | ❌ | ✅ | ❌ | ✅ |
| Native shared cart | ✅ | ✅ | ❌ | ✅ | ❌ |
| Per-brand staff permissions | ❌ | ❌ | ✅ | ❌ | ✅ |
| Clean B2B separation | ❌ | ❌ | ✅ | ❌ | ✅ |

---

## Algorithm Structure

### Phase 1: Hard Constraints

Certain requirements **eliminate** architecture options entirely. These are evaluated first and override all scoring.

#### Constraint 1: Brand-Specific Checkout Required
```
IF checkout_branding === 'required'
THEN:
  - scores.A = -100 (eliminated)
  - scores.B = -100 (eliminated)
  - scores.D = -100 (eliminated)
  - scores.C += 10
  - scores.E += 10
  - Add constraint: "Brand-specific checkout required → Eliminates single-store options"
```

**Rationale:** Shopify checkout is tied to the store backend. 1 store = 1 checkout design. This is not composable even with headless frontends.

#### Constraint 2: Separated Customer Accounts Required
```
IF customer_accounts === 'separated'
THEN:
  - scores.A = -100 (eliminated)
  - scores.B = -100 (eliminated)
  - scores.D = -100 (eliminated)
  - scores.C += 10
  - scores.E += 10
  - Add constraint: "Separated customer accounts required → Eliminates single-store options"
```

**Rationale:** Customer accounts are tied to the store. 1 store = 1 unified account showing all order history.

#### Constraint 3: Shared Cart Required (All Brands)
```
IF shared_cart === 'required'
THEN:
  - scores.C -= 5
  - scores.E -= 5
  - scores.A += 5
  - scores.B += 5
  - scores.D += 5
```

**Rationale:** Native shared cart only works within a single Shopify store. Multi-store shared cart requires complex custom solutions (query parameter cart passing, cross-domain cookie workarounds).

---

### Phase 2: Weighted Scoring

After constraints are applied, remaining questions add/subtract points based on fit.

#### Q1: Domain Structure

| Answer | Impact |
|--------|--------|
| `single` | A += 10 |
| `subdomains` | B += 5, D += 5 |
| `unique` or `mixed` | C += 3, E += 3 |

**Reasoning:**
- Single domain naturally fits Option A
- Subdomains enable shared cookies (easier cart sharing) → favors single-store options
- Unique root domains work best with store separation

#### Q2: Storefront Driver (Multi-Market vs Multi-Brand)

| Answer | Impact |
|--------|--------|
| `multi_market` | A += 3, B += 5, C -= 2 |
| `multi_brand` | C += 3, D += 3, E += 3 |
| `both` | D += 5, E += 3 |

**Reasoning:**
- **Multi-market (same brand internationally):** Shopify Markets + Liquid themes handle international expansion natively. Theme customizer allows market-specific content, Price Lists handle localized pricing. No headless required.
- **Multi-brand (distinct brands):** Distinct brand identities benefit from architectural separation — either via Hydrogen multi-storefront (single backend) or separate stores.
- **Both (multi-brand + international):** Complex scenario where single store + Hydrogen + Markets scales best. Multi-store multiplies sync complexity across brands AND markets.

**Why This Matters:**
Many merchants conflate "multi-storefront" needs that are actually:
- **International expansion** (same brand) → Shopify Markets handles this natively
- **True multi-brand** (different brands) → Requires architectural decisions

We shouldn't push merchants to headless unnecessarily when Markets solves their problem.

#### Q3: Checkout Branding

| Answer | Impact |
|--------|--------|
| `required` | **HARD CONSTRAINT** (see above) |
| `shared` or `dynamic` | A += 3, B += 3, D += 3 |

#### Q3: Customer Accounts

| Answer | Impact |
|--------|--------|
| `separated` | **HARD CONSTRAINT** (see above) |
| `unified` or `sso_filtered` | A += 2, B += 2, D += 2 |

#### Q4: Shared Cart

| Answer | Impact |
|--------|--------|
| `required` | **HARD CONSTRAINT** (see above) |
| `partial` | C += 2, E += 3, Add constraint about hybrid approach |
| `not_required` or `nice_to_have` | No impact |

**Partial Shared Cart Logic:**
When some brands need shared cart but others don't, the recommendation shifts to multi-store with a hybrid approach: group the brands needing shared cart on one store, keep independent brands separate.

#### Q5: B2B Requirements

| Answer | Impact |
|--------|--------|
| `none` or `single_brand` | No significant impact |
| `blended` | C += 3, E += 3 |
| `separate_portals` or `multi_brand_b2b` | C += 5, E += 5 |

**Reasoning:** B2B complexity (company management, price lists, payment terms) is cleaner with separate stores per brand.

#### Q6: Product Catalog

| Answer | Impact |
|--------|--------|
| `same_products` | No significant impact |
| `completely_different` | C += 3, E += 3 |
| `overlap_exclusives` | No significant impact |
| `separate_plus_aggregator` | C += 4, E += 4, Add constraint about sync |
| `aggregated` | D += 3 |

**Separate + Aggregator Logic:**
When brands have separate catalogs but one storefront aggregates all (e.g., Supply Gear pattern), multi-store is favored because:
- Each brand maintains clean catalog separation
- Aggregator pulls products via sync app or API
- Single-store would require complex tagging/filtering

#### Q7: Inventory Management

| Answer | Impact |
|--------|--------|
| `single_warehouse` | No significant impact (all options work) |
| `separate_inventory` | Slight favor to multi-store (natural fit) |
| `multi_location` | Consider sync complexity |

*Currently minimal scoring impact — included for completeness and future refinement.*

#### Q8: Team Capacity (UPDATED with Agency Option)

| Answer | Impact |
|--------|--------|
| `solo` | A += 5, C += 3, D -= 5, E -= 5 |
| `small_team` | A += 3, C += 2, D -= 3, E -= 3 |
| `agency_augmented` | C += 2, D += 2 (no headless penalty) |
| `dev_team` or `enterprise` | D += 3, E += 3 |

**Reasoning:**
- **Solo:** Headed approaches only — limited resources can't support headless complexity.
- **Small team:** Headed preferred, but headless penalty is softer than solo.
- **Agency augmented (NEW):** Agency partnerships bridge capability gaps. Headless becomes viable when experienced agency handles implementation and provides knowledge transfer. No penalty applied.
- **Dev team/Enterprise:** Full headless options open up with experienced teams.

**Why This Matters:**
The original logic heavily penalized headless options for small teams. But reality shows:
- Many successful Plus merchants partner with agencies for implementation
- Agencies provide knowledge transfer to internal teams over time
- "Small team + agency" is a viable path to headless that shouldn't be ignored

#### Q9: Content Needs

| Answer | Impact |
|--------|--------|
| `minimal` | A += 2, B += 2, C += 2 |
| `moderate` | No significant impact |
| `heavy` or `cms_driven` | D += 5, E += 5 |

**Reasoning:** Content-heavy requirements favor headless for superior CMS integration and frontend flexibility.

#### Q10: Timeline & Budget

| Answer | Impact |
|--------|--------|
| `fast_cheap` | A += 5, D -= 3, E -= 3 |
| `balanced` | No significant impact |
| `quality` or `enterprise_timeline` | D += 2, E += 2 |

**Reasoning:** Speed and cost constraints favor simpler architectures.

---

### Phase 3: Recommendation Selection

```javascript
// Find highest score (excluding eliminated options with negative scores)
let best = Object.entries(scores)
  .sort((a, b) => b[1] - a[1])[0][0];

// Find alternatives (positive scores, not the best)
let alternatives = Object.entries(scores)
  .filter(([key, val]) => key !== best && val > 0)
  .sort((a, b) => b[1] - a[1])
  .slice(0, 2)
  .map(([key]) => key);
```

---

## Score Ranges & Interpretation

| Score Range | Interpretation |
|-------------|----------------|
| **-100** | Eliminated by hard constraint |
| **< 0** | Poor fit, likely eliminated |
| **0-5** | Weak fit |
| **6-10** | Moderate fit |
| **11-15** | Good fit |
| **16+** | Strong fit |

---

## Example Scenarios

### Scenario 1: A10-Style (Multi-Brand with B2B + Aggregator)

**Inputs:**
- Domains: Unique (`brand-a.com`, `brand-b.com`)
- Checkout: Brand-specific required
- Accounts: Separated
- Shared Cart: Partial (aggregator needs shared)
- B2B: Separate portals
- Catalog: Separate + aggregator
- Team: Small team
- Content: Moderate
- Timeline: Balanced

**Calculation:**
```
A: -100 (eliminated by checkout constraint)
B: -100 (eliminated by checkout constraint)
C: 10 (checkout) + 10 (accounts) + 2 (partial cart) + 5 (B2B) + 4 (catalog) + 3 (team) = 34
D: -100 (eliminated by checkout constraint)
E: 10 (checkout) + 10 (accounts) + 3 (partial cart) + 5 (B2B) + 4 (catalog) = 32
```

**Result:** Option C (Multiple Stores, Headed) — with note about product sync for aggregator

---

### Scenario 2: Content-Heavy Single Brand Portfolio

**Inputs:**
- Domains: Subdomains
- Checkout: Shared OK
- Accounts: Unified
- Shared Cart: Required
- B2B: None
- Catalog: Same products
- Team: Dev team
- Content: Heavy (CMS-driven)
- Timeline: Quality focus

**Calculation:**
```
A: 3 (checkout) + 2 (accounts) + 5 (cart) = 10
B: 5 (domains) + 3 (checkout) + 2 (accounts) + 5 (cart) = 15
C: -5 (cart penalty) = -5
D: 5 (domains) + 3 (checkout) + 2 (accounts) + 5 (cart) + 3 (team) + 5 (content) + 2 (timeline) = 25
E: 3 (team) + 5 (content) + 2 (timeline) - 5 (cart) = 5
```

**Result:** Option D (Single Store + Headless)

---

### Scenario 3: Fast Launch, Simple Setup

**Inputs:**
- Domains: Single domain
- Checkout: Shared OK
- Accounts: Unified
- Shared Cart: Not required
- B2B: None
- Catalog: Different sections
- Team: Solo operator
- Content: Minimal
- Timeline: Fast & cheap

**Calculation:**
```
A: 10 (domains) + 3 (checkout) + 2 (accounts) + 5 (team) + 2 (content) + 5 (timeline) = 27
B: 3 (checkout) + 2 (accounts) = 5
C: 3 (team) = 3
D: 3 (checkout) + 2 (accounts) - 5 (team) - 3 (timeline) = -3
E: -5 (team) - 3 (timeline) = -8
```

**Result:** Option A (Single Store + Sections)

---

## Constraints Output

When hard constraints are triggered, they appear prominently in the results:

```
⚠️ Hard Constraints Identified
- Brand-specific checkout required → Eliminates single-store options
- Separated customer accounts required → Eliminates single-store options
```

When implementation considerations create dependencies:

```
⚠️ Implementation Notes
- Aggregator storefront needs product sync solution from other brand stores
- Partial shared cart → Consider hybrid: brands needing shared cart on one store
```

---

## Reasoning Output

Each contributing factor is listed with its impact:

```
🎯 Why This Recommendation?

💡 Checkout branding required
   Only multi-store architectures support distinct checkout branding

💡 Separated accounts required  
   Only multi-store architectures support separate customer accounts

💡 Separate catalogs + one aggregator storefront
   Multi-store with product sync to aggregator. Each brand gets clean separation.

💡 Limited technical resources
   Headed approaches require less specialized development expertise
```

---

## Limitations & Future Improvements

### Current Limitations

1. **Binary scoring:** Some factors could benefit from more nuanced weighting
2. **No dependency detection:** Doesn't flag when answer combinations are contradictory
3. **Static weights:** Weights are fixed; could be improved with ML from actual implementations

### Potential Enhancements

1. **Confidence scoring:** Show how strong the recommendation is vs. alternatives
2. **Cost estimation:** Add approximate implementation cost ranges
3. **Timeline estimation:** Add rough timeline expectations per architecture
4. **Integration complexity:** Factor in ERP, OMS, and other system integrations
5. **International considerations:** Markets, multi-currency, regional requirements

---

## Reference Implementations

| Merchant | Pattern | Architecture |
|----------|---------|--------------|
| **Kao Brands** (us.jergens.com) | Multi-brand + international | Single store + Hydrogen multi-storefront + Markets |
| **Being Well** | Same-brand multi-market | Single store + Liquid alias domains |
| **Cobra/Puma Golf** | Multi-brand (distinct) | Separate stores (headed) |
| **Defyned Brands** | Multi-brand headless | Separate stores + custom frontends |
| **Urban Planet** | Sub-brands | Single store + sections |

---

## References

- **Source Material:** Multi-storefronts Enablement deck (Chris Hannaby, Matt Cohn, Oct 2024)
- **Validation:** A10 technical assessment case study
- **Live Examples:** Urban Planet (A), Being Well (B), Cobra/Puma Golf (C), Kao Brands (D), Defyned Brands (E)

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-12-05 | Initial logic documentation |

---

*For questions or feedback, contact the SE team.*

