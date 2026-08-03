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
  <a href="https://github.com/orgs/codellm-devkit/discussions">Discussions</a>
</p>

CodeLLM-DevKit (CLDK) loads a codebase and hands you back a typed object model of it with classes, methods, fields, and call graphs (among other things) through **one consistent `analysis` object**. Instead of token-heavy crawls through files to answer questions like "what calls this method?" or "is this code reachable?", agents and developers can run precise, deterministic lookups against the actual program. The result is grounded answers from ground truth rather than approximations from grepping.

Java, Python, and TypeScript are supported today, with Go, Rust, and C in development.

## Getting Started

- 📚 Read the [Documentation](https://codellm-devkit.info) for guides, core concepts, and common tasks
- ✨ Browse usage [examples and tutorials](https://github.com/codellm-devkit/cldk-tutorial)
- 💻 Install the Python SDK and query your project:

```bash
pip install cldk
```

The Java backend is bundled — you only need a JDK on your `PATH` to analyze Java projects.

```python
import os
from cldk import CLDK
from cldk.analysis import AnalysisLevel

analysis = CLDK(language="java").analysis(
    project_path=os.environ["JAVA_APP_PATH"],
    analysis_level=AnalysisLevel.call_graph,
)

print(len(analysis.get_classes()), "classes")
print(analysis.get_call_graph())
```

Construct a `CLDK` object, build an `analysis` facade pointing at a project, then query it. Only the `language` argument changes across languages.

## Project Structure

- [python-sdk](https://github.com/codellm-devkit/python-sdk) - official Python SDK ([`cldk` on PyPI](https://pypi.org/project/cldk/))
- [typescript-sdk](https://github.com/codellm-devkit/typescript-sdk) - official TypeScript SDK (experimental)
- [codeanalyzer-java](https://github.com/codellm-devkit/codeanalyzer-java) - static analysis for Java/JavaEE using WALA and JavaParser
- [codeanalyzer-python](https://github.com/codellm-devkit/codeanalyzer-python) - static analysis backend for Python
- [codeanalyzer-typescript](https://github.com/codellm-devkit/codeanalyzer-typescript) - static analysis backend for TypeScript/JavaScript
- [cldk-tutorial](https://github.com/codellm-devkit/cldk-tutorial) - start building with CLDK

## Contributing

We welcome contributions of all kinds! Whether you want to fix bugs, improve documentation, or propose new features, please see our [contributing guide](CONTRIBUTING.md) to get started.

Have questions? Join the discussion in our [community forum](https://github.com/orgs/codellm-devkit/discussions) or [Discord server](https://discord.gg/zEjz9YrmqN).

## Reference

To cite Codellm-Devkit, please use the following reference:

```bibtex
@article{krishna2024codellm,
  title={Codellm-Devkit: A Framework for Contextualizing Code LLMs with Program Analysis Insights},
  author={Krishna, Rahul and Pan, Rangeet and Pavuluri, Raju and Tamilselvam, Srikanth and Vukovic, Maja and Sinha, Saurabh},
  journal={arXiv preprint arXiv:2410.13007},
  year={2024}
}
```

## IBM Public Repository Disclosure

Codellm-devkit is an open source project from [IBM Research](https://github.com/IBM) and open to contributions from the entire community. All content in these repositories including code has been provided by IBM under the associated open source software license and IBM is under no obligation to provide enhancements, updates, or support. IBM developers produced this code as an open source project (not as an IBM product), and IBM makes no assertions as to the level of quality nor security.

## Contact

For any questions, feedback, or suggestions, please contact the authors:

| Name | Email |
| ---- | ----- |
| Rahul Krishna | [i.m.ralk@gmail.com](mailto:i.m.ralk@gmail.com) |
| Rangeet Pan | [rangeet.pan@ibm.com](mailto:rangeet.pan@ibm.com) |
| Saurabh Sinha | [sinhas@us.ibm.com](mailto:sinhas@us.ibm.com) |
