<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/cldk-dark.png">
  <source media="(prefers-color-scheme: light)" srcset="assets/cldk-light.png">
  <img src="assets/cldk-light.png" alt="CodeLLM-DevKit">
</picture>

<p align='center'>
  <a href="https://codellm-devkit.info">
    <img src="https://img.shields.io/badge/GitHub%20Pages-Docs-blue?style=for-the-badge" />
  </a>
  <a href="https://pypi.org/project/cldk/">
    <img src="https://img.shields.io/pypi/v/cldk?style=for-the-badge&label=CLDK&color=blue&logo=python" />
  </a>
  <a href="https://arxiv.org/abs/2410.13007">
    <img src="https://img.shields.io/badge/arXiv-2410.13007-b31b1b?style=for-the-badge" />
  </a>
  <a href="https://opensource.org/license/apache-2.0">
    <img src="https://img.shields.io/badge/License-Apache%202.0-green?style=for-the-badge" />
  </a>
</p>

<p align="center">
  <strong>One analysis interface over every language: call graphs, symbol tables, and reachability — program analysis your agents can call.</strong>
</p>

<p align="center">
  <a href="https://codellm-devkit.info">Documentation</a> |
  <a href="https://codellm-devkit.info/quickstart/">Quickstart</a> |
  <a href="https://github.com/orgs/codellm-devkit/discussions">Discussions</a> |
  <a href="https://discord.gg/zEjz9YrmqN">Discord</a>
</p>

CodeLLM-DevKit (CLDK) loads a codebase and hands you back a typed object model of it — classes, methods, fields, call graphs, and dataflow — through **one consistent `analysis` object**. Instead of token-heavy crawls through files to answer questions like "what calls this method?" or "is this code reachable?", agents and developers run precise, deterministic lookups against the actual program. The result is grounded answers from ground truth rather than approximations from grepping.

