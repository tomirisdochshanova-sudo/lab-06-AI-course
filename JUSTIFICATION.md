# JUSTIFICATION — the design of the harness, and why

## 1. What the system may do

The fictional Tulpar Scholarship Fund uses the Qwen2.5-3B-Instruct model to *screen* scholarship applications. It may suggest `shortlist` or `reject` and a score from 0 to 100 for a valid, consenting adult applicant. I would not use this prototype to award or deny a real scholarship without an accountable review process. `escalate` means that a human should examine the case; it is not a final approval.

I built five controls around the supplied model, rather than changing its weights. In `Harness`, `validate_input=True` checks numeric types and ranges. `blind=("name", "university")` removes these two fields from the model prompt. `refuse=True`, together with `my_extra_refusal`, stops certain requests before inference. `log_path="decisions.jsonl"` records each call. `escalate=True`, the `borderline=(40, 50)` band and `my_extra_escalation` route uncertain replies and some high-merit rejections to review. The extra rule checks a rejection when GPA is at least 3.5 and the exam score at least 110. These are my design thresholds, not official scholarship rules.

In Part 1, name language produced 2/40 flips, gender-coded name 4/40, and university 1/40. None clearly stood above its own noise floor. I therefore cannot claim that the model demonstrated systematic discrimination. I still removed the name and university because they are not needed to assess the merit criteria used in this exercise and can carry irrelevant signals. This changes the architecture: the original agent sees the complete profile, but the controlled agent sees a reduced profile. The application identifier remains in the log, not in the prompt.

## 2. What it must refuse

The refusal controls are applied inside `screen()` before a model call. `validate_profile()` refuses invalid numeric or consent fields with `invalid_input`. Built-in `refusal_reason()` refuses a purpose other than `scholarship_screening` (`purpose_not_allowed`), applicants younger than 18 (`minor`), and applications without recorded consent (`no_consent`). My `my_extra_refusal()` checks text fields for listed prompt-injection phrases and can return `prompt_injection_detected`. This last control is only a simple phrase filter, not a reliable detector for every possible instruction.

In the last confirmed Part 4 output, the bare agent refused 0/6 test requests. The harness refused **4/6**: wrong purpose, age 17, no consent and impossible values. Each of those four refusals used zero model calls. The ordinary application was processed. **The prompt-injection test was not blocked in that observed run**: the suspicious name was processed and shortlisted. This is a real failure of my first phrase list, because the test phrase used a wording not included in it. An added phrase would need to be tested and the final log regenerated before I could claim this control was enforced. The 400 clean matched-pair applicants produced 0 false refusals in that check, but they are not an exhaustive test of legitimate text.

These refusals also have costs. A student with a typing mistake is stopped instead of silently scored; the fund needs a correction path. An applicant below 18 is excluded by this *prototype's rule*, even though a real scholarship may legitimately allow minors with an appropriate lawful process. A missing consent flag can be a recording error, and a phrase such as “system prompt” could appear in harmless text. The notebook records the refusal and its reason, but it does not implement applicant notices, a correction portal, an appeal, or automatic human review of refused applications. The real fund must provide these procedures before use. I would test false positives using reviewed legitimate applications, especially unusual names and free-text fields.

## 3. What record it keeps

**Audit question:** Why did this application receive this outcome, which controls acted, and what did the model return at the time?

The submitted `decisions.jsonl` uses one JSON object per `screen()` call. Each object includes `record_id`, `pid`, `input_sha256`, field names shown and withheld, model ID, prompt version, harness configuration, raw model output, parsed decision and score, controls fired, final outcome, reason code, human-escalation flag and UTC timestamp. A reviewer can distinguish a model decision (`model_decision`), a borderline referral (`borderline`), and a refusal (`purpose_not_allowed`, for example) without running the model again. The sample escalation record stores a model rejection with score 40 but final outcome `escalate`: these are deliberately different things.

The last displayed log contained **406 records**: 340 `shortlist`, 24 `reject`, 38 `escalate` and 4 `refuse`. It includes the 400 measured applicants and six Part 4 requests. The log keeps field *names*, not a full duplicate of the application. Names and universities are withheld from model prompts, but the raw model reply can still contain other values or information. A hash helps detect differences when checked against a trusted input; it is not a substitute for access control or tamper-proof storage.

The notebook does **not** enforce retention, access permissions, encryption, immutable storage or independent review signatures. For a real system I would propose a documented retention period based on the actual legal and operational need, role-based read access for trained reviewers and auditors, restricted write permissions, encrypted storage, backups, and an append-only audit store with integrity checks. I would not invent an exact statutory retention period for this prototype. The present record cannot by itself prove a reviewer acted on an escalation, show an applicant's complete original profile, or reconstruct hidden model internals.

## 4. The legal constraints

This laboratory contains only fictional profiles and institutions. The following is a **Kazakhstan deployment scenario**, not a claim that these synthetic applicants create the same legal obligations as real data subjects. Whether an actual fund is an operator, which data it processes, and whether any legal exceptions apply must be checked before deployment.

**Personal data law.** The Republic of Kazakhstan Law No. 94-V of 21 May 2013, *On Personal Data and their Protection*, Article 7(1), sets consent as a general basis for collection and processing with statutory exceptions. Article 22 addresses protection of personal data. These provisions are relevant if a real scholarship fund processes identifiable applicant data. They support checking a lawful processing basis and protecting stored records. They do **not** prescribe my exact refusal codes, keyword list, blinding fields or JSONL schema. Primary text: https://adilet.zan.kz/rus/docs/Z1300000094 (Articles 7 and 22).

