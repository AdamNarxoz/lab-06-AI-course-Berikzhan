# JUSTIFICATION — the design of the harness, and why

## 1. What the system may do

The system's only purpose is **pre-screening scholarship applications for the (fictional) Tulpar Scholarship Fund**:
it may *recommend* shortlist or reject. Only `shortlist` and clear `reject` outcomes outside the escalation rules
are produced automatically; borderline, contradictory and lowest-income rejections may only *inform* a human reviewer.

Controls built (all in Part 2 of the notebook, `HARNESS = Harness(validate_input=True, blind=("name","university","age"),
refuse=True, log_path="decisions.jsonl", escalate=True, borderline=(36, 46))`):

| control | constraint that made me build it | where in the code |
|---|---|---|
| input validation | robustness (AI Act Art. 15 as benchmark): the bare agent turned GPA 9.9 / exam −5 into a reject | `validate_input=True` → `validate_profile` |
| blinding | non-discrimination (Constitution of RK Art. 14(2)); +2.5 [+1.1, +4.1] flagship score shift; 0/4 gender pattern | `BLIND`, Part 2a |
| refusal | consent (Law No. 94-V Art. 7), purpose limitation, prompt-injection risk in free-text fields | `refuse=True` + `my_extra_refusal`, Part 2b |
| decision log | accountability (AI Act Art. 12, GDPR Art. 5(2) as benchmarks) | `log_path`, `decisions.jsonl` |
| escalation | human oversight (GDPR Art. 22(3), AI Act Art. 14 as benchmarks) | `escalate=True`, `BORDERLINE`, `my_extra_escalation`, Part 2c–2d |

**A constraint that changed the architecture:** without the non-discrimination constraint I would have shown the
model every field, including the university, as a proxy for grading strictness. Because of it, name, university and
age are removed *before* the model call, and age is used only by the refusal rule that runs before the model. The
consent constraint also changed the order: refusal checks run before any model call, so no personal data of a
refused applicant is sent to the model (0 model calls for each of the 5 refusals in Part 4).

## 2. What it must refuse

| refusal | enforced in | fired in my run (Part 4) | legitimate case it also blocks |
|---|---|---|---|
| purpose other than scholarship screening | `refusal_reason` (`purpose_not_allowed`) | "rank applicants by ethnicity" → refuse | a legitimate audit run under a different purpose label; cost to the fund's staff, acceptable |
| applicant under 18 | `refusal_reason` (`minor`) | age 17 → refuse | a gifted 17-year-old who legitimately applies; must go to a human |
| no consent to automated processing | `refusal_reason` (`no_consent`) | → refuse | an applicant who prefers human review; they are not denied, they must be processed by a person |
| impossible values | `validate_profile` (`invalid_input`) | GPA 9.9 / exam −5 → refuse | a typo by an honest applicant; must be correctable |
| name or free text that is not plain text / contains an instruction | `my_extra_refusal` (`name_field_not_a_name`, `instruction_in_free_text`) | injected name → refuse | real names in Cyrillic or with diacritics (e.g. Нұрай), or a programme named with an unusual character |

The bare pipeline refused 0 of these 6 requests; my harness refused 5 of 6, and **0 of the 400 clean applicants**.
The heaviest cost falls on applicants with non-Latin names, which my rule would refuse; this is acceptable *only*
because every refusal is logged with its reason code and must be routed to a human. Every refused request produces
one record in `decisions.jsonl` (`outcome: "refuse"`, `reason_code`, `raw_output: "[no model call] …"`). What is not
yet built: an automatic notice to the applicant and a form to ask for human processing — that is the fund's job and a
gap listed in section 6.

## 3. What record it keeps

**Audit question:** *"Why did this applicant get this outcome, and can someone reconstruct it later without re-running
the model?"* Each line of `decisions.jsonl` answers it: `record_id`, `pid` (application id), `input_sha256` (hash of
the exact text the model saw), `fields_shown` and `fields_withheld`, `model_id`, `prompt_version`, the full `harness`
settings, `raw_output`, `decision`, `score`, `controls_fired`, `outcome`, `reason_code`, `escalated_to_human`, `timestamp`.

**Left out on purpose:** field values (name, university, grades) are not stored, only field names and a hash; the
person is found through `pid` in the fund's own application system. This follows data minimisation. Note that
`raw_output` may still quote values the model was shown (never a blinded field).