Every language is analyzed by a dedicated `codeanalyzer-*` engine that emits the same [canonical schema](https://github.com/codellm-devkit/codeanalyzer-schema) — as `analysis.json` or as a Neo4j property graph — and the SDK normalizes it into the same typed API. Java, Python, TypeScript/JavaScript, and Infrastructure as Code (Helm) ship today; Go, Rust, C/C++, and more are in development.

## Getting Started

- 📚 Read the [documentation](https://codellm-devkit.info) for guides, core concepts, and common tasks
- ✨ Browse [examples and tutorials](https://github.com/codellm-devkit/cldk-tutorial)
- 💻 Install the Python SDK and query your project:

```bash
pip install cldk
```

```python
from cldk import CLDK
from cldk.analysis import AnalysisLevel

analysis = CLDK.python(
    project_path="/path/to/python/project",
    analysis_level=AnalysisLevel.call_graph,
)

print(len(analysis.get_classes()), "classes")
print(analysis.get_call_graph())
```

Swap `CLDK.python(...)` for `CLDK.java(...)` or `CLDK.typescript(...)` — the query API is the same across languages. The Python and TypeScript analyzers install as dependencies and the Java analyzer is bundled, so `pip install cldk` is all you need — analyzing Java projects only needs a JDK on your `PATH`.

## Repositories

### Python SDK

| Repository | What it is | Install |
| --- | --- | --- |
| [python-sdk](https://github.com/codellm-devkit/python-sdk) | The official Python SDK. Typed facades over every analyzer below, plus a read-only Neo4j backend for graphs populated out of band. | `pip install cldk`<br>`pip install "cldk[neo4j]"` |

### Analyzers

Each analyzer is a standalone CLI. All four ship on PyPI, so `pip install` works without a language toolchain: Python, TypeScript, and IaC as prebuilt binaries, Java as a JAR with a bundled JVM. Homebrew formulas live in [homebrew-tap](https://github.com/codellm-devkit/homebrew-tap); Java also has a one-line installer for the bare JAR.

| Repository | Language | CLI | Install |
| --- | --- | --- | --- |
| [codeanalyzer-java](https://github.com/codellm-devkit/codeanalyzer-java) | Java / Jakarta EE (source or bytecode), via WALA and JavaParser. The wheel bundles a JVM; the installer needs Java 11+. | `canjv` | `pip install codeanalyzer-java`<br>`curl --proto '=https' --tlsv1.2 -LsSf https://github.com/codellm-devkit/codeanalyzer-java/releases/latest/download/codeanalyzer-installer.sh \| sh` |
| [codeanalyzer-python](https://github.com/codellm-devkit/codeanalyzer-python) | Python. Symbol table, call graph, and native CFG/PDG/SDG dataflow. | `canpy` | `pip install codeanalyzer-python`<br>`pip install "codeanalyzer-python[neo4j]"`<br>`brew install codellm-devkit/tap/codeanalyzer-python` |
| [codeanalyzer-typescript](https://github.com/codellm-devkit/codeanalyzer-typescript) | TypeScript / JavaScript. Symbols, call graph, types, decorators, and intra/interprocedural dataflow. | `cants` | `pip install codeanalyzer-typescript`<br>`brew install codellm-devkit/tap/codeanalyzer-typescript` |
| [codeanalyzer-iac](https://github.com/codellm-devkit/codeanalyzer-iac) | Infrastructure as Code, starting with Helm charts. Rendered-resource graph with typed configuration facets. | `caniac` | `pip install codeanalyzer-iac`<br>`brew install codellm-devkit/tap/codeanalyzer-iac` |

All analyzers also read and write the shared contract in [codeanalyzer-schema](https://github.com/codellm-devkit/codeanalyzer-schema).

### Experimental

Under active development or exploratory. APIs, output, and packaging may change without notice.

- [codeanalyzer-go](https://github.com/codellm-devkit/codeanalyzer-go) - static analysis backend for Go
- [codeanalyzer-clang](https://github.com/codellm-devkit/codeanalyzer-clang) - static analysis backend for the C language family
- [codeanalyzer-rust](https://github.com/codellm-devkit/codeanalyzer-rust) - static analysis backend for Rust, built on the compiler's IR
- [codeanalyzer-kotlin](https://github.com/codellm-devkit/codeanalyzer-kotlin) - static analysis backend for Kotlin
- [codeanalyzer-swift](https://github.com/codellm-devkit/codeanalyzer-swift) - static analysis backend for Swift
- [codeanalyzer-abap](https://github.com/codellm-devkit/codeanalyzer-abap) - static analysis backend for ABAP
- [codeanalyzer-codeql](https://github.com/codellm-devkit/codeanalyzer-codeql) - multi-language backend on top of CodeQL
- [typescript-sdk](https://github.com/codellm-devkit/typescript-sdk) - TypeScript SDK
- [cocoa-mcp](https://github.com/codellm-devkit/cocoa-mcp) - Code Context Agent and toolbox MCP server (Python)
- [cocoa-ts](https://github.com/codellm-devkit/cocoa-ts) - Code Context Agent and toolbox MCP client/server (TypeScript)

### Tooling & docs

- [docs](https://github.com/codellm-devkit/docs) - source for [codellm-devkit.info](https://codellm-devkit.info)
- [cldk-tutorial](https://github.com/codellm-devkit/cldk-tutorial) - worked examples and notebooks
- [cldk-devtools](https://github.com/codellm-devkit/cldk-devtools) - agent skills for extending and maintaining CLDK
- [homebrew-tap](https://github.com/codellm-devkit/homebrew-tap) - Homebrew formulas for the analyzers

## Contributing

We welcome contributions of all kinds! Whether you want to fix bugs, improve documentation, or propose new features, please see our [contributing guide](CONTRIBUTING.md) to get started.

Have questions? Join the discussion in our [community forum](https://github.com/orgs/codellm-devkit/discussions) or [Discord server](https://discord.gg/zEjz9YrmqN).

## Reference

To cite CodeLLM-DevKit, please use the following reference:

```bibtex
@article{krishna2024codellm,
  title={Codellm-Devkit: A Framework for Contextualizing Code LLMs with Program Analysis Insights},
  author={Krishna, Rahul and Pan, Rangeet and Pavuluri, Raju and Tamilselvam, Srikanth and Vukovic, Maja and Sinha, Saurabh},
  journal={arXiv preprint arXiv:2410.13007},
  year={2024}
}
```

## IBM Public Repository Disclosure

CodeLLM-DevKit is an open source project from [IBM Research](https://github.com/IBM) and open to contributions from the entire community. All content in these repositories including code has been provided by IBM under the associated open source software license and IBM is under no obligation to provide enhancements, updates, or support. IBM developers produced this code as an open source project (not as an IBM product), and IBM makes no assertions as to the level of quality nor security.

## Contact

For any questions, feedback, or suggestions, please contact the authors:

| Name | Email |
| ---- | ----- |
| Rahul Krishna | [i.m.ralk@gmail.com](mailto:i.m.ralk@gmail.com) |
| Rangeet Pan | [rangeet.pan@ibm.com](mailto:rangeet.pan@ibm.com) |
| Saurabh Sinha | [sinhas@us.ibm.com](mailto:sinhas@us.ibm.com) |