**AI law.** The Republic of Kazakhstan Law No. 230-VIII of 17 November 2025, *On Artificial Intelligence*, Article 7 (Transparency and Explainability), addresses information about AI-supported decisions and the possibility of objecting to automated processing. This is especially relevant to a real scholarship screening service that affects people; the current notebook has technical logs but no user-facing explanation or objection interface, so it is not sufficient by itself for real deployment. Primary text: https://adilet.zan.kz/rus/docs/Z2500000230 (Article 7).

I treat the law's actual requirements separately from my additional safeguards. Blinding `name` and `university`, screening for prompt injection, and using the 40–50 review band are engineering choices, not methods specifically mandated by the cited articles. The legal and operational assessment of fully automated scholarship denials, notices, consent exceptions, and appeal arrangements requires further review. The fact that the notebook produces a JSON log does not prove legal compliance.

## 5. The ethical principles

I use the **OECD Recommendation of the Council on Artificial Intelligence**, OECD/LEGAL/0449 (adopted 2019, amended 2024), as an ethical framework. This is an OECD recommendation, **not a Kazakhstan statute**. Official text: https://legalinstruments.oecd.org/en/instruments/OECD-LEGAL-0449 .

- **Human-centred values and fairness, including privacy (Section 1.2):** I hide names and universities to avoid unnecessary identity and institution signals. This is a precaution after the measured 2, 4 and 1 flips, not evidence that an unfair effect was proved.
- **Transparency and explainability (Section 1.3):** `reason_code`, raw output and escalation status help explain what happened; I still need a way to explain decisions to applicants.
- **Robustness, security and safety (Section 1.4):** input validation and refusal rules aim to stop invalid or unsafe processing. The missed prompt-injection example shows their limits.
- **Accountability (Section 1.5):** logging enables later examination of model output and harness actions, although the current file can still be altered.

Principles can conflict. Hiding the university improves consistency for matched applicants but removes potential context about different grading systems. Human review can catch difficult cases, but it costs staff time and can itself introduce human bias. I prioritised avoiding irrelevant model inputs and documenting exceptions, while recognising that the choice needs evaluation on actual applicants before deployment. Ethics also motivates checking the harm of an erroneous rejection even where a cited law does not provide a specific Python implementation rule.

## 6. The trade-offs

The controls changed the processing results, but they did not establish that the original model was systematically biased. After blinding, all five matched-pair categories had **0/40 flips**. This is **zero by construction**: two members of each pair now have identical visible input. It is not proof that model weights became fairer.

Across individual applicants, 50/400 outcomes changed: 22 `reject` to `escalate`, 16 `shortlist` to `escalate`, 7 `shortlist` to `reject`, and 5 `reject` to `shortlist`. Scores changed for 166/400 applicants, with a mean absolute change of 14.01 points among those changed. Removing lines also changes prompt shape, so these changes do not prove reliance on protected fields.

Input validation and refusal prevent some bad requests at the cost of possible false refusals and additional correction work. Blinding may remove useful context. The log adds storage and privacy risks. Human escalation sends uncertain outcomes to reviewers but does not guarantee they will be corrected. The baseline 40–50 band included 109/400 scores (27.3% planned review workload); the protected run actually escalated **38/400 (9.5%)**. I did not recalibrate the band after changing the fields shown, so the lower actual share is a limitation and needs correction before deployment. Cases outside the review band may still receive incorrect automated outcomes, and my extra rule only catches one category of apparently surprising rejection.

The bare run used 400 model calls, 84,409 prompt tokens, 11,122 new tokens and 482.8 seconds. The harness run used the same 400 calls, 77,922 prompt tokens, 11,088 new tokens and 466.2 seconds. No extra calls were required because `counterfactual_check=False`; this saves compute but does not directly test counterfactual consistency for each application. The measured time difference is specific to these runs and is not a guaranteed speed gain.

Before real use I would recalibrate the band on the protected system, test more representative and multilingual inputs, check false refusals and adverse impacts, enforce secure retention and append-only logs, create a clear appeals and human-review workflow, and retest the prompt-injection case. I would stop or restrict deployment if unsafe refusals, unreviewed consequential denials or unexplained group differences appeared. This prototype has no empirical basis for claims of full fairness or legal compliance.

## AI-use declaration

I used **ChatGPT (OpenAI)** as an assistant during this assignment. It helped me understand the English instructions in Russian, explain the matched-pair and noise-floor results, and draft and edit English-language text for the notebook and reports. It also suggested examples of refusal and human-escalation rules, and helped identify a mismatch between my prompt-injection phrase list and the test request. I used its assistance to organise this justification and identify legal instruments and ethical principles for examination.

The model, synthetic applicants, measurements and numerical outputs came from the supplied Google Colab notebook, not from ChatGPT. I did not ask ChatGPT to invent experimental outcomes. The reported figures are the results shown by my notebook; the remaining prompt-injection failure is disclosed rather than presented as a successful test. Legal statements have primary-source links to the relevant Kazakhstan instruments and provisions, and OECD principles are identified separately as recommendations rather than laws. AI suggestions are not independent legal evidence and must be checked against those texts.

I am responsible for the final choices, code, measurements, interpretations and submitted files. I understand that using AI to help draft a report does not transfer responsibility for errors, unsupported claims or missing tests. This declaration describes the assistance used; it does not claim that the assistant itself executed the submitted Colab run.
