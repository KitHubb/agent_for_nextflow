# Nextflow Codex Agent Guidelines

2026/10/06

연구실 내 Nextflow 구축을 도와주는 agent 사용 가이드 문서

So-Yeon Kim

This repository provides project-level instructions for using Codex to develop and maintain Nextflow pipelines in a shared bioinformatics environment.

The main document is [`AGENTS.md`](AGENTS.md). Codex reads this file as repository guidance and applies it when inspecting, creating, or modifying pipeline code in this repository.

## What the guidelines cover

`AGENTS.md` defines the working conventions for:

- organizing DSL2 pipelines using a practical nf-core-style layout;
- deciding when to use modules, workflows, and optional subworkflows;
- connecting processes through explicit and reproducible channels;
- using pinned Singularity images and versioned reference data;
- separating shared execution configuration from project-specific run parameters;
- validating inputs, documenting outputs, and preserving provenance;
- testing changes with linting, nf-test, smoke tests, and `-resume` where applicable.

The guidelines follow nf-core conventions where they improve consistency and maintenance. They do not require a subworkflow for every analysis stage and do not imply that a pipeline is officially nf-core compliant.

## Shared environment

The guidelines assume the following shared locations:

| Purpose | Location |
| --- | --- |
| Pipeline installations | `/data/software/nextflow/` |
| Singularity images | `/data/software/singularity/` |
| Reference data | `/data/Reference/` |

These locations are reserved for shared software, images, and reference resources. Project inputs, outputs, work directories, logs, and temporary files must be supplied at run time and must not be hard-coded in committed configuration files.

## Using this repository

Keep `AGENTS.md` at the root of a Nextflow repository, or copy and adapt it when starting a new pipeline. Before using it elsewhere, review the shared paths and execution profiles for the target environment.

When a local pipeline needs additional rules, add them only where they are specific to that repository. Keep general Nextflow and infrastructure conventions in the root `AGENTS.md` so they remain easy to review and maintain.
