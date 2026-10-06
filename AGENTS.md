# Nextflow pipeline development guidelines

This repository contains Nextflow pipelines for shared bioinformatics analyses. Changes should be small, reproducible, and tested with representative data. Keep existing repository conventions unless there is a clear reason to change them.

All code, comments, identifiers, commit messages, and documentation should be written in English.

## Shared environment

The production environment uses the following shared locations:

| Purpose | Location |
| --- | --- |
| Pipeline installations | `/data/software/nextflow/` |
| Singularity images | `/data/software/singularity/` |
| Reference data | `/data/Reference/` |

Install production pipelines under `/data/software/nextflow/<pipeline-name>/`. Do not place results, work directories, logs, or temporary files under any of the shared locations above.

Reuse images already available in `/data/software/singularity/`. Do not overwrite or delete shared images. Reference genomes and annotations must come from `/data/Reference/`, with the assembly and release selected explicitly.

## Repository layout

Use Nextflow DSL2 and follow the parts of the nf-core layout that are useful for the pipeline:

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

Keep `main.nf` limited to parameter setup, input validation, channel creation, and the main workflow call. Put the end-to-end analysis in `workflows/` and single-tool tasks in `modules/`.

Use an existing nf-core module when it fits the analysis. Pipeline-specific processes belong in `modules/local/`, normally with one process per module.

Subworkflows are optional. Do not create one for each analysis stage. Add a subworkflow only when a group of modules is reused, has a useful standalone interface, or is difficult to understand in the main workflow.

## Processes and channels

- Connect processes with channels rather than by scanning output directories.
- Give inputs and outputs explicit contracts and use named `emit` values where this improves readability.
- Keep sample metadata with its files, usually as a tuple such as `[meta, file]`.
- Do not rely on the arrival order of separate channels to match related files. Keep related files together or join them by a stable key.
- Do not modify input files in place. This breaks reproducibility and can prevent `-resume` from working as expected.
- Give each process a useful `tag` and an appropriate resource `label`.
- Use `task.cpus` and `task.memory` in tool commands when the tool supports them.
- Move long Python, R, or shell code to `bin/`. If it has substantial dependencies or its own release cycle, package it separately and run it from a versioned container.
- Record tool and container versions where practical.

## Containers

Production processes should use pinned container versions. Do not use mutable tags such as `latest`.

Prefer an existing `.sif` image in `/data/software/singularity/`. An image used directly by a process should have an absolute URI such as:

```text
file:///data/software/singularity/<image>.sif
```

For remote images, pin a release or digest. Do not install software on a compute node from inside a process.

Check that all input and reference paths are available inside the container. For symbolic links, the link target must also be on an accessible bind path.

## Configuration

Committed configuration is limited to settings shared across analyses, including executors, queues, process labels, resource defaults, retry behavior, reports, and the shared container and reference locations.

Do not put project-specific paths in `nextflow.config` or any file under `conf/`. This includes input data, sample sheets, output directories, work directories, logs, scratch space, temporary directories, and project-specific references.

Run-specific values should be provided on the command line or through `-params-file`. In particular, do not assign a project path to `params.input`, `params.outdir`, or `workDir` in committed configuration.

Use the configuration files as follows:

- `conf/base.config`: resource labels and common retry settings
- `conf/modules.config`: process-specific options, `ext.args`, and publication rules
- `conf/test.config`: small test inputs and test-only settings
- `nextflow.config`: manifest, shared defaults, and profile composition

Prefer `withLabel` for resource classes. Use `withName` only for a process-specific exception. Keep profiles focused on the execution environment, for example `singularity`, `slurm`, and `test`.

A shared Singularity profile can use the curated image directory as a library:

```groovy
profiles {
    singularity {
        singularity.enabled    = true
        singularity.autoMounts = true
        singularity.libraryDir = '/data/software/singularity/'
    }
}
```

Use `singularity.cacheDir` only if `/data/software/singularity/` is managed as a writable shared cache. For a read-only collection of approved images, use `singularity.libraryDir`.

Never commit usernames, credentials, access tokens, or private registry secrets.

## Parameters and references

Define user-facing parameters in `nextflow_schema.json` when the repository uses nf-schema or nf-core tooling. Validate required files, sample identifiers, paired inputs, allowed values, and dependent options before starting expensive processes.

Select references below `/data/Reference/` through an assembly/release parameter or a versioned manifest. Do not infer the reference build from a filename, and do not copy large reference files into the repository. Check that indexes match both the reference assembly and the tool that will use them.

Sample-sheet errors should identify the affected row and field.

## Outputs and provenance

Publish final and otherwise useful results, not every intermediate file. Group outputs by analysis purpose, use stable sample identifiers in filenames, and document the output layout.

Where appropriate, enable the Nextflow report, timeline, trace, and DAG. Record the pipeline revision, Nextflow version, resolved parameters, and tool/container versions. Do not include credentials or sensitive values in reports.

Keep both `.nextflow/cache` and the work directory while a run may need to be resumed. Do not clean either automatically.

## Tests

Use small synthetic or openly redistributable data for routine tests. Run the checks that apply to the change:

1. Format and lint the changed files.
2. Inspect the resolved configuration for the intended profile.
3. Run relevant nf-test tests when available.
4. Run a pipeline smoke test or `-stub-run` with the test profile.
5. Run `nf-core pipelines lint` for repositories based on the nf-core template.
6. Run a small end-to-end test when suitable data and containers are available.
7. Check `-resume` behavior when process inputs, outputs, names, or workflow structure have changed.

Do not submit cluster jobs, download large images or references, or run production datasets unless the user has asked for it. If a check cannot be run in the development environment, state which check remains and provide the command for the target system.

## Documentation

Keep the README and usage documentation consistent with the pipeline. Document the expected input format, parameters, reference build, output layout, container requirements, test command, and a production command template.

Use placeholders such as `<samplesheet.csv>`, `<outdir>`, and `<work-dir>` in examples. Do not use an actual project path in documentation or committed configuration.

## Before finishing a change

- Review the current diff and leave unrelated files unchanged.
- Confirm that channel shapes and metadata remain consistent between processes.
- Confirm that config files contain no project-specific paths.
- Run the relevant tests and lint checks.
- Update documentation when inputs, outputs, parameters, or runtime behavior have changed.
- Report any assumption that affects the scientific result and any check that was not run.
