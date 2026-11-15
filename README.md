# DubPatch

**Evidence-first review of AI-dubbed Shorts.** DubPatch checks a Sarvam dub of an English automotive Short (for example into Telugu) against its source and returns one of three verdicts: `preserved`, `review_required` or `unable_to_verify`. It gives no accuracy score, and it never certifies a dub without a human reviewer.

<!-- Demo: drop the openscreen recording here (GIF or MP4, 20–30 s). -->

![DubPatch overview](https://github.com/user-attachments/assets/bf26c355-88b8-4973-b3d1-5d557bfc400c)

## Why

An automotive Short lives or dies on its facts: *1300 hp, not 1000 hp*, *rear-wheel drive*, *twin-turbo*. An AI dub can sound fluent and still change those facts, and a single fluency or similarity score hides that. DubPatch treats each fact as evidence to check, and says "can't tell" when the evidence is missing.

## What it catches

A synthetic demo fixture in which the dub turns **1300 hp** into **1030 hp**:

```sh
python3 -m shorts_fidelity_judge.cli evaluate benchmark/fixtures/demo_drift/manifest.json --output out/demo
```

```json
{
  "category": "number_or_unit_mismatch",
  "severity": "critical",
  "source_text": "Now it makes 1300 hp, not 1000 hp.",
  "target_text": "Now it makes 1030 hp, not 1000 hp.",
  "evidence": "Source numbers: ['1300', '1000']; target numbers: ['1030', '1000'].",
  "uncertainty": "Lexical alignment and supplied back-translation may not capture an acceptable idiomatic adaptation.",
  "recommended_action": "Human reviewer should compare the corresponding audio and transcript.",
  "deterministic": true
}
```

The demo has no audio, so the overall verdict is `unable_to_verify`; the finding is reported anyway. Every finding carries its evidence, an uncertainty statement and a recommended action. The run also writes `report.html` for a reviewer.

## Verdicts

| verdict | meaning |
|:--|:--|
| `preserved` | Every checked fact in the source survives in the dub, with the evidence needed to say so. |
| `review_required` | Audio was inspected and at least one finding was raised for a human to decide. |
| `unable_to_verify` | No dubbed audio was supplied, so no overall claim is made; findings from the transcripts are still reported. |

## Design rules

- **Deterministic checks win.** Numbers, units, currency, model codes, drivetrains, protected terms and named entities are checked by code against a [glossary](benchmark/glossary.yaml). An LLM judge can add evidence or uncertainty, but it cannot downgrade one of these mismatches to `preserved`.
- **Missing evidence is uncertainty, never a guess.** Without dubbed audio the verdict is `unable_to_verify`; nothing missing is inferred.
- **One network boundary.** Only the Sarvam STT adapter calls out (`SARVAM_API_KEY`, with a timeout and bounded retry). The raw response is cached by the SHA-256 of the audio, so a re-run never uploads the audio again.
- **Offline by default.** With a supplied target transcript, the whole pipeline and the test suite run without a network.

## Pipeline

```text
manifest + English transcript + local WAV
        │                         │
        │                    audio inspection (peak, RMS, loudness, pauses)
        ▼                         ▼
manual target transcript  OR  Sarvam STT (content-addressed cache)
        │
source / target segmentation and alignment
        │
deterministic protected-atom checks  ◄── glossary
        │
semantic judge (optional, strict Pydantic JSON)
        │
report.json + report.html + annotations.jsonl
```

Diagrams: [system overview](docs/architecture/system-overview.svg) · [critical path](docs/architecture/critical-path.svg) · [evidence notes](docs/architecture/README.md) · [architecture](docs/ARCHITECTURE.md)

## Quick start

Python 3.11+.

```sh
pip install -e .
python3 -m pytest -q          # 37 tests, no network
python3 -m shorts_fidelity_judge.cli evaluate benchmark/fixtures/demo_drift/manifest.json --output out/demo
```

For a real Short, see [tests/ytshorts/README.md](tests/ytshorts/README.md): export the dub WAV from Sarvam, write a `case.json`, then run `dubpatch-ingest tests/ytshorts/<case-id>/case.json`. Live STT needs `SARVAM_API_KEY` in `.env` (see `.env.example`).

Other entry points: `dubpatch-review` (reviewer UI), `dubpatch-api` (FastAPI backend) and `frontend/` (a Vite + React prototype whose processing flow is still mocked).

## Reviewer metrics

Reviewer decisions go to `annotations.jsonl`. Two numbers are tracked:

- **precision**: true issues divided by the issues a reviewer could judge;
- **review compression**: issues a reviewer handled divided by issues raised automatically.

No benchmark results are published yet: the bundled `mat_armstrong_*` and `demo_drift` files are synthetic test inputs, not transcriptions of real videos or real dubs ([fixture notice](benchmark/fixtures/README.md)).

## Limitations

- Alignment is lexical, so an acceptable idiomatic rewording can be flagged, and a reordering of the same terms (for example swapped drivetrains) can be missed.
- Selective repair of flagged segments is planned; the current release reviews, it does not patch.
- Only automotive Shorts are covered by the glossary today.
