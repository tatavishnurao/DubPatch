# Architecture diagrams

## What the system does
DubPatch is an evidence-first review system for Sarvam-dubbed automotive Shorts. It emits preserved, review_required, or unable_to_verify and keeps evidence and uncertainty in its reports.

## Problem addressed
A single accuracy score can hide missing transcripts, alignment uncertainty, protected-atom changes, and places requiring human judgement. DubPatch keeps those boundaries explicit.

## Overview
[System overview](system-overview.svg) shows source/target media and transcripts feeding alignment, deterministic checks, optional semantic evidence, and explicit report states.

## Critical path
[Critical path](critical-path.svg) follows a claim from segment evidence through confidence/verification boundaries to a decision state. It does not depict autonomous certification or a universal score.

## Evidence used
README.md; docs/ARCHITECTURE.md; backend/app.py; backend/services/job_service.py, semantic_judge.py, dubbing.py; shorts_fidelity_judge/ingest.py, alignment.py, review.py, report.py, stt.py, transcripts.py; tests/test_pipeline.py, test_review.py, test_judge.py, test_semantic_judge.py, test_job_service.py, test_deterministic_checks.py; benchmark/fixtures/; scripts/run_mat_armstrong_fixture.sh.

## Limitations and uncertainty
The diagrams do not imply dubbing generation is implemented here: the provider boundary can pause for an uploaded artifact. Alignment is heuristic unless matching timestamps/speakers are supplied. Missing evidence becomes uncertainty, semantic judging is optional, and human review remains required.

