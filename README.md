# Smoking history benchmark for lung cancer screening decision support

A benchmark of 3,000 synthetic outpatient progress notes with a known answer for each note: smoking status, pack-years, and quit date. It was built to measure how well language models can recover the smoking information that lung cancer screening (LCS) clinical decision support depends on, and it can be used to compare any extraction method on the same notes.

## These notes are synthetic

**Every note in this repository was generated. None of it comes from a real patient, a real clinician, or a real health system.**

- The notes were written from scratch for this benchmark. They are not copies, excerpts, paraphrases, or de-identified versions of real clinical notes.
- No real clinical text was used to produce them, and no existing clinical corpus was used as a source.
- Every name, date of birth, medical record number, date of service, address, and clinician name is invented. Any resemblance to a real person is coincidental.
- The notes contain no protected health information, so they can be shared, posted, and sent to commercial model APIs without a data use agreement.

Because the notes are synthetic, the answer key states exactly what each note says, rather than what a chart reviewer guessed from it.

## What is here

| Condition | Folder | Notes | How the notes were written | Median words |
|---|---|---|---|---|
| Basic | `basic/` | 1,000 (`PT0001`–`PT1000`) | A template program, in six note formats | 242 |
| Messy | `messy/` | 1,000 (`MS0001`–`MS1000`) | Written by a large language model from a specification of facts | 567 |
| Complex | `complex/` | 1,000 (`MC0001`–`MC1000`) | The same, plus histories that need multi-step arithmetic or date reasoning | 559 |

Each folder holds `notes/`, with one plain-text note per patient, and `answers.csv`, the answer key.

The three conditions separate two different difficulties. The messy condition tests whether a model can find the smoking history in long, realistic, cluttered documentation. The complex condition tests whether it can compute the answer once it has found it, for example from two or three smoking periods at different rates, or from a quit date given as "three years after her husband died in 2010".

## How the benchmark was built

The answer came first. For each patient, a program generated demographics and a smoking history, decided how the note would document that history, and computed the reference labels from that decision. Only then was a note written to convey it. This means the labels reflect what the note actually says, not latent facts the note never states.

**Basic condition.** A program rendered each note in one of six formats: an EHR-style note with discrete tobacco fields, a SOAP note, a terse note, a dictated narrative or consult letter, an assessment-first note, and a brief telehealth note. Each note has a chief complaint and history for one of 18 visit types, comorbidities, medications, vital signs, an examination, and an assessment and plan. Smoking information may appear in the history, the social history, a templated block, the problem list, or the assessment and plan, and is sometimes split between sections. Terse and assessment-first notes use heavy abbreviation, and about 9% of notes contain one or two misspellings.

**Messy and complex conditions.** For each patient, code generated a specification: demographics, visit type, note type, known conditions, a target length of 450 to 1,500 words, three to five sources of "messiness", and a list of facts that had to appear along with distractors that had to appear but must not change the answer. Claude Sonnet 5 then wrote the note from that specification. The messiness includes text copied forward from an older visit, a templated social history with blank fields, long boilerplate, speech-recognition errors in non-smoking text, heavy abbreviation, an outdated problem list with ICD-10 codes, an addendum, direct patient quotes, and smoking information split across sections.

Every generated note was then checked in code, with no model involved: each required fact had to appear in a verbatim quote from the note, the quoted text had to contain the fact's values (accepting number words, fractions such as "half a pack", and cigarette equivalents of pack rates), the note could not state a pack-year figure unless the specification gave one, and it could not comment on screening eligibility. Notes that failed were sent back with the specific problems, up to three attempts; specifications that still failed were replaced.

**Distractors.** Many notes contain material designed to mislead a naive extractor: a relapse after an earlier quit, a future quit date, a date for quitting alcohol rather than tobacco, a family member who smokes, secondhand smoke, e-cigarettes, cannabis, cigars or smokeless tobacco, a patient who "tried a few cigarettes" as a teenager, a stale "never smoker" template field that the narrative corrects, and copied-forward text from an earlier visit.

**Checks on the answer key.** An LLM-based reviewer (Claude Opus 5) blinded to the labels abstracted 60 of the notes independently and agreed with the answer key on all three variables for every note. After an evaluation run, every note on which any system disagreed with the answer key was adjudicated, and automated consistency checks were run over all 3,000 notes. This found several systematic errors in the generators, which were corrected in code and the labels re-derived without changing any note. Those corrections affected 84 of the 3,000 notes.

