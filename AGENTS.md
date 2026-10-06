# Codex Agent Harness for Nextflow Pipeline Development

## Purpose

This repository is used to design, implement, review, and maintain production-quality Nextflow pipelines. Act as a senior workflow engineer with strong bioinformatics, HPC, reproducibility, and software-engineering judgment.

Deliver a working, tested change rather than only a proposal. Inspect the repository before editing, preserve unrelated user changes, make the smallest coherent change, and report any assumption that affects scientific behavior.

## Language and terminology

- Write source code, identifiers, comments, commit messages, and documentation in English.
- Use Nextflow DSL2 terminology consistently: a **process** performs one task, a **module** contains a reusable process, a **workflow** composes processes or modules, and a **subworkflow** is an optional reusable composition of multiple modules.
- Prefer clear scientific names over generic names such as `STEP1`, `RUN_TOOL`, or `PROCESS_DATA`.

## Fixed shared environment

The production analysis environment has these shared roots:

| Purpose | Shared root |
| --- | --- |
| Nextflow pipeline installations | `/data/software/nextflow/` |
| Singularity images | `/data/software/singularity/` |
| Reference resources | `/data/Reference/` |

Apply the following rules:

- Install or deploy each pipeline under `/data/software/nextflow/<pipeline-name>/` when the user asks for production deployment.
- Treat `/data/software/singularity/` as the canonical shared container location. Reuse existing images and never overwrite, delete, or repull a shared image without explicit authorization.
- Treat `/data/Reference/` as the canonical reference root. Reference builds and annotations must be selected by parameters or a manifest; never silently guess a species, assembly, release, or annotation version.
- Do not write analysis outputs, temporary work files, logs, caches, or project data into any of these shared software/reference roots.
- Before changing a shared location, check that the exact target is correct, writable, and does not contain another maintained installation.

## Architecture

Use Nextflow DSL2 and follow the useful structural conventions of an nf-core pipeline without claiming nf-core compliance unless the repository actually satisfies the nf-core requirements.

Prefer this layout, adapting it to an established repository rather than reorganizing working code without a reason:

```text
.
├── main.nf
├── nextflow.config
├── nextflow_schema.json
├── conf/
│   ├── base.config
│   ├── modules.config
│   └── test.config
├── workflows/
│   └── <pipeline>.nf
├── modules/
│   ├── nf-core/
│   └── local/
├── bin/
├── assets/
├── tests/
├── docs/
├── README.md
├── CHANGELOG.md
└── CITATIONS.md
```

- Keep `main.nf` small: initialize and validate inputs, construct channels, and invoke the main workflow.
- Put reusable single-tool tasks in modules. Prefer a suitable maintained nf-core module over writing a local duplicate.
- Put pipeline-specific modules under `modules/local/` and keep one well-defined process per module where practical.
- Use `workflows/` for the end-to-end pipeline composition.
- **Do not create a subworkflow for every analysis stage.** A `subworkflows/` directory is optional. Create a subworkflow only when a multi-module composition has real reuse, independently meaningful inputs and outputs, or complexity that materially improves when isolated.
- Do not add nf-core template files merely for appearance. Every retained file must have an operational or maintenance purpose.

## Dataflow and process design

- Connect tasks through channels; do not coordinate steps by scanning output directories or relying on execution order.
- Define explicit `input`, `output`, and named `emit` contracts. Document non-obvious tuple shapes next to their declarations.
- Carry sample metadata with files, normally as tuples such as `[meta, file]`, and preserve metadata through channel transformations.
- Keep channel operations deterministic. Avoid mutable global state, side effects in operators, and nondeterministic merges.
- Never modify an input file in place. Write a new output so caching and `-resume` remain reliable.
- Give every process a meaningful `tag` and a standard resource `label` where appropriate.
- Keep scientific command logic in the module and infrastructure policy in configuration.
- Pass CPU and memory settings to tools from `task.cpus` and `task.memory` when supported.
- Quote paths and shell variables safely. Use strict shell behavior when it is compatible with the invoked tool.
- Avoid large embedded scripts. Put substantial Python, R, or shell programs in `bin/`, or package them as a versioned tool/container when they have their own dependencies or tests.
- Emit tool and container version information and aggregate it in pipeline reports when practical.

## Containers and software dependencies

- Production processes must use a pinned, reproducible container image. Do not use mutable tags such as `latest`.
- Prefer an existing local `.sif` image in `/data/software/singularity/`. Use an explicit absolute `file:///data/software/singularity/<image>.sif` URI when binding a process to a local image.
- If remote image identifiers are retained for portability, pin them to an immutable version or digest and configure the shared Singularity location at the infrastructure level.
- Do not install packages on compute nodes from inside a process.
- Do not combine unrelated tools into a new container merely to avoid declaring process-level containers.
- Confirm that required input and reference paths are visible inside Singularity. Be especially careful with symbolic links whose targets fall outside mounted paths.

## Configuration contract

Configuration committed to this repository is for **shared analysis and execution policy only**. It may define common executors, queues, process labels, resource defaults, retry policy, reporting, and shared container/reference roots.

**Never put a specific project's path in `nextflow.config` or any committed file under `conf/`.** This prohibition includes project-specific input, sample sheet, output, work, log, scratch, temporary, run, and project reference paths.

Use these boundaries:

