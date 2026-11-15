# Test cases for maintainers of this skill

These folders are tests of the skill, not instructions for using it. An agent that applies the
skill to a project should ignore this folder.

Each case is a folder in the layout `claude plugin eval` reads: `prompt.md` (run settings and
the user's prompt), `case.yaml` (what a good result looks like), `graders/` (one assertion per
file) and `files/` (input files copied into the run's working directory by `place-files.sh`).
Run them with a fresh agent that has the skill, or from the skill directory with
`claude plugin eval . --scaffold`. Results belong in a workspace next to the skill, not here.
The packaging script of the skill-creator leaves this folder out.