## The answer key

`answers.csv` in each folder has one row per note.

| Column | Meaning |
|---|---|
| `patient_id` | Matches the file name in `notes/` |
| `age`, `sex`, `note_date` | Demographics and visit date, as an EHR would hold them |
| `smoking_status` | `current`, `former`, `never`, or `unknown` |
| `pack_years` | Packs per day multiplied by years smoked; empty when the note does not support a value |
| `pack_years_min`, `pack_years_max` | The accepted range (see below) |
| `quit_date` | `YYYY`, `YYYY-MM`, or `YYYY-MM-DD`, at the precision the note supports; empty for anyone who is not a former smoker with a documented quit time |
| `quit_date_precision` | `year`, `month`, or `day` |
| `quit_year_min`, `quit_year_max` | The accepted range of quit years |
| `uspstf2021`, `acs2023` | The screening decision implied by the note (see below) |
| `pack_years_source`, `quit_date_source` | How the note documents each value, for error analysis by documentation pattern |
| `note_format`, `visit_type`, `challenge_tags` | The kind of note, the reason for the visit, and which distractors it contains |

**Definitions.** Status refers to cigarette smoking as of the visit date. Patients who are cutting down, have set a future quit date, or have relapsed after an earlier quit are current smokers; patients who have quit, however recently, are former smokers, even if they now use e-cigarettes or nicotine replacement; patients who smoked fewer than 100 cigarettes are never smokers; status is unknown when the note does not establish it. Cigars, pipes, cannabis, e-cigarettes, smokeless tobacco, secondhand smoke, and family members' smoking do not count. Pack-years are packs per day times years smoked, with one pack being 20 cigarettes; they are taken from a stated value when the note gives one, computed when the note gives enough information (summing periods at different rates, using start and stop ages or years, excluding stated breaks, taking the midpoint of a stated range), 0 for never smokers, and empty when the note gives only a rate, only a duration, or only a vague description.

**Ranges.** Some notes are inherently imprecise. "1 ppd since age 17" in a 60-year-old implies about 43 years of smoking, depending on the patient's birthday, so 42 to 44 pack-years are accepted; "quit 10 years ago" accepts a quit year within a year of the visit year minus 10. Ranges are exact for stated values.

**Suggested scoring.** Status is correct if it matches exactly. Pack-years are correct if the value falls within `[pack_years_min, pack_years_max]` (a tolerance of 0.5 for rounding is reasonable), or is null when the reference is empty. A quit date is correct if its year falls within `[quit_year_min, quit_year_max]`, or is null when the reference is empty.

**Screening decisions.** `uspstf2021` and `acs2023` give the decision that follows from the note under a simplified eligibility framework based only on age, pack-years, and, for former smokers, years since quitting. Under USPSTF 2021, a patient is eligible if they are 50 to 80 years old, have at least 20 pack-years, and currently smoke or quit 15 or fewer years before the visit; the ACS 2023 guideline drops the years-since-quitting criterion. Every patient in the benchmark is 50 to 80 years old. The value is `eligible`, `not_eligible`, or `insufficient` when the note does not support a decision (for example, a former smoker with at least 20 pack-years and no documented quit date). A few notes have a reference range that straddles a threshold, such as "15 to 25 pack-years"; for those, two decisions are accepted and both are listed, separated by `|`. The framework is deliberately simple and does not consider comorbidities, life expectancy, prior screening, or shared decision-making.

## What this benchmark does not tell you

The notes are realistic, but they are not real. In particular, notes written by a language model tend to spell facts out in full sentences, where real notes often compress them into shorthand such as "1-2 ppd x53y". Real documentation is also messier in ways that are hard to synthesize, including scanned text, templates from many source systems, and years of accumulated copy-forward. A method that does well here should still be validated on local notes before it is used in care.

## Citation

Wright A, Liu S, Wright A. Extracting smoking history from clinical notes for lung cancer screening decision support: comparing a structured-judgment model with general-purpose large language models. Preprint, 2026.

Code for running models over this benchmark and scoring the results is at [vclic/smokingeval](https://github.com/vclic/smokingeval), along with the predictions from every run in the study.

## License

The notes and answer keys are released under the Creative Commons Attribution 4.0 International license (CC BY 4.0). See [LICENSE](LICENSE).
