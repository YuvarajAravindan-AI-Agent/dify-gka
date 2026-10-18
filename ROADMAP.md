# Governed Knowledge Assistant → Production Governance Platform
## Roadmap for IIT Madras WSAI AI Venture Studio (Cohort opening 3 Oct 2026)

**Positioning:** not "I built a chatbot," but "I built and proved a governance-enforcement
pattern that CeRAI's own research themes — Making AI Understandable, Safe AI — need a
deployable reference implementation of." Ravindran's RL background rewards *safety by
construction*, not by instruction. CeRAI's own outputs (PolicyBot, IndiCASA, AI Incident
Reporting Framework) are concrete artifacts, not white papers — match that register.

---

## 0. The one-sentence pitch

> Most "AI governance" is a prompt asking the model to behave. This is a platform where
> ineligible data is structurally removed before the model ever sees it — proven live,
> now being generalized into deployable infrastructure for regulated and public-sector AI
> deployments in India.

---

## 1. Phase 0 — Before 3 Oct (application-ready proof)

Goal: the existing Dify prototype is demo-solid, not "mostly working."

- [ ] Fix the outstanding Knowledge Retrieval defect (§5.2 of HANDOFF.md) — check dataset-level
      Score Threshold setting first, per the handoff's own diagnosis.
- [ ] Run and screenshot/record all 7 acceptance tests in §8 of HANDOFF.md passing.
- [ ] Confirm AI credits still read 1/200 used (proves the pipeline is honest about model routing).
- [ ] Write the reference-architecture positioning piece (drafted separately) as the
      application's core narrative artifact — pair it with the live demo link.
- [ ] Re-read CeRAI's "AI Incident Reporting Framework for India" and PolicyBot outputs
      (cerai.iitm.ac.in) closely enough to name them specifically in the application —
      show you did the homework on what they've already built, not just who runs it.

**Deliverable:** one working demo + one 1-page architecture note, not a platform.

---

## 2. Phase 1 — Months 1–2 of the program: generalize the enforcement layer

The current build hard-codes two policy dimensions (status, region) for one corpus.
A platform needs the *filter itself* to be data, not code.

- [ ] **Policy-as-config**: move eligibility rules (status/region/effective-date/whatever-comes-next)
      out of hardcoded node conditions into a declarative rule set per deployment — a small
      JSON/YAML schema, evaluated the same way regardless of domain.
- [ ] **Domain-agnostic proof**: re-run the same engine against a second, unrelated corpus
      (e.g. a public-sector circular dataset or a healthcare formulary) to prove the pattern
      isn't HR-policy-specific. This is the single most important proof point for
      "platform" vs. "app" — do it early.
- [ ] **Effective-dating as a first-class rule**, not prompt instruction (HR-POL-021 future-dated
      case) — generalize the "state the date, don't apply it yet" behavior into the rule engine
      itself, so it doesn't depend on the LLM remembering to check.
- [ ] Move off Dify's UI-only node config where it blocks programmatic policy authoring —
      evaluate whether the retrieval-filtering logic needs to move to a thin service layer
      Dify calls out to, so policies can be authored/versioned outside the no-code UI.

**Deliverable:** the same enforcement engine running over 2+ unrelated corpora with different
rule sets, controlled by config, not code changes.

---

## 3. Phase 2 — Months 3–4: auditability and accountability

This is where the platform explicitly answers to CeRAI's "accountable and explainable" mission
language, and to their AI Incident Reporting Framework work specifically.

- [ ] **Decision logging**: every answer logs which documents were candidates, which were
      excluded and why (status/region/date rule that filtered them out), and which were used —
      not just the final answer. This is the audit trail regulators and incident-reporting
      frameworks actually need.
- [ ] **Citation verification**: automated check that every claim in an answer traces to a
      passage that actually passed the eligibility filter (catch the model quietly using
      excluded context it shouldn't have — a known failure mode of naive RAG).
- [ ] **Refusal correctness metric**: track false-refusal and false-answer rates against a
      held-out test set (you already have the scaffolding — extend §8's acceptance tests
      into a proper eval set).
- [ ] Map this logging schema explicitly to what an "AI Incident Report" for a deployed
      system in India would need to contain — write this up as a document CeRAI could
      plausibly cite, not just internal engineering notes.

**Deliverable:** an audit log format + eval harness that could be handed to a compliance
or incident-review team, not just a chat transcript.

---

## 4. Phase 3 — Months 5–6: deployment shape and handoff

- [ ] **Multi-tenant**: same engine, isolated rule sets and corpora per deploying organization
      (the realistic shape of "platform," since no single org needs bespoke infra).
- [ ] **Reference deployment**: one real or realistic pilot — ideally something CeRAI's existing
      partners (NASSCOM, Vidhi Legal, a gig-economy or public-sector angle matching their
      published work) would recognize as relevant, not a synthetic HR corpus.
- [ ] **Open publication**: write the pattern up formally enough that it could sit alongside
      CeRAI's own outputs (PolicyBot, IndiCASA) — position this as a contribution to their
      body of work, not a startup pitch deck.
- [ ] Decide the actual venture shape coming out of the studio: open infrastructure/reference
      implementation (credibility play, aligns with CeRAI's "standard body" ambition) vs.
      commercial platform (revenue play) — these pull in different directions; make the call
      explicitly rather than drifting.

**Deliverable:** a decision point, with evidence, on what this becomes after the studio —
not just "still building."

---

## 5. What NOT to do

- Don't rebuild this on unfamiliar infra to look more "production" (e.g. abandoning Dify for
  a from-scratch stack) unless Phase 1's programmatic-policy-authoring need actually forces it.
  Momentum and a working demo beat architectural purity this early.
- Don't chase "state of the art" model performance — the differentiator is the *enforcement
  boundary*, not retrieval quality. Keep using whatever model is cheapest/credit-safe.
- Don't scope Phase 2/3 work before Phase 0 is airtight. The application is judged on what
  exists today plus a credible plan — not on the plan alone.

---

## 6. Immediate next actions (this week)

1. Fix the Knowledge Retrieval defect.
2. Re-run and record all 7 acceptance tests.
3. Finalize the reference-architecture write-up.
4. Read CeRAI's PolicyBot / IndiCASA / Incident Reporting Framework outputs closely enough
   to reference them by name and substance in the application.
