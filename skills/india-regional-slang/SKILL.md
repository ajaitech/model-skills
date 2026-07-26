---
name: india-regional-slang
description: >-
  Configure agents to understand and reply in India's regional languages,
  code-mixed dialects, romanized or transliterated input, and colloquial register
  while keeping technical terms and numbers correct. Use when localizing a Bedrock
  agent or chatbot for Indian users, mirroring language, script, and register,
  handling lakh and crore, and mixing English technical terms naturally.
---

# India regional languages, code-mixing & colloquial register

Indian users rarely write "pure" anything — they mix their mother tongue with
English, often in **Latin script** (romanized), in a casual register. An agent
that mirrors that feels native. Goal: **understand messy multilingual input and
reply in the user's language + script + register**, without corrupting facts.

## The languages & code-mix names (detect, don't assume)
Major: Hindi, Bengali, Telugu, Marathi, Tamil, Urdu, Gujarati, Kannada, Malayalam,
Odia, Punjabi, Assamese (+ English as the link language). Common code-mix blends:

| Blend | Mix | Example (romanized) |
|---|---|---|
| Hinglish | Hindi + English | "boss, invoice ka payment kab aayega?" |
| Tanglish | Tamil + English | "anna, overdue invoices epdi check pannardu?" |
| Tenglish | Telugu + English | "balance enta undi cheppandi?" |
| Kanglish | Kannada + English | "report yaava time ge banthu?" |
| Manglish | Malayalam + English | "ee customer-inte balance ethra aanu?" |
| Banglish | Bengali + English | "ei month-er sales koto holo?" |

Input may be in native script (देवनागरी/தமிழ்/తెలుగు…) OR romanized OR mixed mid-sentence.

## Agent configuration (bake into the instruction)
> "Detect the user's language and script per message. **Reply in the same language,
> script, and register** (formal vs casual) the user used — including romanized
> code-mix if that's how they wrote. Keep proper nouns, product names, and
> technical/ERP field names in English. Render money in the **Indian system**
> (lakh = 1,00,000; crore = 1,00,00,000) with the Indian digit grouping (₹5,00,000).
> Be warm and respectful; use region-appropriate honorifics (ji / anna / bhai /
> sir/madam) only if the user's tone invites it. Never guess a fact to sound fluent —
> stay grounded (see erp-agent-grounding)."

## Practical handling
- **Romanized → meaning:** don't require native script; "kitna", "evlo", "entha",
  "eshto" all mean "how much" — interpret intent, not spelling. Spelling varies
  wildly (transliteration has no single standard); be tolerant.
- **Numbers/finance:** "5 lakh" = 500000; "2.5 cr" = 25000000; group as 2,50,00,000.
  ERP queries must convert these to plain integers before SQL (erp-agent-grounding).
- **Mixed sentences:** keep the English tech tokens English ("payment", "invoice",
  "GST", "ledger") and translate only the natural-language scaffolding.
- **Code-switch back:** if the user switches to English, switch with them.

## Bedrock wiring
- Set the above as the agent `--instruction`; the model (Claude/Gemini-class) is
  already multilingual — you're steering register + Indian conventions, not teaching
  the language.
- **Guardrail**: keep PII/finance controls language-agnostic; ensure denied topics
  trigger in all input languages (test guardrails with romanized + native script).
- **Transliteration help**: for native-script output where the user wrote romanized,
  reply romanized (match them) unless they used native script.
- **KB content**: store regional-language FAQ/docs in the KB so retrieval grounds
  answers in the user's language (tag with `lang` metadata; filter on it).

## Verify
1. Send Hinglish/Tanglish/Tenglish/Kanglish/Manglish/Banglish + native-script samples →
   each reply mirrors the language, script, and register.
2. "5 lakh", "2.5 cr" → correct integers + Indian grouping.
3. A finance question in romanized Tamil still hits the grounded ERP path and cites the source.
4. Guardrail blocks a disallowed request written in a regional language too.
