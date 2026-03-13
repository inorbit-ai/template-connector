# InOrbit Template Connector

This is a template repository for building InOrbit connectors. It contains minimal Github settings and no actual contents, which are managed by our cookiecutter package.

## Getting Started

1. Create a new repository using this repository as a template.
2. Run `pipx run cookiecutter gh:inorbit-ai/inorbit-connector-cookiecutter`. This will install the [cookiecutter package](https://cookiecutter.readthedocs.io/) if you don't have it already and download the InOrbit connector cookiecutter.
3. Follow the prompts to create a new connector.
4. Once all questions are answered, the cookiecutter will generate a bunch of files and directories for you to start working on your connector.
5. Read the notice at the top of the generated README.md file and follow the instructions to complete the setup.

## AI-Assisted Development

This template includes an `inorbit-connector-developer` skill for [Claude Code](https://claude.com/claude-code) to aid in the building of a connector from scratch or improving an existing one. To use it:

1. Open Claude Code in your connector repository (potentially created from this very template)
2. Describe what you want to build (e.g., "Build a connector for the Acme Fleet API" or "Add battery monitoring to this connector")
3. Invoke the skill when prompted, or reference it directly

The skill lives in `.claude/skills/inorbit-connector-developer/` and is carried over when you generate a new project from this template.

## Contributing

For contributing to the cookiecutter package refer to https://github.com/inorbit-ai/inorbit-connector-cookiecutter

If you have any questions or feedback, please feel free to reach out to us at support@inorbit.ai.
