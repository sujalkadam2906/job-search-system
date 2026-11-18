# Notice and attribution

This repository is derived from **PWP_jobhunt** by Pratyush Narain
(<https://github.com/perryvegehan/PWP_jobhunt>). The upstream project did not include a
LICENSE file at the time of copying, so no open-source license is granted here. Confirm
reuse terms with the original author before relying on this code beyond personal use.

## Original work (upstream)

The pipeline design, the `jobhunt/` package (fetch, prefilter, providers, llm, digest,
mailer, store, mock, cli), the test suite, the GitHub Actions workflow and the setup guides.

## Changes in this copy

- `config.yaml`: filters retuned for fresher / entry-level Java, backend and full-stack roles in India.
- `companies.yaml`: edited company list.
- `jobhunt/cli.py`: the CLI now changes to the project root and loads `.env` from there, so it works from any directory.
- `pyproject.toml`: added packaging metadata.
- `tests/`: the filter fixture is now frozen in `tests/fixtures/test_config.yaml` so personal edits to `config.yaml` no longer break two tests.
- `.env.example`: contains placeholders only.
- Removed personal files (`.env`, `profile.json`, `resume.pdf`, `seen.json`, virtualenv, build artifacts) and the upstream git history.