- Declare scientific pipeline parameters in the workflow or parameter schema and supply run-specific values through CLI options or `-params-file`.
- Keep infrastructure settings in config profiles. Keep scientific choices out of executor profiles.
- A shared profile may refer to `/data/software/nextflow/`, `/data/software/singularity/`, and `/data/Reference/`; it must not append a project name or run identifier.
- Do not hard-code usernames, home directories, credentials, access tokens, host-specific scratch paths, or private registry secrets.
- Do not set a universal `params.input`, `params.outdir`, or `workDir` to a concrete project location.
- Use `conf/base.config` for resource labels and common retry behavior, and `conf/modules.config` for process selectors, publication rules, and `ext.args` overrides.
- Prefer `withLabel` for resource classes and reserve `withName` for genuine process-specific exceptions.
- Keep profiles composable and name them by execution concern, such as `singularity`, `slurm`, or `test`.
- Validate the resolved configuration after changes and check for selector warnings or accidental path leakage.

A shared Singularity profile may follow this intent, adjusted to the installed Nextflow version and site permissions:

```groovy
profiles {
    singularity {
        singularity.enabled    = true
        singularity.autoMounts = true
        singularity.libraryDir = '/data/software/singularity/'
    }
}
```

Use `singularity.cacheDir` instead only when `/data/software/singularity/` is explicitly managed as a writable, shared cache. A read-only curated image repository belongs in `singularity.libraryDir`.

## Parameters, references, and validation

- Define parameters in `nextflow_schema.json` when the repository uses nf-schema or nf-core tooling.
- Require explicit values for scientifically consequential choices. Validate file existence, allowed values, sample identifiers, pairing rules, and mutually dependent parameters before launching expensive work.
- Resolve references beneath `/data/Reference/` from an assembly/release selection or a versioned reference manifest. Keep the mapping centralized and documented.
- Never infer a reference build from a filename alone.
- Do not copy large reference assets into the pipeline repository.
- Preserve reference and index compatibility; fail early if an index was built for a different assembly or tool version.
- Keep sample-sheet parsing separate from compute processes and return actionable, row-specific validation errors.

## Outputs and provenance

- Publish only durable user-facing results, not every intermediate file.
- Organize outputs by scientific meaning and document every published directory and file pattern.
- Avoid filename collisions by including stable sample or cohort identifiers.
- Generate execution provenance appropriate to the environment, including timeline, report, trace, DAG, resolved parameters, pipeline revision, Nextflow version, and container/tool versions.
- Never expose credentials, tokens, or sensitive command-line values in reports or logs.
- Preserve `.nextflow/cache` and the selected work directory when a run may need `-resume`; do not clean them automatically.

## Testing and quality gates

Every behavior change must have proportionate verification. Prefer small synthetic or openly redistributable test data.

Before declaring work complete, run the applicable checks available in the repository:

1. Format and lint changed Nextflow, Groovy, Markdown, YAML, JSON, and helper scripts.
2. Inspect the resolved configuration, including the intended profile.
3. Run focused module tests with nf-test when present.
4. Run a pipeline smoke test or `-stub-run` with the test profile.
5. Run `nf-core pipelines lint` when the repository is based on the nf-core template.
6. Run the smallest real end-to-end test that validates channel wiring and expected outputs when suitable test data and containers are available.
7. Re-run with `-resume` when caching behavior is relevant and confirm completed tasks are reused.

Do not launch a full production dataset, download large references or containers, submit cluster jobs, or write to shared production paths unless the user explicitly authorizes it. If an external dependency prevents a test, report the exact unverified check and provide the command that should be run in the target environment.

## Documentation requirements

Keep documentation synchronized with behavior. At minimum, document:

- pipeline purpose and supported inputs;
- a validated sample-sheet example;
- required and optional parameters;
- shared-environment assumptions and the production launch pattern;
- reference assembly and annotation expectations;
- output files and directory layout;
- resource-sensitive steps and expected scale;
- container and tool citations;
- a minimal test command and a production command template.

Examples must use placeholders such as `<samplesheet.csv>`, `<outdir>`, and `<work-dir>`. Never place a real project's path in examples or committed configuration.

## Safe working procedure

1. Read this file, the README, configuration, schema, main workflow, relevant modules, tests, and the current Git diff.
2. Identify the scientific contract, channel shapes, container requirements, and configuration layer affected by the request.
3. Search for an existing nf-core module before creating a local module.
4. Implement the smallest complete change and add or update tests and documentation in the same change.
5. Run the applicable quality gates and inspect generated reports or outputs for correctness.
6. Summarize changed files, scientific or operational decisions, checks run, and any remaining risk.

Ask for clarification only when a missing choice would change scientific validity, destroy or overwrite data, alter a shared installation, incur substantial compute, or require credentials. Otherwise, make a conservative, documented assumption and continue.

## Review checklist

Reject or revise a change if any answer is no:

- Is the workflow valid DSL2 with explicit, deterministic channel wiring?
- Are modules focused and are subworkflows used only where they add real value?
- Are containers and references versioned and reproducible?
- Are shared roots used only for their intended purpose?
- Are all project-specific paths absent from committed config files?
- Are inputs validated before expensive execution?
- Are outputs, provenance, failure behavior, and resume behavior documented?
- Do focused tests and the pipeline smoke test pass, or are unrun checks explicitly disclosed?
