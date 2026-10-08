---
title: The one substantive type-design gap is that `field` is typed as a bare `string`
kind: preference
summary: The one substantive type-design gap is that `field` is typed as a bare `string` everywhere it's threaded through (`IBlendValidationError.field`, `CUSTOM_LOGIC_OWNED_FIELDS`, `BLEND_VALIDATION_FIELD_LA
category: habits
topic: bare blend custom design field fields gap iblendvalidationerror la logic one owned string substantive threaded through type typed validation
applies_to: []
projects: 1
first_seen: 2026-09-01
last_seen: 2026-09-01
confidence: 0.8
generality: general
evidence:
- project: default/api.higg.org
  source: session:0c01c1d2-f85d-4ad7-9846-3bbdac126c1c
  date: 2026-09-01
  quote: The one substantive type-design gap is that `field` is typed as a bare `string` everywhere it's threaded through (`IBlendValidationError.field`, `CUSTOM_LOGIC_OWNED_FIELDS`, `BLEND_VALIDATION_FIELD_LABELS`'s `Record&lt;string, string&gt;` keys) despite being a closed, 5-6-member set that the code itself already treats as fixed and warns future maintainers to keep in sync by hand.
generated_by: profile-harvest
generated_sha: 60cb66636fcbead84d7172c0796de87ed3c7d75d3a6b11cd9e3dd92d99e82acf
tier: semantic
type: Note
description: The one substantive type-design gap is that `field` is typed as a bare `string` everywhere it's threaded through (`IBlendValidationError.field`, `CUSTOM_LOGIC_OWNED_FIELDS`, `BLEND_VALIDATION_FIELD_LA
generated:
  by: process:ai-memory/2.6.1
  at: 2026-10-08T19:06:53Z
---
# The one substantive type-design gap is that `field` is typed as a bare `string`

The one substantive type-design gap is that `field` is typed as a bare `string` everywhere it's threaded through (`IBlendValidationError.field`, `CUSTOM_LOGIC_OWNED_FIELDS`, `BLEND_VALIDATION_FIELD_LA

## In your words

- "The one substantive type-design gap is that `field` is typed as a bare `string` everywhere it's threaded through (`IBlendValidationError.field`, `CUSTOM_LOGIC_OWNED_FIELDS`, `BLEND_VALIDATION_FIELD_LABELS`'s `Record&lt;string, string&gt;` keys) despite being a closed, 5-6-member set that the code itself already treats as fixed and warns future maintainers to keep in sync by hand." (default/api.higg.org, 2026-09-01)
