<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/cldk-dark.png">
  <source media="(prefers-color-scheme: light)" srcset="assets/cldk-light.png">
  <img alt="Logo">
</picture>

<p align='center'>
  <a href="https://arxiv.org/abs/2410.13007">
    <img src="https://img.shields.io/badge/arXiv-2410.13007-b31b1b?style=for-the-badge" />
  </a>
  <a href="https://www.python.org/downloads/release/python-3110/">
    <img src="https://img.shields.io/badge/python-3.11-blue?style=for-the-badge" />
  </a>
  <a href="https://opensource.org/licenses/Apache-2.0">
    <img src="https://img.shields.io/badge/License-Apache%202.0-green?style=for-the-badge" />
  </a>
  <a href="https://codellm-devkit.info">
    <img src="https://img.shields.io/badge/GitHub%20Pages-Docs-blue?style=for-the-badge" />
  </a>
  <a href="https://badge.fury.io/py/cldk">
    <img src="https://img.shields.io/pypi/v/cldk?style=for-the-badge&label=codellm-devkit&color=blue" />
  </a>
  <a href="https://discord.com/channels/1333486179667935403/1334150434348208208">
    <img src="https://dcbadge.limes.pink/api/server/https://discord.gg/zEjz9YrmqN?style=for-the-badge"/>
  </a>
</p>



<p align="center">
  <strong>A framework that bridges the gap between traditional program analysis tools and Large Language Models (LLMs) specialized for code (CodeLLMs).</strong>
</p>

<p align="center">
  <a href="https://codellm-devkit.github.io/docs">Documentation</a> |
  <a href="https://github.com/orgs/codellm-devkit/discussions">Discussions</a>
</p>

Codellm-Devkit (CLDK) is a multilingual program analysis framework that bridges the gap between traditional static analysis tools and Large Language Models (LLMs) specialized for code (CodeLLMs). Codellm-Devkit allows developers to streamline the process of transforming raw code into actionable insights by providing a unified interface for integrating outputs from various analysis tools and preparing them for effective use by CodeLLMs.

## Getting Started

- 📚 Read the [Documentation](https://codellm-devkit.github.io/docs) for guides and tutorials
- 💻 Use our SDKs to start building:
  - [Python SDK](https://github.com/codellm-devkit/python-sdk)
  - [TypeScript SDK](https://github.com/codellm-devkit/typescript-sdk)
- 🤖 CLDK wrappers for [Tool Calling](https://github.com/codellm-devkit/cldk-tool-calling) with various LLMs.
- ✨ CLDK usage [examples](https://github.com/codellm-devkit/cldk-examples)

## Software Development Kits (SDKs)

- [python-sdk](https://github.com/codellm-devkit/python-sdk) - Python implementation
- [typescript-sdk](https://github.com/codellm-devkit/typescript-sdk) - TypeScript implementation

## Language Specific Analysis Backends
- [codeanalyzer-java](https://github.com/codellm-devkit/codeanalyzer-java) - Uses WALA and Java Parser to provide static analysis artifacts for Java and JavaEE projects.
- [codeanalyzer-rs](https://github.com/codellm-devkit/codeanalyzer-rs) - Provides static analysis artifacts for Rust projects.

## CodeLLM usage plugins  
- [cldk-tool-calling](https://github.com/codellm-devkit/cldk-tool-calling) - Wrappers for various tool calling with LLMs

## Documentation and Examples
- [docs](https://github.com/codellm-devkit/docs) - User documentation and guides
- [cldk-examples](https://github.com/codellm-devkit/cldk-examples) - several usage examples with CLDK

## Contributing

We welcome contributions of all kinds! Whether you want to fix bugs, improve documentation, or propose new features, please see our [contributing guide](CONTRIBUTING.md) to get started.

Have questions? Join the discussion in our [community forum](https://github.com/orgs/codellm-devkit/discussions) or [Discord server](https://discord.com/channels/1333486179667935403/1334150434348208208).

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
