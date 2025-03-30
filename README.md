# D4.2: Technical specifications, in machine-actionable metadata formats, for common workflow interfaces

This repository contains [technical specifications](https://github.com/ord-premise/common-workflow-schemas/blob/main/technical_specifications.md) for OO-LD (Object-Oriented Linked Data) schemas designed to standardize input/output interfaces for common materials science workflows. Built atop JSON Schema and JSON-LD, these schemas combine structural validation with semantic annotation to support:

- Interoperability through JSON-LD contexts and IRIs
- UI auto-generation from JSON Schema definitions

- Code-agnostic interfaces that abstract over simulation backends (e.g., VASP, QE, CP2K)

- Engine-agnostic compatibility with workflow systems (e.g., AiiDA, pyiron, FireWorks)

OO-LD enables both humans and machines to construct, validate, share, and interpret workflow definitions and results in a unified, extensible, and FAIR-compliant manner.

## Authors

- Edan Bainglass (PSI)
- Giovanni Pizzi (PSI)

## Goal

To provide specifications for standard schemas of common materials science workflows, including inputs, processes, and outputs. We will first deliver input/output schemas generalizing the inputs and outputs of the AiiDA common workflows (ACWF). Future work will aim at generalizing the schemas of the workflows themselves, providing a standard language for driving common materials science workflows. We will also in the future develop UIs for building and submitting common materials science workflows automatically constructed from the schemas.

## Achievement

We developed engine-agnostic pydantic schemas for input/output components for common materials science workflows leveraging a code-agnostic structure geometry optimization (relaxation) workflow. The models are enriched with semantic annotation and export functionality to OO-LD documents, supporting JSON-LD algorithms for data interoperability. We provide an example notebook demonstrating the use of OO-LD documents in transferring workflow input based on engine-agnostic schemas down to the AiiDA workflow engine. AiiDA uses the transferred data and metadata to drive a structure relaxation workflow, yielding output in its own format. The notebook further demostrates conversion back to an engine-agnostic output schema.

## External links

- [Common workflow schemas repo](https://github.com/edan-bainglass/common-workflow-schemas)
- [OO-LD specifications](https://github.com/OO-LD/schema)
- [AiiDA Common Workflows paper](https://www.nature.com/articles/s41524-021-00594-6)
- [AiiDA Common Workflows documentation](https://aiida-common-workflows.readthedocs.io/en/latest/)

## Acknowledgements

The [PREMISE](https://ord-premise.org/) project is supported by the [Open Research Data Program](https://ethrat.ch/en/eth-domain/open-research-data/) of the ETH Board.

![image](https://ord-premise.org/assets/img/logos/PREMISE-logo.svg)

![image](https://ethrat.ch/wp-content/uploads/2021/12/ethr_en_rgb_black.svg)