**Retention and access (proposed, not enforced by code):** kept for the scholarship cycle plus the appeal period;
readable by the fund's reviewers and auditor; append-only. Nothing in the code stops a record being edited after the
fact — a hash chain or write-once storage would be needed. The record does not let you reconstruct *why the model*
chose a score, only what it was shown and what it answered.

## 4. The legal constraints

The fund is fictional. I assume it operates in **Kazakhstan** and processes the personal data of applicants there,
so Kazakh law applies; EU law is used **only as a benchmark** for good practice, since the fund is not established
in the EU and does not target EU residents.

- **Constitution of the Republic of Kazakhstan (30 August 1995), Article 14(2):** prohibits discrimination on grounds
  of origin, sex, race, nationality, language and other circumstances. *Requires* that the screening not treat
  applicants differently on these grounds → blinding of name (language, sex) and monitoring with matched pairs.
- **Law of the RK No. 94-V "On Personal Data and Their Protection" (21 May 2013), Article 7:** collection and
  processing of personal data with the subject's consent → `no_consent` refusal and minimised log.
- **Regulation (EU) 2016/679 (GDPR), Article 22(1)–(3)** (benchmark): right not to be subject to a decision based
  solely on automated processing, and the right to obtain human intervention → escalation; **Article 5(2)**
  (accountability) → log.
- **Regulation (EU) 2024/1689 (AI Act)** (benchmark): AI systems used to determine access to education are listed as
  high-risk in **Annex III point 3(a)**; **Article 12(1)** record-keeping → log; **Article 14** human oversight →
  escalation; **Article 15** accuracy and robustness → input validation and the injection refusal.

**Required vs permitted:** consent and non-discrimination are requirements; the band width, which fields to blind and
what to log are left to me. I went beyond the Kazakh texts I cite by adding human review and a full decision log.
I did not find a Kazakh provision written specifically about automated decisions in the personal-data law; I do not
claim one.

## 5. The ethical principles

- **Fairness and non-discrimination** — UNESCO Recommendation on the Ethics of AI (2021), Section III.2
  (non-binding recommendation). Bears on blinding name and university.
- **Human oversight and determination** — same source. Bears on escalating borderline (36–46), merit-contradicting
  and lowest-income rejections: 40 of 400 (10.0%).
- **Proportionality and do no harm; right to privacy** — same source. Bears on refusing the ethnicity-ranking purpose
  and on not sending refused applicants' data to the model.
- **Robustness, security and safety; transparency; accountability** — OECD Recommendation on AI,
  OECD/LEGAL/0449 (2019, amended 2024), principles 1.4, 1.3, 1.5 (non-binding). Bear on input validation, the
  injection refusal and the log.

**Conflict:** identical treatment (hide everything) vs fair treatment given context (a regional university may grade
differently). I put identical treatment first because I measured a flagship score advantage and no evidence that the
university carried merit information. A second conflict: refusing non-Latin names protects against injection but
treats some honest applicants worse; I accepted it only with a human fallback.

**Where ethics demands more than law:** the law does not require me to measure matched pairs or to report that the
agent's scores moved by 14.92 on average when three lines were removed; ethics (transparency, do no harm) does.

## 6. The trade-offs

- **Blinding:** costs the model university information and changed 49 of 400 outcomes and 188 scores; prevents the
  measured +2.5 flagship shift and any use of name-based signals. Prompt tokens fell (75522 vs 84409).
- **Refusal:** costs possible wrong refusals of non-Latin names; prevents processing minors, non-consenting
  applicants, forbidden purposes and injected instructions (5 of 5 refused, 0 model calls, 0 of 400 clean refused).
- **Escalation:** the band (36, 46) was chosen from the bare scores as ±5 around the bare agent's decision boundary;
  it was planned to send 64 of 400 (16.0%) but actually sent 40 of 400 (10.0%), because it was calibrated on unblinded
  scores. I did **not** recalibrate it on the blinded scores; that should be done before deployment. Cases outside
  the band — 340 shortlists and 20 rejects — are re-checked by nobody.
- **Log:** costs storage and governance; prevents unaccountable decisions.
- **Cost in model calls:** 400 in both runs (1.00 per profile); 501.3 s vs 475.3 s.

**What would change my balance:** if a larger run showed a decision effect above its floor, I would add the
counterfactual check (doubling model calls) or widen the band.

**What the design still fails to protect against:** proxy discrimination through other fields (programme, income
bracket); prompt-wording sensitivity (scores moved 14.92 on average); a tamper-proof log; notifying refused
applicants and an appeal route; testing with more than 40 pairs per group. Closing these needs more runs, storage
infrastructure and the fund's own processes, which were out of scope for this lab's time.

