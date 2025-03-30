# Technical Specification: Authoring Semantically-Enriched Object-Oriented Schemas for Common Materials Science Workflows

## 1. Introduction

Cross-platform interoperability of Open Research Data (ORD) remains a fundamental challenge in modern scientific research. As research becomes increasingly data-driven, collaborative, and automated, the need for shared understanding of data and metadata across diverse systems grows rapidly. Different tools, platforms, and communities often define and manage their data in incompatible ways, making it difficult to achieve true interoperability. Without consistent semantics and structure, the promise of FAIR (Findable, Accessible, Interoperable, Reusable) data remains only partially realized. This challenge spans both computational and experimental domains and must be understood and standardized, as the future of AI-driven laboratories will be powered by seamless exchange and interoperability of ORD between experiment (hardware) and simulation (software) platforms.

In the domain of computational materials science, this challenge is compounded by the diversity of workflow engines (e.g. AiiDA, pyiron, FireWorks) and simulation codes (e.g. Quantum ESPRESSO, VASP, CP2K). Despite often targeting the same computational tasks, each of these systems defines its own input and output formats, making workflow portability and reproducibility a persistent challenge. What is needed is a set of common, semantically-enriched schemas that describe simulation workflows, their inputs, and outputs in a code- and engine-agnostic way.

To this end, we propose a standards-driven framework for defining and exchanging reusable workflow schemas across the materials science community. These schemas serve as common ground for communication between diverse platforms and enable scalable, machine-readable, and semantically meaningful interaction between experimental and simulation systems.

## 2. Terminology

This section defines key terms used throughout this specification:

- **Open Research Data (ORD)**: Scientific data that is freely accessible, reusable, and shareable in compliance with FAIR principles.
- **FAIR**: An acronym describing data that is Findable, Accessible, Interoperable, and Reusable.
- **Interoperability**: The ability of different platforms, systems, or tools to exchange, interpret, and reuse data with minimal transformation.
- **OO-LD (Object-Oriented Linked Data)**: A document format that enriches a JSON Schema with JSON-LD semantics, combining structural validation with machine-readable context.
- **JSON Schema**: A vocabulary that allows the definition and validation of the structure of JSON documents.
- **JSON-LD**: A lightweight Linked Data format for encoding semantic meaning in JSON documents using IRIs and contexts.
- **IRI (Internationalized Resource Identifier)**: A global identifier used in JSON-LD to uniquely identify terms in the context of Linked Data.
- **Pydantic**: A Python library for defining data models with type annotations, validation, and JSON Schema export capabilities.
- **SemanticModel**: A base class used in this work to build semantic-rich Pydantic models that expose a `.model_oo_ld()` method for OO-LD generation.
- **Workflow Engine**: Software responsible for executing sequences of tasks, such as AiiDA, pyiron, or FireWorks.
- **Simulation Code**: A scientific code used for atomistic simulations, such as VASP, Quantum ESPRESSO, or CP2K.
- **StructureResource**: A standard OPTIMADE object used to describe atomic structures and their properties.

## 3. Problem Statement

Current approaches to defining simulation workflows in materials science are fragmented and tightly coupled to the specific engines or codes that execute them. Inputs and outputs are often defined in ad hoc formats, with little consideration for reusability across tools or clarity in semantics. This fragmentation makes it difficult to build interoperable platforms, slows down collaborative efforts, and introduces unnecessary effort in re-implementation or translation between systems.

Moreover, the lack of standardized, machine-readable semantic meaning in these definitions prevents software agents—such as workflow engines, optimizers, or user interfaces—from interpreting and exchanging data autonomously. The result is a proliferation of tool-specific interfaces and a failure to deliver on the promise of AI-assisted or autonomous materials design workflows.

## 4. Proposed Solution

To address these challenges, we propose a structured and semantic approach to schema authoring using [Object-Oriented Linked Data (OO-LD)](https://github.com/OO-LD/schema). OO-LD combines two proven technologies:

- **[JSON Schema](https://json-schema.org/)**: for describing the structure and validation rules of data objects, and for enabling automatic generation of user interface (UI) forms to assist users in preparing valid inputs.
- **[JSON-LD](https://json-ld.org/)**: for embedding semantic context in JSON documents via linked data principles, enabling structured, machine-interpretable metadata.

We define reusable, extensible, and semantically annotated input/output schemas for common materials workflows. These schemas are:

- **Engine-agnostic**: decoupled from any specific workflow engine like AiiDA or pyiron
- **Code-agnostic**: applicable across different ab-initio simulation codes like VASP, QE, and CP2K
- **Human- and machine-readable**: compatible with both form generation and autonomous agent consumption

OO-LD documents thus serve a dual role: (1) providing a standardized schema for validating inputs and outputs, and (2) embedding semantic annotations that allow systems to interpret the meaning of data in a context-aware manner. This dual capability enables platforms to generate UI forms for data entry and then export those inputs in a semantically structured format suitable for use by other platforms, thereby facilitating the FAIR exchange of ORD.

This solution makes it possible for different platforms to exchange workflow specifications and results through a shared language, facilitating reproducibility, automation, and standardization in the computational materials science community.

## 5. Design Principles

As stated above, the motivation for our design is to enable (1) full compatibility with the JSON-LD expansion and compaction algorithms, enabling cross-platform semantic data exchange and round-tripping between different schema contexts, and (2) compatibility with automatic UI generation and validation that rely on clean JSON Schema documents. We achieve this by defining our schemas in the OO-LD format. This allows us to leverage the strengths of both technologies while ensuring that our schemas are easy to use and understand.

The common workflow format demonstrated in this work is derived from the [AiiDA Common Workflows](https://aiida-common-workflows.readthedocs.io) (ACWF), a system of code-agnostic workflows driven by the [AiiDA workflow engine](https://aiida.net) to compute material properties including groundstate structure geomentry, Equation of State, and bond dissociation energies. These workflows have demonstrated their generality and reliability in [benchmarking studies](https://www.nature.com/articles/s41524-021-00594-6). We develop here semantically-enriched schemas for the inputs and outputs of these workflows.

The choice to start from [Pydantic](https://docs.pydantic.dev/latest/) models is motivated by their ease of use, flexibility, robustness, utility, performance, compatibility with established tooling, and familiarity within the scientific community of the Python programming language. Pydantic models are natively exportable as JSON Schemas, which serves as the structural foundation of an OO-LD document. we enrich the structural schema with semantic meaning by deriving a JSON-LD context (compatible with the [JSON-LD 1.1 specifications](https://www.w3.org/TR/json-ld11/)) from custom class- and property-level Internationalized Resource Identifiers, or IRIs for short. To focus on the ontology-enabled features rather than ontology alignment, we default our IRIs to a placeholder root namespace (`https://example.com/commonWorkflows/`), while still retaining compatability with expansion and compaction operations via JSON-LD tooling.

From a Python engineering perspective, semantic annotation behavior is centralized in a `SemanticModel` base class. IRIs are defined as private class-level attributes, and are automatically cast as `@id` entries in the model's JSON Schema via the associated metaclass. Class properties (or model fields) are annotated with a custom `MetadataField` extension of Pydantic's `Field` function, which provides a path for IRIs, units, and other attribute-level metadata.

<details>
<summary><code>SemanticModel</code></summary>

```python
from pydantic import BaseModel

class SemanticMetaclass(ModelConfigMetaclass):
    """Metaclass to add the defined IRI as an `@id` field in the JSON schema."""

    def __new__(cls, name: str, bases: tuple, _dict: dict):
        if class_iri := _dict.get("_IRI"):
            _dict["model_config"] = pdt.ConfigDict(
                json_schema_extra={
                    "@id": class_iri,
                },
            )
        return super().__new__(cls, name, bases, _dict)

class SemanticModel(BaseModel, metaclass=SemanticMetaclass):
    _IRI = ""

    def model_oo_ld(self):
        schema = self.model_json_schema()
        object_type = self.__class__.__name__
        return {
            "@context": build_context(object_type, schema),
            **schema,
            "@type": object_type,
            **serialize_model(self),
        }
```

</details>

<br>

The `model_oo_ld` method of `SemanticModel` is made available to all schema objects and provides an export path as a complete OO-LD document (JSON Schema classes (`$defs`) and `properties` + JSON-LD `@context`) derived from the annotated model using context-generation and IRI-lifting logic built specifically for this purpose. The method ensures that any subclass of `SemanticModel` can be used to automatically generate an OO-LD-compliant schema, including structural and semantic annotations.

<details>
<summary>An example OO-LD document produced for a computational <code>Code</code> object</summary>

```json
{
  "@context": {
    "@vocab": "https://example.com/",
    "ex": "https://example.com/",
    "cw": "ex:commonWorkflows/",
    "Code": "cw:Code",
    "identifier": "cw:UniqueIdentifier",
    "name": "cw:Code/Name",
    "package": "cw:Code/Package",
    "executionEnvironment": "cw:Code/ExecutionEnvironment",
    "ExecutionEnvironment": {
      "@id": "cw:ExecutionEnvironment",
      "@context": {
        "name": "cw:ExecutionEnvironment/Name",
        "metadata": "cw:ExecutionEnvironment/Metadata"
      }
    },
    "PackageManager": {
      "@id": "cw:PackageManager",
      "@context": {
        "name": "cw:PackageManager/Name",
        "metadata": "cw:PackageManager/Metadata"
      }
    },
    "Package": {
      "@id": "cw:Package",
      "@context": {
        "name": "cw:Package/Name",
        "package_manager": "cw:Package/PackageManager",
        "metadata": "cw:Package/Metadata"
      }
    }
  },
  "$defs": {
    "ExecutionEnvironment": {
      "properties": {
        "name": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The name of the execution environment.",
          "title": "Name"
        },
        "metadata": {
          "description": "The metadata of the execution environment.",
          "title": "Metadata",
          "type": "object"
        }
      },
      "required": ["metadata"],
      "title": "ExecutionEnvironment",
      "type": "object"
    },
    "Package": {
      "properties": {
        "name": {
          "description": "The name of the package.",
          "title": "Name",
          "type": "string"
        },
        "package_manager": {
          "$ref": "#/$defs/PackageManager",
          "description": "The package manager from which to obtain the package."
        },
        "metadata": {
          "description": "The metadata of the package.",
          "title": "Metadata",
          "type": "object"
        }
      },
      "required": ["name", "package_manager", "metadata"],
      "title": "Package",
      "type": "object"
    },
    "PackageManager": {
      "properties": {
        "name": {
          "description": "The name of the package manager.",
          "title": "Name",
          "type": "string"
        },
        "metadata": {
          "description": "The metadata of the package manager.",
          "title": "Metadata",
          "type": "object"
        }
      },
      "required": ["name", "metadata"],
      "title": "PackageManager",
      "type": "object"
    }
  },
  "properties": {
    "identifier": {
      "anyOf": [
        {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "format": "uuid4",
              "type": "string"
            }
          ],
          "default": null,
          "description": "Unique UUID identifier"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "The unique identifier of the code.",
      "title": "Identifier"
    },
    "name": {
      "description": "The name of the code.",
      "title": "Name",
      "type": "string"
    },
    "package": {
      "$ref": "#/$defs/Package",
      "description": "The code package."
    },
    "executionEnvironment": {
      "$ref": "#/$defs/ExecutionEnvironment",
      "description": "The execution environment of the code."
    }
  },
  "required": ["name", "package", "executionEnvironment"],
  "title": "Code",
  "type": "object",
  "@type": "Code",
  "name": "Quantum ESPRESSO",
  "package": {
    "name": "qe",
    "package_manager": {
      "name": "conda",
      "metadata": {
        "channel": "conda-forge",
        "version": "24.7.1"
      }
    },
    "metadata": {
      "version": "7.4"
    }
  },
  "executionEnvironment": {
    "name": "hpc-test",
    "metadata": {
      "hostname": "hpc",
      "description": "Test HPC machine",
      "transport_protocol": "ssh",
      "scheduler": "slurm",
      "queue": "compute",
      "architecture": "x86_64",
      "os": {
        "name": "Linux",
        "metadata": {
          "distribution": {
            "name": "RedHat",
            "version": "Enterprise 7"
          }
        }
      },
      "preinstalled": false,
      "path": "/usr/bin/pw.x"
    }
  }
}
```

</details>

<br>

The key features/benefits of this approach are:

- **Standardization**: The schema offers a standardized structure for a commonly used task in computational materials science. Pydantic provides built-in functionality to define the field, its type and description, units, and semantic meaning via an IRI
- **Extensibility**: Additional metadata or workflow steps can be added as needed without breaking compatibility. The schema classes are extensible and hierarchical, allowing extension to composite workflows.
- **UI Compatibility**: The JSON Schema component allows for automatic UI form generation, while the JSON-LD context ensures semantic consistency across platforms. The ease of UI interaction and interoperability potential of semantics emphasize the power of the OO-LD standard in common workflows.
- **Interoperability**: With OO-LD, workflow inputs and outputs can be semantically interpreted and exchanged across different simulation engines and platforms, expanding open science throughout the research community.

## 6. Leveraging JSON-LD for Interoperability

The use of JSON-LD in the OO-LD schema format facilitates semantic interoperability through structured contexts. JSON-LD allows JSON documents to carry semantic meaning by associating terms in the data with globally unique IRIs. This enables data to be interpreted consistently across platforms regardless of internal representations.

Two core operations/algorithms in JSON-LD are **expansion** and **compaction**:

- **[Expansion](https://www.w3.org/TR/json-ld11-api/#expansion)** replaces all terms in a JSON document with their full IRIs (prefix-expanded), ensuring that data is unambiguous and self-contained
- **[Compaction](https://www.w3.org/TR/json-ld11-api/#compaction)** maps the expanded IRIs to those provided in another `@context` that are semantically equivalent (referencing the same IRIs) but syntactically different (e.g., `last_name -> surname`)

By embedding a JSON-LD context directly in the schema, OO-LD documents support these transformations natively. This allows user input to be exported and interpreted by a different system using a sementically-equivalent context. The OO-LD context thus acts as a semantic bridge between different tools, platforms, and workflows, enabling meaningful data exchange without brittle or lossy transformations.

In the following example, these JSON-LD algorithms is used to map the engine-agnostic input schema onto the AiiDA-specific input schema. The mapping is trivial and for demonstration purposes only. The conversion of AiiDA outputs to the engine-agnostic schema does not leverage JSON-LD (done manually), but can in principle follow the same procedure.

## 7. Example: Common Relaxation Workflow

To illustrate the use of OO-LD schemas, we consider the example of a common relaxation workflow in computational materials science. This workflow takes as input a structure, a relaxation engine defined by a code and computational resources, and a set of calculation parameters. It outputs the relaxed structure along with relevant computed properties such as forces, stress, total energy, and magnetization.

In the selected components below, `BASE_PREFIX` refers to the ontological root of all IRIs, in this case `https://example.com/commonWorkflows`. We emphasize again that this is not a real ontology, nor are any derived from it. We further emphasize that this deliverable focuses on leveraging semantic annotations. For details on creating ontologies, please refer to our [guidelines on ontologies and semantically enriching metadata](https://github.com/ord-premise/interoperability-guidelines).

### 7.1. Schema Components

#### 7.1.1. Structure Input

The parent `RelaxInputs` schema uses the `StructureResource` model from the OPTIMADE specification to represent atomic structures. This keeps the schema engine-agnostic and ensures compatibility with widely adopted data exchange standards and databases in materials science.

<details>
<summary><code>RelaxInputs</code></summary>

```python
from optimade.models import StructureResource

class RelaxInputs(
  CommonRelaxInputs,
    WithArbitraryTypes,
):
    _IRI = f"{BASE_PREFIX}/relax/Input"

    structure: t.Annotated[
        StructureResource,
        MetadataField(
            description="The structure to relax.",
            iri=f"{BASE_PREFIX}/Structure",
        ),
    ]
```

</details>

#### 7.1.2. Common Relax Inputs

The common relax inputs are the root of the common workflows. They represent the inputs for a self-consistent force (SCF) calculation, optionally flagged for relaxation. Composite workflows computing equation of state or bond dissociation energies, for example, leverage this base calculation.

Here we define the following:

- A protocol specifying the accuracy levels as a shortcut to the complex underlying parameters
- The relaxation type specifying the degree of structure relaxation to perform, if any
- Thresholds for forces and stress (defaults defined internally from the selected protocol)
- Fields specifying the type of calculation (metallic, insulating, magnetic).
- An optional reference process (UUID) linking a previous calculation for comparison

<details>
<summary><code>CommonRelaxInputs</code></summary>

```python
class CommonRelaxInputs(SemanticModel):
    _IRI = f"{BASE_PREFIX}/relax/Input"

    engines: t.Annotated[
        dict[str, Engine],
        MetadataField(
            description="A dictionary specifying the codes and the corresponding computational resources for each step of the workflow.",
            iri=f"{_IRI}/Engines",
        ),
    ]
    protocol: t.Annotated[
        t.Literal[
            "fast",
            "moderate",
            "precise",
        ],
        MetadataField(
            description="A single string summarizing the computational accuracy of the underlying DFT calculation and relaxation algorithm. Three protocol names are defined and implemented for each code: 'fast', 'moderate' and 'precise'. The details of how each implementation translates a protocol string into a choice of parameters is code dependent, or more specifically, they depend on the implementation choices of the corresponding AiiDA plugin.",
            iri=f"{BASE_PREFIX}/scf/Protocol",
        ),
    ]
    relax_type: t.Annotated[
        t.Literal[
            "none",
            "positions",
            "volume",
            "shape",
            "cell",
            "positions_cell",
            "positions_volume",
            "positions_shape",
        ],
        MetadataField(
            description="The type of relaxation to perform, ranging from the relaxation of only atomic coordinates to the full cell relaxation for extended systems. The complete list of supported options is: 'none', 'positions', 'volume', 'shape', 'cell', 'positions_cell', 'positions_volume', 'positions_shape'. Each name indicates the physical quantities allowed to relax. For instance, 'positions_shape' corresponds to a relaxation where both the shape of the cell and the atomic coordinates are relaxed, but not the volume; in other words, this option indicates a geometric optimization at constant volume. On the other hand, the 'shape' option designates a situation when the shape of the cell is relaxed and the atomic coordinates are rescaled following the variation of the cell, not following a force minimization process. The term 'cell' is short-hand for the combination of 'shape' and 'volume'. The option 'none' indicates the possibility to calculate the total energy of the system without optimizing the structure. Not all options are available for each code. The 'none' and 'positions' options are shared by all codes.",
            iri=f"{BASE_PREFIX}/scf/RelaxType",
        ),
    ]
    threshold_forces: t.Annotated[
        t.Optional[pdt.PositiveFloat],
        MetadataField(
            description="A real positive number indicating the target threshold for the forces in eV/Å. If not specified, the protocol specification will select an appropriate value.",
            iri=f"{BASE_PREFIX}/scf/ThresholdForces",
            units="eV/Å",
        ),
    ] = None
    threshold_stress: t.Annotated[
        t.Optional[pdt.PositiveFloat],
        MetadataField(
            description="A real positive number indicating the target threshold for the stress in eV/Å^3. If not specified, the protocol specification will select an appropriate value.",
            iri=f"{BASE_PREFIX}/scf/ThresholdStress",
            units="eV/Å^3",
        ),
    ] = None
    electronic_type: t.Annotated[
        t.Optional[
            t.Literal[
                "metal",
                "insulator",
            ]
        ],
        MetadataField(
            description="An optional string to signal whether to perform the simulation for a metallic or an insulating system. It accepts only the 'insulator' and 'metal' values. This input is relevant only for calculations on extended systems. In case such option is not specified, the calculation is assumed to be metallic which is the safest assumption.",
            iri=f"{BASE_PREFIX}/scf/ElectronicType",
        ),
    ] = None
    spin_type: t.Annotated[
        t.Optional[
            t.Literal[
                "none",
                "collinear",
            ]
        ],
        MetadataField(
            description="An optional string to specify the spin degree of freedom for the calculation. It accepts the values 'none' or 'collinear'. These will be extended in the future to include, for instance, non-collinear magnetism and spin-orbit coupling. The default is to run the calculation without spin polarization.",
            iri=f"{BASE_PREFIX}/scf/SpinType",
        ),
    ] = None
    magnetization_per_site: t.Annotated[
        t.Optional[list[float]],
        MetadataField(
            description="An input devoted to the initial magnetization specifications. It accepts a list where each entry refers to an atomic site in the structure. The quantity is passed as the spin polarization in units of electrons, meaning the difference between spin up and spin down electrons for the site. This also corresponds to the magnetization of the site in Bohr magnetons (μB). The default for this input is the Python value None and, in case of calculations with spin, the None value signals that the implementation should automatically decide an appropriate default initial magnetization.",
            iri=f"{BASE_PREFIX}/Structure/Site/Magnetization",
            units="μB",
        ),
    ] = None
    reference_process: t.Annotated[
        t.Optional[UniqueIdentifier],
        MetadataField(
            description="The UUID of the process. When present, the interface returns a set of inputs which ensure that results of the new process (to be run) can be directly compared to the `reference_process`.",
            iri=f"{BASE_PREFIX}/ReferenceProcess",
        ),
    ] = None
```

</details>

#### 7.1.3. Engines and Codes

The workflow is designed to support multiple simulation codes (e.g., Quantum ESPRESSO, VASP) and decouples code execution from workflow logic. The `Code`  component represents the metadata of an executable code, including either an UUID identifier or a set of metadata sufficient for the engine to reproduce the code (future feature).

<details>
<summary><code>Code</code></summary>

```python
class Code(SemanticModel):
    _IRI: str = f"{BASE_PREFIX}/Code"

    identifier: t.Annotated[
        t.Optional[UniqueIdentifier],
        MetadataField(
            description="The unique identifier of the code.",
            iri=f"{BASE_PREFIX}/UniqueIdentifier",
        ),
    ] = None
    name: t.Annotated[
        str,
        MetadataField(
            description="The name of the code.",
            iri=f"{BASE_PREFIX}/Code/Name",
        ),
    ]
    package: t.Annotated[
        Package,
        MetadataField(
            description="The code package.",
            iri=f"{BASE_PREFIX}/Code/Package",
        ),
    ]
    executionEnvironment: t.Annotated[
        ExecutionEnvironment,
        MetadataField(
            description="The execution environment of the code.",
            iri=f"{BASE_PREFIX}/Code/ExecutionEnvironment",
        ),
    ]
```

</details>

<br>

An `Engine` object includes in addition to the associated `Code` a set of `options` defining the computational resources and remote machine metadata.

<details>
<summary><code>Engine</code></summary>

```python
class Engine(SemanticModel):
    _IRI = f"{BASE_PREFIX}/Engine"

    code: t.Annotated[
        Code,
        MetadataField(
            description="A code that can execute the engine workflow.",
        ),
    ]
    options: t.Annotated[
        dict[str, t.Any],
        MetadataField(
            description="A dictionary of metadata options for the engine, such as computational resources, parallelization, etc. These usually depend on the job scheduler of the machine on which the code is executed.",
            iri=f"{_IRI}/Options",
        ),
    ]
```

</details>

#### 7.1.4. Output

Just as with the inputs, the output schema is defined as a Pydantic model annotated with domain-relevant metadata. It encodes properties like total energy, forces, stress, and relaxed structures.

<details>
<summary><code>RelaxOutputs</code></summary>

```python
class RelaxOutputs(
    SemanticModel,
    WithArbitraryTypes,
):
    _IRI = f"{BASE_PREFIX}/relax/Output"

    forces: t.Annotated[
        FloatArray,
        MetadataField(
            description="The forces on the atoms.",
            iri=f"{BASE_PREFIX}/Forces",
            units="eV/Å",
        ),
    ]
    relaxed_structure: t.Annotated[
        t.Optional[StructureResource],
        MetadataField(
            description="The relaxed structure, if relaxation was performed.",
            iri=f"{BASE_PREFIX}/Structure",
            units="Å",
        ),
    ] = None
    total_energy: TotalEnergy
    stress: t.Annotated[
        t.Optional[FloatArray],
        MetadataField(
            description="The final stress tensor in eV/Å^3, if relaxation was performed.",
            iri=f"{BASE_PREFIX}/relax/Stress",
            units="eV/Å^3",
        ),
    ]
    total_magnetization: t.Optional[TotalMagnetization] = None
    hartree_potential: t.Annotated[
        t.Optional[FloatArray],
        MetadataField(
            description="The Hartree potential.",
            iri=f"{BASE_PREFIX}/scf/HartreePotential",
            units="Rydberg",
        ),
    ] = None
    charge_density: t.Annotated[
        t.Optional[FloatArray],
        MetadataField(
            description="The total magnetization of the system in μB.",
            iri=f"{BASE_PREFIX}/scf/ChargeDensity",
            units="Rydberg",
        ),
    ] = None
```

</details>

### 7.2. Inputs OO-LD

The OO-LD schema generated for inputs captures both structural validation and semantic annotations in a single document. This enriched input schema can be used to construct interoperable data packages, allowing inputs to be reused or adapted across engines or platforms. The semantic annotations ensure that terms like `protocol`, `relax_type`, or `spin_type` retain consistent meaning across systems, even when internal implementations vary.

<details>
<summary>An example OO-LD document for the final inputs to a common relax workflow</summary>

````json
{
  "@context": {
    "@vocab": "https://example.com/",
    "ex": "https://example.com/",
    "cw": "ex:commonWorkflows/",
    "RelaxInputs": "cw:relax/Input",
    "engines": "cw:relax/Input/Engines",
    "protocol": "cw:scf/Protocol",
    "relax_type": "cw:scf/RelaxType",
    "threshold_forces": "cw:scf/ThresholdForces",
    "threshold_stress": "cw:scf/ThresholdStress",
    "electronic_type": "cw:scf/ElectronicType",
    "spin_type": "cw:scf/SpinType",
    "magnetization_per_site": "cw:Structure/Site/Magnetization",
    "reference_process": "cw:ReferenceProcess",
    "structure": "cw:Structure",
    "PackageManager": {
      "@id": "cw:PackageManager",
      "@context": {
        "name": "cw:PackageManager/Name",
        "metadata": "cw:PackageManager/Metadata"
      }
    },
    "Package": {
      "@id": "cw:Package",
      "@context": {
        "name": "cw:Package/Name",
        "package_manager": "cw:Package/PackageManager",
        "metadata": "cw:Package/Metadata"
      }
    },
    "ExecutionEnvironment": {
      "@id": "cw:ExecutionEnvironment",
      "@context": {
        "name": "cw:ExecutionEnvironment/Name",
        "metadata": "cw:ExecutionEnvironment/Metadata"
      }
    },
    "Code": {
      "@id": "cw:Code",
      "@context": {
        "identifier": "cw:UniqueIdentifier",
        "name": "cw:Code/Name",
        "package": "cw:Code/Package",
        "executionEnvironment": "cw:Code/ExecutionEnvironment"
      }
    },
    "Engine": {
      "@id": "cw:Engine",
      "@context": {
        "options": "cw:Engine/Options"
      }
    }
  },
  "$defs": {
    "Assembly": {
      "description": "A description of groups of sites that are statistically correlated.\n\n- **Examples** (for each entry of the assemblies list):\n    - `{\"sites_in_groups\": [[0], [1]], \"group_probabilities: [0.3, 0.7]}`: the first site and the second site never occur at the same time in the unit cell.\n      Statistically, 30 % of the times the first site is present, while 70 % of the times the second site is present.\n    - `{\"sites_in_groups\": [[1,2], [3]], \"group_probabilities: [0.3, 0.7]}`: the second and third site are either present together or not present; they form the first group of atoms for this assembly.\n      The second group is formed by the fourth site. Sites of the first group (the second and the third) are never present at the same time as the fourth site.\n      30 % of times sites 1 and 2 are present (and site 3 is absent); 70 % of times site 3 is present (and sites 1 and 2 are absent).",
      "properties": {
        "sites_in_groups": {
          "description": "Index of the sites (0-based) that belong to each group for each assembly.\n\n- **Examples**:\n    - `[[1], [2]]`: two groups, one with the second site, one with the third.\n    - `[[1,2], [3]]`: one group with the second and third site, one with the fourth.",
          "items": {
            "items": {
              "type": "integer"
            },
            "type": "array"
          },
          "title": "Sites In Groups",
          "type": "array",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "must"
        },
        "group_probabilities": {
          "description": "Statistical probability of each group. It MUST have the same length as `sites_in_groups`.\nIt SHOULD sum to one.\nSee below for examples of how to specify the probability of the occurrence of a vacancy.\nThe possible reasons for the values not to sum to one are the same as already specified above for the `concentration` of each `species`.",
          "items": {
            "type": "number"
          },
          "title": "Group Probabilities",
          "type": "array",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "must"
        }
      },
      "required": ["sites_in_groups", "group_probabilities"],
      "title": "Assembly",
      "type": "object"
    },
    "BaseRelationshipMeta": {
      "additionalProperties": true,
      "description": "Specific meta field for base relationship resource",
      "properties": {
        "description": {
          "description": "OPTIONAL human-readable description of the relationship.",
          "title": "Description",
          "type": "string"
        }
      },
      "required": ["description"],
      "title": "BaseRelationshipMeta",
      "type": "object"
    },
    "BaseRelationshipResource": {
      "description": "Minimum requirements to represent a relationship resource",
      "properties": {
        "id": {
          "description": "Resource ID",
          "title": "Id",
          "type": "string"
        },
        "type": {
          "description": "Resource type",
          "title": "Type",
          "type": "string"
        },
        "meta": {
          "anyOf": [
            {
              "$ref": "#/$defs/BaseRelationshipMeta"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Relationship meta field. MUST contain 'description' if supplied."
        }
      },
      "required": ["id", "type"],
      "title": "BaseRelationshipResource",
      "type": "object"
    },
    "Code": {
      "properties": {
        "identifier": {
          "anyOf": [
            {
              "@id": "https://example.com/commonWorkflows/UUID",
              "anyOf": [
                {
                  "type": "string"
                },
                {
                  "format": "uuid4",
                  "type": "string"
                }
              ],
              "default": null,
              "description": "Unique UUID identifier"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The unique identifier of the code.",
          "title": "Identifier"
        },
        "name": {
          "description": "The name of the code.",
          "title": "Name",
          "type": "string"
        },
        "package": {
          "$ref": "#/$defs/Package",
          "description": "The code package."
        },
        "executionEnvironment": {
          "$ref": "#/$defs/ExecutionEnvironment",
          "description": "The execution environment of the code."
        }
      },
      "required": ["name", "package", "executionEnvironment"],
      "title": "Code",
      "type": "object"
    },
    "Engine": {
      "properties": {
        "code": {
          "$ref": "#/$defs/Code",
          "description": "A code that can execute the engine workflow."
        },
        "options": {
          "description": "A dictionary of metadata options for the engine, such as computational resources, parallelization, etc. These usually depend on the job scheduler of the machine on which the code is executed.",
          "title": "Options",
          "type": "object"
        }
      },
      "required": ["code", "options"],
      "title": "Engine",
      "type": "object"
    },
    "EntryRelationships": {
      "description": "This model wraps the JSON API Relationships to include type-specific top level keys.",
      "properties": {
        "references": {
          "anyOf": [
            {
              "$ref": "#/$defs/ReferenceRelationship"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Object containing links to relationships with entries of the `references` type."
        },
        "structures": {
          "anyOf": [
            {
              "$ref": "#/$defs/StructureRelationship"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Object containing links to relationships with entries of the `structures` type."
        }
      },
      "title": "EntryRelationships",
      "type": "object"
    },
    "ExecutionEnvironment": {
      "properties": {
        "name": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The name of the execution environment.",
          "title": "Name"
        },
        "metadata": {
          "description": "The metadata of the execution environment.",
          "title": "Metadata",
          "type": "object"
        }
      },
      "required": ["metadata"],
      "title": "ExecutionEnvironment",
      "type": "object"
    },
    "Link": {
      "description": "A link **MUST** be represented as either: a string containing the link's URL or a link object.",
      "properties": {
        "href": {
          "description": "a string containing the link's URL.",
          "format": "uri",
          "minLength": 1,
          "title": "Href",
          "type": "string"
        },
        "meta": {
          "anyOf": [
            {
              "$ref": "#/$defs/Meta"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a meta object containing non-standard meta-information about the link."
        }
      },
      "required": ["href"],
      "title": "Link",
      "type": "object"
    },
    "Meta": {
      "additionalProperties": true,
      "description": "Non-standard meta-information that can not be represented as an attribute or relationship.",
      "properties": {},
      "title": "Meta",
      "type": "object"
    },
    "Package": {
      "properties": {
        "name": {
          "description": "The name of the package.",
          "title": "Name",
          "type": "string"
        },
        "package_manager": {
          "$ref": "#/$defs/PackageManager",
          "description": "The package manager from which to obtain the package."
        },
        "metadata": {
          "description": "The metadata of the package.",
          "title": "Metadata",
          "type": "object"
        }
      },
      "required": ["name", "package_manager", "metadata"],
      "title": "Package",
      "type": "object"
    },
    "PackageManager": {
      "properties": {
        "name": {
          "description": "The name of the package manager.",
          "title": "Name",
          "type": "string"
        },
        "metadata": {
          "description": "The metadata of the package manager.",
          "title": "Metadata",
          "type": "object"
        }
      },
      "required": ["name", "metadata"],
      "title": "PackageManager",
      "type": "object"
    },
    "Periodicity": {
      "description": "Integer enumeration of dimension_types values",
      "enum": [0, 1],
      "title": "Periodicity",
      "type": "integer"
    },
    "ReferenceRelationship": {
      "properties": {
        "links": {
          "anyOf": [
            {
              "$ref": "#/$defs/RelationshipLinks"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a links object containing at least one of the following: self, related"
        },
        "data": {
          "anyOf": [
            {
              "$ref": "#/$defs/BaseRelationshipResource"
            },
            {
              "items": {
                "$ref": "#/$defs/BaseRelationshipResource"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Resource linkage",
          "title": "Data",
          "uniqueItems": true
        },
        "meta": {
          "anyOf": [
            {
              "$ref": "#/$defs/Meta"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a meta object that contains non-standard meta-information about the relationship."
        }
      },
      "title": "ReferenceRelationship",
      "type": "object"
    },
    "RelationshipLinks": {
      "description": "A resource object **MAY** contain references to other resource objects (\"relationships\").\nRelationships may be to-one or to-many.\nRelationships can be specified by including a member in a resource's links object.",
      "properties": {
        "self": {
          "anyOf": [
            {
              "format": "uri",
              "minLength": 1,
              "type": "string"
            },
            {
              "$ref": "#/$defs/Link"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A link for the relationship itself (a 'relationship link').\nThis link allows the client to directly manipulate the relationship.\nWhen fetched successfully, this link returns the [linkage](https://jsonapi.org/format/1.0/#document-resource-object-linkage) for the related resources as its primary data.\n(See [Fetching Relationships](https://jsonapi.org/format/1.0/#fetching-relationships).)",
          "title": "Self"
        },
        "related": {
          "anyOf": [
            {
              "format": "uri",
              "minLength": 1,
              "type": "string"
            },
            {
              "$ref": "#/$defs/Link"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A [related resource link](https://jsonapi.org/format/1.0/#document-resource-object-related-resource-links).",
          "title": "Related"
        }
      },
      "title": "RelationshipLinks",
      "type": "object"
    },
    "ResourceLinks": {
      "description": "A Resource Links object",
      "properties": {
        "self": {
          "anyOf": [
            {
              "format": "uri",
              "minLength": 1,
              "type": "string"
            },
            {
              "$ref": "#/$defs/Link"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A link that identifies the resource represented by the resource object.",
          "title": "Self"
        }
      },
      "title": "ResourceLinks",
      "type": "object"
    },
    "Species": {
      "description": "A list describing the species of the sites of this structure.\n\nSpecies can represent pure chemical elements, virtual-crystal atoms representing a\nstatistical occupation of a given site by multiple chemical elements, and/or a\nlocation to which there are attached atoms, i.e., atoms whose precise location are\nunknown beyond that they are attached to that position (frequently used to indicate\nhydrogen atoms attached to another element, e.g., a carbon with three attached\nhydrogens might represent a methyl group, -CH3).\n\n- **Examples**:\n    - `[ {\"name\": \"Ti\", \"chemical_symbols\": [\"Ti\"], \"concentration\": [1.0]} ]`: any site with this species is occupied by a Ti atom.\n    - `[ {\"name\": \"Ti\", \"chemical_symbols\": [\"Ti\", \"vacancy\"], \"concentration\": [0.9, 0.1]} ]`: any site with this species is occupied by a Ti atom with 90 % probability, and has a vacancy with 10 % probability.\n    - `[ {\"name\": \"BaCa\", \"chemical_symbols\": [\"vacancy\", \"Ba\", \"Ca\"], \"concentration\": [0.05, 0.45, 0.5], \"mass\": [0.0, 137.327, 40.078]} ]`: any site with this species is occupied by a Ba atom with 45 % probability, a Ca atom with 50 % probability, and by a vacancy with 5 % probability. The mass of this site is (on average) 88.5 a.m.u.\n    - `[ {\"name\": \"C12\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"mass\": [12.0]} ]`: any site with this species is occupied by a carbon isotope with mass 12.\n    - `[ {\"name\": \"C13\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"mass\": [13.0]} ]`: any site with this species is occupied by a carbon isotope with mass 13.\n    - `[ {\"name\": \"CH3\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"attached\": [\"H\"], \"nattached\": [3]} ]`: any site with this species is occupied by a methyl group, -CH3, which is represented without specifying precise positions of the hydrogen atoms.",
      "properties": {
        "name": {
          "description": "Gives the name of the species; the **name** value MUST be unique in the `species` list.",
          "title": "Name",
          "type": "string",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "must"
        },
        "chemical_symbols": {
          "description": "MUST be a list of strings of all chemical elements composing this species. Each item of the list MUST be one of the following:\n\n- a valid chemical-element symbol, or\n- the special value `\"X\"` to represent a non-chemical element, or\n- the special value `\"vacancy\"` to represent that this site has a non-zero probability of having a vacancy (the respective probability is indicated in the `concentration` list, see below).\n\nIf any one entry in the `species` list has a `chemical_symbols` list that is longer than 1 element, the correct flag MUST be set in the list `structure_features`.",
          "items": {
            "pattern": "(H|He|Li|Be|B|C|N|O|F|Ne|Na|Mg|Al|Si|P|S|Cl|Ar|K|Ca|Sc|Ti|V|Cr|Mn|Fe|Co|Ni|Cu|Zn|Ga|Ge|As|Se|Br|Kr|Rb|Sr|Y|Zr|Nb|Mo|Tc|Ru|Rh|Pd|Ag|Cd|In|Sn|Sb|Te|I|Xe|Cs|Ba|La|Ce|Pr|Nd|Pm|Sm|Eu|Gd|Tb|Dy|Ho|Er|Tm|Yb|Lu|Hf|Ta|W|Re|Os|Ir|Pt|Au|Hg|Tl|Pb|Bi|Po|At|Rn|Fr|Ra|Ac|Th|Pa|U|Np|Pu|Am|Cm|Bk|Cf|Es|Fm|Md|No|Lr|Rf|Db|Sg|Bh|Hs|Mt|Ds|Rg|Cn|Nh|Fl|Mc|Lv|Ts|Og|X|vacancy)",
            "type": "string"
          },
          "title": "Chemical Symbols",
          "type": "array",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "must"
        },
        "concentration": {
          "description": "MUST be a list of floats, with same length as `chemical_symbols`. The numbers represent the relative concentration of the corresponding chemical symbol in this species. The numbers SHOULD sum to one. Cases in which the numbers do not sum to one typically fall only in the following two categories:\n\n- Numerical errors when representing float numbers in fixed precision, e.g. for two chemical symbols with concentrations `1/3` and `2/3`, the concentration might look something like `[0.33333333333, 0.66666666666]`. If the client is aware that the sum is not one because of numerical precision, it can renormalize the values so that the sum is exactly one.\n- Experimental errors in the data present in the database. In this case, it is the responsibility of the client to decide how to process the data.\n\nNote that concentrations are uncorrelated between different site (even of the same species).",
          "items": {
            "type": "number"
          },
          "title": "Concentration",
          "type": "array",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "must"
        },
        "mass": {
          "anyOf": [
            {
              "items": {
                "type": "number"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "If present MUST be a list of floats expressed in a.m.u.\nElements denoting vacancies MUST have masses equal to 0.",
          "title": "Mass",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional",
          "x-optimade-unit": "a.m.u."
        },
        "original_name": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Can be any valid Unicode string, and SHOULD contain (if specified) the name of the species that is used internally in the source database.\n\nNote: With regards to \"source database\", we refer to the immediate source being queried via the OPTIMADE API implementation.",
          "title": "Original Name",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional"
        },
        "attached": {
          "anyOf": [
            {
              "items": {
                "type": "string"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "If provided MUST be a list of length 1 or more of strings of chemical symbols for the elements attached to this site, or \"X\" for a non-chemical element.",
          "title": "Attached",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional"
        },
        "nattached": {
          "anyOf": [
            {
              "items": {
                "type": "integer"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "If provided MUST be a list of length 1 or more of integers indicating the number of attached atoms of the kind specified in the value of the :field:`attached` key.",
          "title": "Nattached",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional"
        }
      },
      "required": ["name", "chemical_symbols", "concentration"],
      "title": "Species",
      "type": "object"
    },
    "StructureFeatures": {
      "description": "Enumeration of structure_features values",
      "enum": ["disorder", "implicit_atoms", "site_attachments", "assemblies"],
      "title": "StructureFeatures",
      "type": "string"
    },
    "StructureRelationship": {
      "properties": {
        "links": {
          "anyOf": [
            {
              "$ref": "#/$defs/RelationshipLinks"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a links object containing at least one of the following: self, related"
        },
        "data": {
          "anyOf": [
            {
              "$ref": "#/$defs/BaseRelationshipResource"
            },
            {
              "items": {
                "$ref": "#/$defs/BaseRelationshipResource"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Resource linkage",
          "title": "Data",
          "uniqueItems": true
        },
        "meta": {
          "anyOf": [
            {
              "$ref": "#/$defs/Meta"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a meta object that contains non-standard meta-information about the relationship."
        }
      },
      "title": "StructureRelationship",
      "type": "object"
    },
    "StructureResource": {
      "description": "Representing a structure.",
      "properties": {
        "id": {
          "description": "An entry's ID as defined in section Definition of Terms.\n\n- **Type**: string.\n\n- **Requirements/Conventions**:\n    - **Support**: MUST be supported by all implementations, MUST NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - **Response**: REQUIRED in the response.\n\n- **Examples**:\n    - `\"db/1234567\"`\n    - `\"cod/2000000\"`\n    - `\"cod/2000000@1234567\"`\n    - `\"nomad/L1234567890\"`\n    - `\"42\"`",
          "title": "Id",
          "type": "string",
          "x-optimade-queryable": "must",
          "x-optimade-support": "must"
        },
        "type": {
          "const": "structures",
          "default": "structures",
          "description": "The name of the type of an entry.\n\n- **Type**: string.\n\n- **Requirements/Conventions**:\n    - **Support**: MUST be supported by all implementations, MUST NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - **Response**: REQUIRED in the response.\n    - MUST be an existing entry type.\n    - The entry of type `<type>` and ID `<id>` MUST be returned in response to a request for `/<type>/<id>` under the versioned base URL.\n\n- **Examples**:\n    - `\"structures\"`",
          "pattern": "^structures$",
          "title": "Type",
          "type": "string",
          "x-optimade-queryable": "must",
          "x-optimade-support": "must"
        },
        "links": {
          "anyOf": [
            {
              "$ref": "#/$defs/ResourceLinks"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a links object containing links related to the resource."
        },
        "meta": {
          "anyOf": [
            {
              "$ref": "#/$defs/Meta"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a meta object containing non-standard meta-information about a resource that can not be represented as an attribute or relationship."
        },
        "attributes": {
          "$ref": "#/$defs/StructureResourceAttributes"
        },
        "relationships": {
          "anyOf": [
            {
              "$ref": "#/$defs/EntryRelationships"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A dictionary containing references to other entries according to the description in section Relationships encoded as [JSON API Relationships](https://jsonapi.org/format/1.0/#document-resource-object-relationships).\nThe OPTIONAL human-readable description of the relationship MAY be provided in the `description` field inside the `meta` dictionary of the JSON API resource identifier object."
        }
      },
      "required": ["id", "type", "attributes"],
      "title": "StructureResource",
      "type": "object"
    },
    "StructureResourceAttributes": {
      "additionalProperties": true,
      "description": "This class contains the Field for the attributes used to represent a structure, e.g. unit cell, atoms, positions.",
      "properties": {
        "immutable_id": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The entry's immutable ID (e.g., an UUID). This is important for databases having preferred IDs that point to \"the latest version\" of a record, but still offer access to older variants. This ID maps to the version-specific record, in case it changes in the future.\n\n- **Type**: string.\n\n- **Requirements/Conventions**:\n    - **Support**: OPTIONAL support in implementations, i.e., MAY be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n\n- **Examples**:\n    - `\"8bd3e750-b477-41a0-9b11-3a799f21b44f\"`\n    - `\"fjeiwoj,54;@=%<>#32\"` (Strings that are not URL-safe are allowed.)",
          "title": "Immutable Id",
          "x-optimade-queryable": "must",
          "x-optimade-support": "optional"
        },
        "last_modified": {
          "anyOf": [
            {
              "format": "date-time",
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "description": "Date and time representing when the entry was last modified.\n\n- **Type**: timestamp.\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - **Response**: REQUIRED in the response unless the query parameter `response_fields` is present and does not include this property.\n\n- **Example**:\n    - As part of JSON response format: `\"2007-04-05T14:30:20Z\"` (i.e., encoded as an [RFC 3339 Internet Date/Time Format](https://tools.ietf.org/html/rfc3339#section-5.6) string.)",
          "title": "Last Modified",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "elements": {
          "anyOf": [
            {
              "items": {
                "type": "string"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The chemical symbols of the different elements present in the structure.\n\n- **Type**: list of strings.\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - The strings are the chemical symbols, i.e., either a single uppercase letter or an uppercase letter followed by a number of lowercase letters.\n    - The order MUST be alphabetical.\n    - MUST refer to the same elements in the same order, and therefore be of the same length, as `elements_ratios`, if the latter is provided.\n    - Note: This property SHOULD NOT contain the string \"X\" to indicate non-chemical elements or \"vacancy\" to indicate vacancies (in contrast to the field `chemical_symbols` for the `species` property).\n\n- **Examples**:\n    - `[\"Si\"]`\n    - `[\"Al\",\"O\",\"Si\"]`\n\n- **Query examples**:\n    - A filter that matches all records of structures that contain Si, Al **and** O, and possibly other elements: `elements HAS ALL \"Si\", \"Al\", \"O\"`.\n    - To match structures with exactly these three elements, use `elements HAS ALL \"Si\", \"Al\", \"O\" AND elements LENGTH 3`.\n    - Note: length queries on this property can be equivalently formulated by filtering on the `nelements`_ property directly.",
          "title": "Elements",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "nelements": {
          "anyOf": [
            {
              "type": "integer"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Number of different elements in the structure as an integer.\n\n- **Type**: integer\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - MUST be equal to the lengths of the list properties `elements` and `elements_ratios`, if they are provided.\n\n- **Examples**:\n    - `3`\n\n- **Querying**:\n    - Note: queries on this property can equivalently be formulated using `elements LENGTH`.\n    - A filter that matches structures that have exactly 4 elements: `nelements=4`.\n    - A filter that matches structures that have between 2 and 7 elements: `nelements>=2 AND nelements<=7`.",
          "title": "Nelements",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "elements_ratios": {
          "anyOf": [
            {
              "items": {
                "type": "number"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Relative proportions of different elements in the structure.\n\n- **Type**: list of floats\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - Composed by the proportions of elements in the structure as a list of floating point numbers.\n    - The sum of the numbers MUST be 1.0 (within floating point accuracy)\n    - MUST refer to the same elements in the same order, and therefore be of the same length, as `elements`, if the latter is provided.\n\n- **Examples**:\n    - `[1.0]`\n    - `[0.3333333333333333, 0.2222222222222222, 0.4444444444444444]`\n\n- **Query examples**:\n    - Note: Useful filters can be formulated using the set operator syntax for correlated values.\n      However, since the values are floating point values, the use of equality comparisons is generally inadvisable.\n    - OPTIONAL: a filter that matches structures where approximately 1/3 of the atoms in the structure are the element Al is: `elements:elements_ratios HAS ALL \"Al\":>0.3333, \"Al\":<0.3334`.",
          "title": "Elements Ratios",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "chemical_formula_descriptive": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The chemical formula for a structure as a string in a form chosen by the API implementation.\n\n- **Type**: string\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - The chemical formula is given as a string consisting of properly capitalized element symbols followed by integers or decimal numbers, balanced parentheses, square, and curly brackets `(`,`)`, `[`,`]`, `{`, `}`, commas, the `+`, `-`, `:` and `=` symbols. The parentheses are allowed to be followed by a number. Spaces are allowed anywhere except within chemical symbols. The order of elements and any groupings indicated by parentheses or brackets are chosen freely by the API implementation.\n    - The string SHOULD be arithmetically consistent with the element ratios in the `chemical_formula_reduced` property.\n    - It is RECOMMENDED, but not mandatory, that symbols, parentheses and brackets, if used, are used with the meanings prescribed by [IUPAC's Nomenclature of Organic Chemistry](https://www.qmul.ac.uk/sbcs/iupac/bibliog/blue.html).\n\n- **Examples**:\n    - `\"(H2O)2 Na\"`\n    - `\"NaCl\"`\n    - `\"CaCO3\"`\n    - `\"CCaO3\"`\n    - `\"(CH3)3N+ - [CH2]2-OH = Me3N+ - CH2 - CH2OH\"`\n\n- **Query examples**:\n    - Note: the free-form nature of this property is likely to make queries on it across different databases inconsistent.\n    - A filter that matches an exactly given formula: `chemical_formula_descriptive=\"(H2O)2 Na\"`.\n    - A filter that does a partial match: `chemical_formula_descriptive CONTAINS \"H2O\"`.",
          "title": "Chemical Formula Descriptive",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "chemical_formula_reduced": {
          "anyOf": [
            {
              "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The reduced chemical formula for a structure as a string with element symbols and integer chemical proportion numbers.\nThe proportion number MUST be omitted if it is 1.\n\n- **Type**: string\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property.\n      However, support for filters using partial string matching with this property is OPTIONAL (i.e., BEGINS WITH, ENDS WITH, and CONTAINS).\n      Intricate queries on formula components are instead suggested to be formulated using set-type filter operators on the multi valued `elements` and `elements_ratios` properties.\n    - Element symbols MUST have proper capitalization (e.g., `\"Si\"`, not `\"SI\"` for \"silicon\").\n    - Elements MUST be placed in alphabetical order, followed by their integer chemical proportion number.\n    - For structures with no partial occupation, the chemical proportion numbers are the smallest integers for which the chemical proportion is exactly correct.\n    - For structures with partial occupation, the chemical proportion numbers are integers that within reasonable approximation indicate the correct chemical proportions. The precise details of how to perform the rounding is chosen by the API implementation.\n    - No spaces or separators are allowed.\n\n- **Examples**:\n    - `\"H2NaO\"`\n    - `\"ClNa\"`\n    - `\"CCaO3\"`\n\n- **Query examples**:\n    - A filter that matches an exactly given formula is `chemical_formula_reduced=\"H2NaO\"`.",
          "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
          "title": "Chemical Formula Reduced",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "chemical_formula_hill": {
          "anyOf": [
            {
              "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The chemical formula for a structure in [Hill form](https://dx.doi.org/10.1021/ja02046a005) with element symbols followed by integer chemical proportion numbers. The proportion number MUST be omitted if it is 1.\n\n- **Type**: string\n\n- **Requirements/Conventions**:\n    - **Support**: OPTIONAL support in implementations, i.e., MAY be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n      If supported, only a subset of the filter features MAY be supported.\n    - The overall scale factor of the chemical proportions is chosen such that the resulting values are integers that indicate the most chemically relevant unit of which the system is composed.\n      For example, if the structure is a repeating unit cell with four hydrogens and four oxygens that represents two hydroperoxide molecules, `chemical_formula_hill` is `\"H2O2\"` (i.e., not `\"HO\"`, nor `\"H4O4\"`).\n    - If the chemical insight needed to ascribe a Hill formula to the system is not present, the property MUST be handled as unset.\n    - Element symbols MUST have proper capitalization (e.g., `\"Si\"`, not `\"SI\"` for \"silicon\").\n    - Elements MUST be placed in [Hill order](https://dx.doi.org/10.1021/ja02046a005), followed by their integer chemical proportion number.\n      Hill order means: if carbon is present, it is placed first, and if also present, hydrogen is placed second.\n      After that, all other elements are ordered alphabetically.\n      If carbon is not present, all elements are ordered alphabetically.\n    - If the system has sites with partial occupation and the total occupations of each element do not all sum up to integers, then the Hill formula SHOULD be handled as unset.\n    - No spaces or separators are allowed.\n\n- **Examples**:\n    - `\"H2O2\"`\n\n- **Query examples**:\n    - A filter that matches an exactly given formula is `chemical_formula_hill=\"H2O2\"`.",
          "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
          "title": "Chemical Formula Hill",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional"
        },
        "chemical_formula_anonymous": {
          "anyOf": [
            {
              "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The anonymous formula is the `chemical_formula_reduced`, but where the elements are instead first ordered by their chemical proportion number, and then, in order left to right, replaced by anonymous symbols A, B, C, ..., Z, Aa, Ba, ..., Za, Ab, Bb, ... and so on.\n\n- **Type**: string\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property.\n      However, support for filters using partial string matching with this property is OPTIONAL (i.e., BEGINS WITH, ENDS WITH, and CONTAINS).\n\n- **Examples**:\n    - `\"A2B\"`\n    - `\"A42B42C16D12E10F9G5\"`\n\n- **Querying**:\n    - A filter that matches an exactly given formula is `chemical_formula_anonymous=\"A2B\"`.",
          "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
          "title": "Chemical Formula Anonymous",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "dimension_types": {
          "anyOf": [
            {
              "items": {
                "$ref": "#/$defs/Periodicity"
              },
              "maxItems": 3,
              "minItems": 3,
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "List of three integers.\nFor each of the three directions indicated by the three lattice vectors (see property `lattice_vectors`), this list indicates if the direction is periodic (value `1`) or non-periodic (value `0`).\nNote: the elements in this list each refer to the direction of the corresponding entry in `lattice_vectors` and *not* the Cartesian x, y, z directions.\n\n- **Type**: list of integers.\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n    - MUST be a list of length 3.\n    - Each integer element MUST assume only the value 0 or 1.\n\n- **Examples**:\n    - For a molecule: `[0, 0, 0]`\n    - For a wire along the direction specified by the third lattice vector: `[0, 0, 1]`\n    - For a 2D surface/slab, periodic on the plane defined by the first and third lattice vectors: `[1, 0, 1]`\n    - For a bulk 3D system: `[1, 1, 1]`",
          "title": "Dimension Types",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "should"
        },
        "nperiodic_dimensions": {
          "anyOf": [
            {
              "type": "integer"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "An integer specifying the number of periodic dimensions in the structure, equivalent to the number of non-zero entries in `dimension_types`.\n\n- **Type**: integer\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - The integer value MUST be between 0 and 3 inclusive and MUST be equal to the sum of the items in the `dimension_types` property.\n    - This property only reflects the treatment of the lattice vectors provided for the structure, and not any physical interpretation of the dimensionality of its contents.\n\n- **Examples**:\n    - `2` should be indicated in cases where `dimension_types` is any of `[1, 1, 0]`, `[1, 0, 1]`, `[0, 1, 1]`.\n\n- **Query examples**:\n    - Match only structures with exactly 3 periodic dimensions: `nperiodic_dimensions=3`\n    - Match all structures with 2 or fewer periodic dimensions: `nperiodic_dimensions<=2`",
          "title": "Nperiodic Dimensions",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "lattice_vectors": {
          "anyOf": [
            {
              "items": {
                "items": {
                  "anyOf": [
                    {
                      "type": "number"
                    },
                    {
                      "type": "null"
                    }
                  ]
                },
                "maxItems": 3,
                "minItems": 3,
                "type": "array"
              },
              "maxItems": 3,
              "minItems": 3,
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The three lattice vectors in Cartesian coordinates, in \u00e5ngstr\u00f6m (\u00c5).\n\n- **Type**: list of list of floats or unknown values.\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n      If supported, filters MAY support only a subset of comparison operators.\n    - MUST be a list of three vectors *a*, *b*, and *c*, where each of the vectors MUST BE a list of the vector's coordinates along the x, y, and z Cartesian coordinates.\n      (Therefore, the first index runs over the three lattice vectors and the second index runs over the x, y, z Cartesian coordinates).\n    - For databases that do not define an absolute Cartesian system (e.g., only defining the length and angles between vectors), the first lattice vector SHOULD be set along *x* and the second on the *xy*-plane.\n    - MUST always contain three vectors of three coordinates each, independently of the elements of property `dimension_types`.\n      The vectors SHOULD by convention be chosen so the determinant of the `lattice_vectors` matrix is different from zero.\n      The vectors in the non-periodic directions have no significance beyond fulfilling these requirements.\n    - The coordinates of the lattice vectors of non-periodic dimensions (i.e., those dimensions for which `dimension_types` is `0`) MAY be given as a list of all `null` values.\n        If a lattice vector contains the value `null`, all coordinates of that lattice vector MUST be `null`.\n\n- **Examples**:\n    - `[[4.0,0.0,0.0],[0.0,4.0,0.0],[0.0,1.0,4.0]]` represents a cell, where the first vector is `(4, 0, 0)`, i.e., a vector aligned along the `x` axis of length 4 \u00c5; the second vector is `(0, 4, 0)`; and the third vector is `(0, 1, 4)`.",
          "title": "Lattice Vectors",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "should",
          "x-optimade-unit": "\u00c5"
        },
        "cartesian_site_positions": {
          "anyOf": [
            {
              "items": {
                "items": {
                  "type": "number"
                },
                "maxItems": 3,
                "minItems": 3,
                "type": "array"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Cartesian positions of each site in the structure.\nA site is usually used to describe positions of atoms; what atoms can be encountered at a given site is conveyed by the `species_at_sites` property, and the species themselves are described in the `species` property.\n\n- **Type**: list of list of floats\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n      If supported, filters MAY support only a subset of comparison operators.\n    - It MUST be a list of length equal to the number of sites in the structure, where every element is a list of the three Cartesian coordinates of a site expressed as float values in the unit angstrom (\u00c5).\n    - An entry MAY have multiple sites at the same Cartesian position (for a relevant use of this, see e.g., the property `assemblies`).\n\n- **Examples**:\n    - `[[0,0,0],[0,0,2]]` indicates a structure with two sites, one sitting at the origin and one along the (positive) *z*-axis, 2 \u00c5 away from the origin.",
          "title": "Cartesian Site Positions",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "should",
          "x-optimade-unit": "\u00c5"
        },
        "nsites": {
          "anyOf": [
            {
              "type": "integer"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "An integer specifying the length of the `cartesian_site_positions` property.\n\n- **Type**: integer\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n\n- **Examples**:\n    - `42`\n\n- **Query examples**:\n    - Match only structures with exactly 4 sites: `nsites=4`\n    - Match structures that have between 2 and 7 sites: `nsites>=2 AND nsites<=7`",
          "title": "Nsites",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "species": {
          "anyOf": [
            {
              "items": {
                "$ref": "#/$defs/Species"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A list describing the species of the sites of this structure.\nSpecies can represent pure chemical elements, virtual-crystal atoms representing a statistical occupation of a given site by multiple chemical elements, and/or a location to which there are attached atoms, i.e., atoms whose precise location are unknown beyond that they are attached to that position (frequently used to indicate hydrogen atoms attached to another element, e.g., a carbon with three attached hydrogens might represent a methyl group, -CH3).\n\n- **Type**: list of dictionary with keys:\n    - `name`: string (REQUIRED)\n    - `chemical_symbols`: list of strings (REQUIRED)\n    - `concentration`: list of float (REQUIRED)\n    - `attached`: list of strings (REQUIRED)\n    - `nattached`: list of integers (OPTIONAL)\n    - `mass`: list of floats (OPTIONAL)\n    - `original_name`: string (OPTIONAL).\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n        If supported, filters MAY support only a subset of comparison operators.\n    - Each list member MUST be a dictionary with the following keys:\n        - **name**: REQUIRED; gives the name of the species; the **name** value MUST be unique in the `species` list;\n        - **chemical_symbols**: REQUIRED; MUST be a list of strings of all chemical elements composing this species.\n          Each item of the list MUST be one of the following:\n            - a valid chemical-element symbol, or\n            - the special value `\"X\"` to represent a non-chemical element, or\n            - the special value `\"vacancy\"` to represent that this site has a non-zero probability of having a vacancy (the respective probability is indicated in the `concentration` list, see below).\n\n          If any one entry in the `species` list has a `chemical_symbols` list that is longer than 1 element, the correct flag MUST be set in the list `structure_features`.\n\n        - **concentration**: REQUIRED; MUST be a list of floats, with same length as `chemical_symbols`.\n          The numbers represent the relative concentration of the corresponding chemical symbol in this species.\n          The numbers SHOULD sum to one. Cases in which the numbers do not sum to one typically fall only in the following two categories:\n\n            - Numerical errors when representing float numbers in fixed precision, e.g. for two chemical symbols with concentrations `1/3` and `2/3`, the concentration might look something like `[0.33333333333, 0.66666666666]`. If the client is aware that the sum is not one because of numerical precision, it can renormalize the values so that the sum is exactly one.\n            - Experimental errors in the data present in the database. In this case, it is the responsibility of the client to decide how to process the data.\n\n            Note that concentrations are uncorrelated between different sites (even of the same species).\n\n        - **attached**: OPTIONAL; if provided MUST be a list of length 1 or more of strings of chemical symbols for the elements attached to this site, or \"X\" for a non-chemical element.\n\n        - **nattached**: OPTIONAL; if provided MUST be a list of length 1 or more of integers indicating the number of attached atoms of the kind specified in the value of the `attached` key.\n\n          The implementation MUST include either both or none of the `attached` and `nattached` keys, and if they are provided, they MUST be of the same length.\n          Furthermore, if they are provided, the `structure_features` property MUST include the string `site_attachments`.\n\n        - **mass**: OPTIONAL. If present MUST be a list of floats, with the same length as `chemical_symbols`, providing element masses expressed in a.m.u.\n          Elements denoting vacancies MUST have masses equal to 0.\n\n        - **original_name**: OPTIONAL. Can be any valid Unicode string, and SHOULD contain (if specified) the name of the species that is used internally in the source database.\n\n          Note: With regards to \"source database\", we refer to the immediate source being queried via the OPTIMADE API implementation.\n\n          The main use of this field is for source databases that use species names, containing characters that are not allowed (see description of the list property `species_at_sites`).\n\n    - For systems that have only species formed by a single chemical symbol, and that have at most one species per chemical symbol, SHOULD use the chemical symbol as species name (e.g., `\"Ti\"` for titanium, `\"O\"` for oxygen, etc.)\n      However, note that this is OPTIONAL, and client implementations MUST NOT assume that the key corresponds to a chemical symbol, nor assume that if the species name is a valid chemical symbol, that it represents a species with that chemical symbol.\n      This means that a species `{\"name\": \"C\", \"chemical_symbols\": [\"Ti\"], \"concentration\": [1.0]}` is valid and represents a titanium species (and *not* a carbon species).\n    - It is NOT RECOMMENDED that a structure includes species that do not have at least one corresponding site.\n\n- **Examples**:\n    - `[ {\"name\": \"Ti\", \"chemical_symbols\": [\"Ti\"], \"concentration\": [1.0]} ]`: any site with this species is occupied by a Ti atom.\n    - `[ {\"name\": \"Ti\", \"chemical_symbols\": [\"Ti\", \"vacancy\"], \"concentration\": [0.9, 0.1]} ]`: any site with this species is occupied by a Ti atom with 90 % probability, and has a vacancy with 10 % probability.\n    - `[ {\"name\": \"BaCa\", \"chemical_symbols\": [\"vacancy\", \"Ba\", \"Ca\"], \"concentration\": [0.05, 0.45, 0.5], \"mass\": [0.0, 137.327, 40.078]} ]`: any site with this species is occupied by a Ba atom with 45 % probability, a Ca atom with 50 % probability, and by a vacancy with 5 % probability. The mass of this site is (on average) 88.5 a.m.u.\n    - `[ {\"name\": \"C12\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"mass\": [12.0]} ]`: any site with this species is occupied by a carbon isotope with mass 12.\n    - `[ {\"name\": \"C13\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"mass\": [13.0]} ]`: any site with this species is occupied by a carbon isotope with mass 13.\n    - `[ {\"name\": \"CH3\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"attached\": [\"H\"], \"nattached\": [3]} ]`: any site with this species is occupied by a methyl group, -CH3, which is represented without specifying precise positions of the hydrogen atoms.",
          "title": "Species",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "should"
        },
        "species_at_sites": {
          "anyOf": [
            {
              "items": {
                "type": "string"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Name of the species at each site (where values for sites are specified with the same order of the property `cartesian_site_positions`).\nThe properties of the species are found in the property `species`.\n\n- **Type**: list of strings.\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n      If supported, filters MAY support only a subset of comparison operators.\n    - MUST have length equal to the number of sites in the structure (first dimension of the list property `cartesian_site_positions`).\n    - Each species name mentioned in the `species_at_sites` list MUST be described in the list property `species` (i.e. for each value in the `species_at_sites` list there MUST exist exactly one dictionary in the `species` list with the `name` attribute equal to the corresponding `species_at_sites` value).\n    - Each site MUST be associated only to a single species.\n      **Note**: However, species can represent mixtures of atoms, and multiple species MAY be defined for the same chemical element.\n      This latter case is useful when different atoms of the same type need to be grouped or distinguished, for instance in simulation codes to assign different initial spin states.\n\n- **Examples**:\n    - `[\"Ti\",\"O2\"]` indicates that the first site is hosting a species labeled `\"Ti\"` and the second a species labeled `\"O2\"`.\n    - `[\"Ac\", \"Ac\", \"Ag\", \"Ir\"]` indicating the first two sites contains the `\"Ac\"` species, while the third and fourth sites contain the `\"Ag\"` and `\"Ir\"` species, respectively.",
          "title": "Species At Sites",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "should"
        },
        "assemblies": {
          "anyOf": [
            {
              "items": {
                "$ref": "#/$defs/Assembly"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A description of groups of sites that are statistically correlated.\n\n- **Type**: list of dictionary with keys:\n    - `sites_in_groups`: list of list of integers (REQUIRED)\n    - `group_probabilities`: list of floats (REQUIRED)\n\n- **Requirements/Conventions**:\n    - **Support**: OPTIONAL support in implementations, i.e., MAY be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n        If supported, filters MAY support only a subset of comparison operators.\n    - The property SHOULD be `null` for entries that have no partial occupancies.\n    - If present, the correct flag MUST be set in the list `structure_features`.\n    - Client implementations MUST check its presence (as its presence changes the interpretation of the structure).\n    - If present, it MUST be a list of dictionaries, each of which represents an assembly and MUST have the following two keys:\n        - **sites_in_groups**: Index of the sites (0-based) that belong to each group for each assembly.\n\n            Example: `[[1], [2]]`: two groups, one with the second site, one with the third.\n            Example: `[[1,2], [3]]`: one group with the second and third site, one with the fourth.\n\n        - **group_probabilities**: Statistical probability of each group. It MUST have the same length as `sites_in_groups`.\n            It SHOULD sum to one.\n            See below for examples of how to specify the probability of the occurrence of a vacancy.\n            The possible reasons for the values not to sum to one are the same as already specified above for the `concentration` of each `species`.\n\n    - If a site is not present in any group, it means that it is present with 100 % probability (as if no assembly was specified).\n    - A site MUST NOT appear in more than one group.\n\n- **Examples** (for each entry of the assemblies list):\n    - `{\"sites_in_groups\": [[0], [1]], \"group_probabilities: [0.3, 0.7]}`: the first site and the second site never occur at the same time in the unit cell.\n        Statistically, 30 % of the times the first site is present, while 70 % of the times the second site is present.\n    - `{\"sites_in_groups\": [[1,2], [3]], \"group_probabilities: [0.3, 0.7]}`: the second and third site are either present together or not present; they form the first group of atoms for this assembly.\n        The second group is formed by the fourth site.\n        Sites of the first group (the second and the third) are never present at the same time as the fourth site.\n        30 % of times sites 1 and 2 are present (and site 3 is absent); 70 % of times site 3 is present (and sites 1 and 2 are absent).\n\n- **Notes**:\n    - Assemblies are essential to represent, for instance, the situation where an atom can statistically occupy two different positions (sites).\n\n    - By defining groups, it is possible to represent, e.g., the case where a functional molecule (and not just one atom) is either present or absent (or the case where it it is present in two conformations)\n\n    - Considerations on virtual alloys and on vacancies: In the special case of a virtual alloy, these specifications allow two different, equivalent ways of specifying them.\n        For instance, for a site at the origin with 30 % probability of being occupied by Si, 50 % probability of being occupied by Ge, and 20 % of being a vacancy, the following two representations are possible:\n\n        - Using a single species:\n            ```json\n            {\n              \"cartesian_site_positions\": [[0,0,0]],\n              \"species_at_sites\": [\"SiGe-vac\"],\n              \"species\": [\n              {\n                \"name\": \"SiGe-vac\",\n                \"chemical_symbols\": [\"Si\", \"Ge\", \"vacancy\"],\n                \"concentration\": [0.3, 0.5, 0.2]\n              }\n              ]\n              // ...\n            }\n            ```\n\n        - Using multiple species and the assemblies:\n            ```json\n            {\n              \"cartesian_site_positions\": [ [0,0,0], [0,0,0], [0,0,0] ],\n              \"species_at_sites\": [\"Si\", \"Ge\", \"vac\"],\n              \"species\": [\n                { \"name\": \"Si\", \"chemical_symbols\": [\"Si\"], \"concentration\": [1.0] },\n                { \"name\": \"Ge\", \"chemical_symbols\": [\"Ge\"], \"concentration\": [1.0] },\n                { \"name\": \"vac\", \"chemical_symbols\": [\"vacancy\"], \"concentration\": [1.0] }\n              ],\n              \"assemblies\": [\n                {\n              \"sites_in_groups\": [ [0], [1], [2] ],\n              \"group_probabilities\": [0.3, 0.5, 0.2]\n                }\n              ]\n              // ...\n            }\n            ```\n\n    - It is up to the database provider to decide which representation to use, typically depending on the internal format in which the structure is stored.\n        However, given a structure identified by a unique ID, the API implementation MUST always provide the same representation for it.\n\n    - The probabilities of occurrence of different assemblies are uncorrelated.\n        So, for instance in the following case with two assemblies:\n        ```json\n        {\n          \"assemblies\": [\n            {\n              \"sites_in_groups\": [ [0], [1] ],\n              \"group_probabilities\": [0.2, 0.8],\n            },\n            {\n              \"sites_in_groups\": [ [2], [3] ],\n              \"group_probabilities\": [0.3, 0.7]\n            }\n          ]\n        }\n        ```\n\n        Site 0 is present with a probability of 20 % and site 1 with a probability of 80 %. These two sites are correlated (either site 0 or 1 is present). Similarly, site 2 is present with a probability of 30 % and site 3 with a probability of 70 %.\n        These two sites are correlated (either site 2 or 3 is present).\n        However, the presence or absence of sites 0 and 1 is not correlated with the presence or absence of sites 2 and 3 (in the specific example, the pair of sites (0, 2) can occur with 0.2*0.3 = 6 % probability; the pair (0, 3) with 0.2*0.7 = 14 % probability; the pair (1, 2) with 0.8*0.3 = 24 % probability; and the pair (1, 3) with 0.8*0.7 = 56 % probability).",
          "title": "Assemblies",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional"
        },
        "structure_features": {
          "description": "A list of strings that flag which special features are used by the structure.\n\n- **Type**: list of strings\n\n- **Requirements/Conventions**:\n    - **Support**: MUST be supported by all implementations, MUST NOT be `null`.\n    - **Query**: MUST be a queryable property.\n    Filters on the list MUST support all mandatory HAS-type queries.\n    Filter operators for comparisons on the string components MUST support equality, support for other comparison operators are OPTIONAL.\n    - MUST be an empty list if no special features are used.\n    - MUST be sorted alphabetically.\n    - If a special feature listed below is used, the list MUST contain the corresponding string.\n    - If a special feature listed below is not used, the list MUST NOT contain the corresponding string.\n    - **List of strings used to indicate special structure features**:\n        - `disorder`: this flag MUST be present if any one entry in the `species` list has a `chemical_symbols` list that is longer than 1 element.\n        - `implicit_atoms`: this flag MUST be present if the structure contains atoms that are not assigned to sites via the property `species_at_sites` (e.g., because their positions are unknown).\n           When this flag is present, the properties related to the chemical formula will likely not match the type and count of atoms represented by the `species_at_sites`, `species` and `assemblies` properties.\n        - `site_attachments`: this flag MUST be present if any one entry in the `species` list includes `attached` and `nattached`.\n        - `assemblies`: this flag MUST be present if the property `assemblies` is present.\n\n- **Examples**: A structure having implicit atoms and using assemblies: `[\"assemblies\", \"implicit_atoms\"]`",
          "items": {
            "$ref": "#/$defs/StructureFeatures"
          },
          "title": "Structure Features",
          "type": "array",
          "x-optimade-queryable": "must",
          "x-optimade-support": "must"
        }
      },
      "required": ["last_modified", "structure_features"],
      "title": "StructureResourceAttributes",
      "type": "object"
    }
  },
  "properties": {
    "engines": {
      "additionalProperties": {
        "$ref": "#/$defs/Engine"
      },
      "description": "A dictionary specifying the codes and the corresponding computational resources for each step of the workflow.",
      "title": "Engines",
      "type": "object"
    },
    "protocol": {
      "description": "A single string summarizing the computational accuracy of the underlying DFT calculation and relaxation algorithm. Three protocol names are defined and implemented for each code: 'fast', 'moderate' and 'precise'. The details of how each implementation translates a protocol string into a choice of parameters is code dependent, or more specifically, they depend on the implementation choices of the corresponding AiiDA plugin.",
      "enum": ["fast", "moderate", "precise"],
      "title": "Protocol",
      "type": "string"
    },
    "relax_type": {
      "description": "The type of relaxation to perform, ranging from the relaxation of only atomic coordinates to the full cell relaxation for extended systems. The complete list of supported options is: 'none', 'positions', 'volume', 'shape', 'cell', 'positions_cell', 'positions_volume', 'positions_shape'. Each name indicates the physical quantities allowed to relax. For instance, 'positions_shape' corresponds to a relaxation where both the shape of the cell and the atomic coordinates are relaxed, but not the volume; in other words, this option indicates a geometric optimization at constant volume. On the other hand, the 'shape' option designates a situation when the shape of the cell is relaxed and the atomic coordinates are rescaled following the variation of the cell, not following a force minimization process. The term 'cell' is short-hand for the combination of 'shape' and 'volume'. The option 'none' indicates the possibility to calculate the total energy of the system without optimizing the structure. Not all options are available for each code. The 'none' and 'positions' options are shared by all codes.",
      "enum": [
        "none",
        "positions",
        "volume",
        "shape",
        "cell",
        "positions_cell",
        "positions_volume",
        "positions_shape"
      ],
      "title": "Relax Type",
      "type": "string"
    },
    "threshold_forces": {
      "anyOf": [
        {
          "exclusiveMinimum": 0.0,
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "A real positive number indicating the target threshold for the forces in eV/\u00c5. If not specified, the protocol specification will select an appropriate value.",
      "title": "Threshold Forces",
      "units": "eV/\u00c5"
    },
    "threshold_stress": {
      "anyOf": [
        {
          "exclusiveMinimum": 0.0,
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "A real positive number indicating the target threshold for the stress in eV/\u00c5^3. If not specified, the protocol specification will select an appropriate value.",
      "title": "Threshold Stress",
      "units": "eV/\u00c5^3"
    },
    "electronic_type": {
      "anyOf": [
        {
          "enum": ["metal", "insulator"],
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "An optional string to signal whether to perform the simulation for a metallic or an insulating system. It accepts only the 'insulator' and 'metal' values. This input is relevant only for calculations on extended systems. In case such option is not specified, the calculation is assumed to be metallic which is the safest assumption.",
      "title": "Electronic Type"
    },
    "spin_type": {
      "anyOf": [
        {
          "enum": ["none", "collinear"],
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "An optional string to specify the spin degree of freedom for the calculation. It accepts the values 'none' or 'collinear'. These will be extended in the future to include, for instance, non-collinear magnetism and spin-orbit coupling. The default is to run the calculation without spin polarization.",
      "title": "Spin Type"
    },
    "magnetization_per_site": {
      "anyOf": [
        {
          "items": {
            "type": "number"
          },
          "type": "array"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "An input devoted to the initial magnetization specifications. It accepts a list where each entry refers to an atomic site in the structure. The quantity is passed as the spin polarization in units of electrons, meaning the difference between spin up and spin down electrons for the site. This also corresponds to the magnetization of the site in Bohr magnetons (\u03bcB). The default for this input is the Python value None and, in case of calculations with spin, the None value signals that the implementation should automatically decide an appropriate default initial magnetization.",
      "title": "Magnetization Per Site",
      "units": "\u03bcB"
    },
    "reference_process": {
      "anyOf": [
        {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "format": "uuid4",
              "type": "string"
            }
          ],
          "default": null,
          "description": "Unique UUID identifier"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "The UUID of the process. When present, the interface returns a set of inputs which ensure that results of the new process (to be run) can be directly compared to the `reference_process`.",
      "title": "Reference Process"
    },
    "structure": {
      "$ref": "#/$defs/StructureResource",
      "description": "The structure to relax."
    }
  },
  "required": ["engines", "protocol", "relax_type", "structure"],
  "title": "RelaxInputs",
  "type": "object",
  "@type": "RelaxInputs",
  "engines": {
    "relax": {
      "code": {
        "identifier": "07f316b1-5403-40eb-b4dc-6be4a529ce67",
        "name": "Quantum ESPRESSO",
        "package": {
          "name": "qe",
          "package_manager": {
            "name": "conda",
            "metadata": {
              "channel": "conda-forge",
              "version": "24.7.1"
            }
          },
          "metadata": {
            "version": "7.4"
          }
        },
        "executionEnvironment": {
          "name": "hpc-test",
          "metadata": {
            "hostname": "hpc",
            "description": "Test HPC machine",
            "transport_protocol": "ssh",
            "scheduler": "slurm",
            "queue": "compute",
            "architecture": "x86_64",
            "os": {
              "name": "Linux",
              "metadata": {
                "distribution": {
                  "name": "RedHat",
                  "version": "Enterprise 7"
                }
              }
            },
            "preinstalled": false,
            "path": "/usr/bin/pw.x"
          }
        }
      },
      "options": {
        "resources": {
          "num_machines": 1
        }
      }
    }
  },
  "protocol": "fast",
  "relax_type": "positions_cell",
  "threshold_forces": null,
  "threshold_stress": null,
  "electronic_type": null,
  "spin_type": null,
  "magnetization_per_site": null,
  "reference_process": "e03d0b01-5ab4-4628-8049-ca2b940ce19a",
  "structure": {
    "id": "138723",
    "type": "structures",
    "links": null,
    "meta": null,
    "attributes": {
      "immutable_id": "85cbb651-21fa-47a0-831d-3eeaffa4d46c",
      "last_modified": "2024-11-12T12:42:42+00:00",
      "elements": ["Si"],
      "nelements": 1,
      "elements_ratios": [1.0],
      "chemical_formula_descriptive": "Si",
      "chemical_formula_reduced": "Si",
      "chemical_formula_hill": "Si",
      "chemical_formula_anonymous": "A",
      "dimension_types": [1, 1, 1],
      "nperiodic_dimensions": 3,
      "lattice_vectors": [
        [0.0, 1.934867891363, 1.934867891363],
        [1.934867891363, 0.0, 1.934867891363],
        [1.934867891363, 1.934867891363, 0.0]
      ],
      "cartesian_site_positions": [[0.0, 0.0, 0.0]],
      "nsites": 1,
      "species": [
        {
          "name": "Si",
          "chemical_symbols": ["Si"],
          "concentration": [1.0],
          "mass": [28.0855],
          "original_name": "Si",
          "attached": null,
          "nattached": null
        }
      ],
      "species_at_sites": ["Si"],
      "assemblies": null,
      "structure_features": [],
      "_mcloud_ctime": "2019-08-17T14:08:31Z"
    },
    "relationships": null
  }
}
````

</details>

### 7.3. Output OO-LD

The semantic layer in the OO-LD output document ensures that these results can be interpreted meaningfully across data consumers—such as search services, benchmarking tools, or downstream workflows.

<details>
<summary>An example OO-LD document for the output of a common relax workflow</summary>

````json
{
  "@context": {
    "@vocab": "https://example.com/",
    "ex": "https://example.com/",
    "cw": "ex:commonWorkflows/",
    "RelaxOutputs": "cw:relax/Output",
    "forces": "cw:Forces",
    "relaxed_structure": "cw:Structure",
    "total_energy": "cw:scf/TotalEnergy",
    "stress": "cw:relax/Stress",
    "hartree_potential": "cw:scf/HartreePotential",
    "charge_density": "cw:scf/ChargeDensity"
  },
  "$defs": {
    "Assembly": {
      "description": "A description of groups of sites that are statistically correlated.\n\n- **Examples** (for each entry of the assemblies list):\n    - `{\"sites_in_groups\": [[0], [1]], \"group_probabilities: [0.3, 0.7]}`: the first site and the second site never occur at the same time in the unit cell.\n      Statistically, 30 % of the times the first site is present, while 70 % of the times the second site is present.\n    - `{\"sites_in_groups\": [[1,2], [3]], \"group_probabilities: [0.3, 0.7]}`: the second and third site are either present together or not present; they form the first group of atoms for this assembly.\n      The second group is formed by the fourth site. Sites of the first group (the second and the third) are never present at the same time as the fourth site.\n      30 % of times sites 1 and 2 are present (and site 3 is absent); 70 % of times site 3 is present (and sites 1 and 2 are absent).",
      "properties": {
        "sites_in_groups": {
          "description": "Index of the sites (0-based) that belong to each group for each assembly.\n\n- **Examples**:\n    - `[[1], [2]]`: two groups, one with the second site, one with the third.\n    - `[[1,2], [3]]`: one group with the second and third site, one with the fourth.",
          "items": {
            "items": {
              "type": "integer"
            },
            "type": "array"
          },
          "title": "Sites In Groups",
          "type": "array",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "must"
        },
        "group_probabilities": {
          "description": "Statistical probability of each group. It MUST have the same length as `sites_in_groups`.\nIt SHOULD sum to one.\nSee below for examples of how to specify the probability of the occurrence of a vacancy.\nThe possible reasons for the values not to sum to one are the same as already specified above for the `concentration` of each `species`.",
          "items": {
            "type": "number"
          },
          "title": "Group Probabilities",
          "type": "array",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "must"
        }
      },
      "required": ["sites_in_groups", "group_probabilities"],
      "title": "Assembly",
      "type": "object"
    },
    "BaseRelationshipMeta": {
      "additionalProperties": true,
      "description": "Specific meta field for base relationship resource",
      "properties": {
        "description": {
          "description": "OPTIONAL human-readable description of the relationship.",
          "title": "Description",
          "type": "string"
        }
      },
      "required": ["description"],
      "title": "BaseRelationshipMeta",
      "type": "object"
    },
    "BaseRelationshipResource": {
      "description": "Minimum requirements to represent a relationship resource",
      "properties": {
        "id": {
          "description": "Resource ID",
          "title": "Id",
          "type": "string"
        },
        "type": {
          "description": "Resource type",
          "title": "Type",
          "type": "string"
        },
        "meta": {
          "anyOf": [
            {
              "$ref": "#/$defs/BaseRelationshipMeta"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Relationship meta field. MUST contain 'description' if supplied."
        }
      },
      "required": ["id", "type"],
      "title": "BaseRelationshipResource",
      "type": "object"
    },
    "EntryRelationships": {
      "description": "This model wraps the JSON API Relationships to include type-specific top level keys.",
      "properties": {
        "references": {
          "anyOf": [
            {
              "$ref": "#/$defs/ReferenceRelationship"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Object containing links to relationships with entries of the `references` type."
        },
        "structures": {
          "anyOf": [
            {
              "$ref": "#/$defs/StructureRelationship"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Object containing links to relationships with entries of the `structures` type."
        }
      },
      "title": "EntryRelationships",
      "type": "object"
    },
    "Link": {
      "description": "A link **MUST** be represented as either: a string containing the link's URL or a link object.",
      "properties": {
        "href": {
          "description": "a string containing the link's URL.",
          "format": "uri",
          "minLength": 1,
          "title": "Href",
          "type": "string"
        },
        "meta": {
          "anyOf": [
            {
              "$ref": "#/$defs/Meta"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a meta object containing non-standard meta-information about the link."
        }
      },
      "required": ["href"],
      "title": "Link",
      "type": "object"
    },
    "Meta": {
      "additionalProperties": true,
      "description": "Non-standard meta-information that can not be represented as an attribute or relationship.",
      "properties": {},
      "title": "Meta",
      "type": "object"
    },
    "Periodicity": {
      "description": "Integer enumeration of dimension_types values",
      "enum": [0, 1],
      "title": "Periodicity",
      "type": "integer"
    },
    "ReferenceRelationship": {
      "properties": {
        "links": {
          "anyOf": [
            {
              "$ref": "#/$defs/RelationshipLinks"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a links object containing at least one of the following: self, related"
        },
        "data": {
          "anyOf": [
            {
              "$ref": "#/$defs/BaseRelationshipResource"
            },
            {
              "items": {
                "$ref": "#/$defs/BaseRelationshipResource"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Resource linkage",
          "title": "Data",
          "uniqueItems": true
        },
        "meta": {
          "anyOf": [
            {
              "$ref": "#/$defs/Meta"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a meta object that contains non-standard meta-information about the relationship."
        }
      },
      "title": "ReferenceRelationship",
      "type": "object"
    },
    "RelationshipLinks": {
      "description": "A resource object **MAY** contain references to other resource objects (\"relationships\").\nRelationships may be to-one or to-many.\nRelationships can be specified by including a member in a resource's links object.",
      "properties": {
        "self": {
          "anyOf": [
            {
              "format": "uri",
              "minLength": 1,
              "type": "string"
            },
            {
              "$ref": "#/$defs/Link"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A link for the relationship itself (a 'relationship link').\nThis link allows the client to directly manipulate the relationship.\nWhen fetched successfully, this link returns the [linkage](https://jsonapi.org/format/1.0/#document-resource-object-linkage) for the related resources as its primary data.\n(See [Fetching Relationships](https://jsonapi.org/format/1.0/#fetching-relationships).)",
          "title": "Self"
        },
        "related": {
          "anyOf": [
            {
              "format": "uri",
              "minLength": 1,
              "type": "string"
            },
            {
              "$ref": "#/$defs/Link"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A [related resource link](https://jsonapi.org/format/1.0/#document-resource-object-related-resource-links).",
          "title": "Related"
        }
      },
      "title": "RelationshipLinks",
      "type": "object"
    },
    "ResourceLinks": {
      "description": "A Resource Links object",
      "properties": {
        "self": {
          "anyOf": [
            {
              "format": "uri",
              "minLength": 1,
              "type": "string"
            },
            {
              "$ref": "#/$defs/Link"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A link that identifies the resource represented by the resource object.",
          "title": "Self"
        }
      },
      "title": "ResourceLinks",
      "type": "object"
    },
    "Species": {
      "description": "A list describing the species of the sites of this structure.\n\nSpecies can represent pure chemical elements, virtual-crystal atoms representing a\nstatistical occupation of a given site by multiple chemical elements, and/or a\nlocation to which there are attached atoms, i.e., atoms whose precise location are\nunknown beyond that they are attached to that position (frequently used to indicate\nhydrogen atoms attached to another element, e.g., a carbon with three attached\nhydrogens might represent a methyl group, -CH3).\n\n- **Examples**:\n    - `[ {\"name\": \"Ti\", \"chemical_symbols\": [\"Ti\"], \"concentration\": [1.0]} ]`: any site with this species is occupied by a Ti atom.\n    - `[ {\"name\": \"Ti\", \"chemical_symbols\": [\"Ti\", \"vacancy\"], \"concentration\": [0.9, 0.1]} ]`: any site with this species is occupied by a Ti atom with 90 % probability, and has a vacancy with 10 % probability.\n    - `[ {\"name\": \"BaCa\", \"chemical_symbols\": [\"vacancy\", \"Ba\", \"Ca\"], \"concentration\": [0.05, 0.45, 0.5], \"mass\": [0.0, 137.327, 40.078]} ]`: any site with this species is occupied by a Ba atom with 45 % probability, a Ca atom with 50 % probability, and by a vacancy with 5 % probability. The mass of this site is (on average) 88.5 a.m.u.\n    - `[ {\"name\": \"C12\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"mass\": [12.0]} ]`: any site with this species is occupied by a carbon isotope with mass 12.\n    - `[ {\"name\": \"C13\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"mass\": [13.0]} ]`: any site with this species is occupied by a carbon isotope with mass 13.\n    - `[ {\"name\": \"CH3\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"attached\": [\"H\"], \"nattached\": [3]} ]`: any site with this species is occupied by a methyl group, -CH3, which is represented without specifying precise positions of the hydrogen atoms.",
      "properties": {
        "name": {
          "description": "Gives the name of the species; the **name** value MUST be unique in the `species` list.",
          "title": "Name",
          "type": "string",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "must"
        },
        "chemical_symbols": {
          "description": "MUST be a list of strings of all chemical elements composing this species. Each item of the list MUST be one of the following:\n\n- a valid chemical-element symbol, or\n- the special value `\"X\"` to represent a non-chemical element, or\n- the special value `\"vacancy\"` to represent that this site has a non-zero probability of having a vacancy (the respective probability is indicated in the `concentration` list, see below).\n\nIf any one entry in the `species` list has a `chemical_symbols` list that is longer than 1 element, the correct flag MUST be set in the list `structure_features`.",
          "items": {
            "pattern": "(H|He|Li|Be|B|C|N|O|F|Ne|Na|Mg|Al|Si|P|S|Cl|Ar|K|Ca|Sc|Ti|V|Cr|Mn|Fe|Co|Ni|Cu|Zn|Ga|Ge|As|Se|Br|Kr|Rb|Sr|Y|Zr|Nb|Mo|Tc|Ru|Rh|Pd|Ag|Cd|In|Sn|Sb|Te|I|Xe|Cs|Ba|La|Ce|Pr|Nd|Pm|Sm|Eu|Gd|Tb|Dy|Ho|Er|Tm|Yb|Lu|Hf|Ta|W|Re|Os|Ir|Pt|Au|Hg|Tl|Pb|Bi|Po|At|Rn|Fr|Ra|Ac|Th|Pa|U|Np|Pu|Am|Cm|Bk|Cf|Es|Fm|Md|No|Lr|Rf|Db|Sg|Bh|Hs|Mt|Ds|Rg|Cn|Nh|Fl|Mc|Lv|Ts|Og|X|vacancy)",
            "type": "string"
          },
          "title": "Chemical Symbols",
          "type": "array",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "must"
        },
        "concentration": {
          "description": "MUST be a list of floats, with same length as `chemical_symbols`. The numbers represent the relative concentration of the corresponding chemical symbol in this species. The numbers SHOULD sum to one. Cases in which the numbers do not sum to one typically fall only in the following two categories:\n\n- Numerical errors when representing float numbers in fixed precision, e.g. for two chemical symbols with concentrations `1/3` and `2/3`, the concentration might look something like `[0.33333333333, 0.66666666666]`. If the client is aware that the sum is not one because of numerical precision, it can renormalize the values so that the sum is exactly one.\n- Experimental errors in the data present in the database. In this case, it is the responsibility of the client to decide how to process the data.\n\nNote that concentrations are uncorrelated between different site (even of the same species).",
          "items": {
            "type": "number"
          },
          "title": "Concentration",
          "type": "array",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "must"
        },
        "mass": {
          "anyOf": [
            {
              "items": {
                "type": "number"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "If present MUST be a list of floats expressed in a.m.u.\nElements denoting vacancies MUST have masses equal to 0.",
          "title": "Mass",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional",
          "x-optimade-unit": "a.m.u."
        },
        "original_name": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Can be any valid Unicode string, and SHOULD contain (if specified) the name of the species that is used internally in the source database.\n\nNote: With regards to \"source database\", we refer to the immediate source being queried via the OPTIMADE API implementation.",
          "title": "Original Name",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional"
        },
        "attached": {
          "anyOf": [
            {
              "items": {
                "type": "string"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "If provided MUST be a list of length 1 or more of strings of chemical symbols for the elements attached to this site, or \"X\" for a non-chemical element.",
          "title": "Attached",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional"
        },
        "nattached": {
          "anyOf": [
            {
              "items": {
                "type": "integer"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "If provided MUST be a list of length 1 or more of integers indicating the number of attached atoms of the kind specified in the value of the :field:`attached` key.",
          "title": "Nattached",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional"
        }
      },
      "required": ["name", "chemical_symbols", "concentration"],
      "title": "Species",
      "type": "object"
    },
    "StructureFeatures": {
      "description": "Enumeration of structure_features values",
      "enum": ["disorder", "implicit_atoms", "site_attachments", "assemblies"],
      "title": "StructureFeatures",
      "type": "string"
    },
    "StructureRelationship": {
      "properties": {
        "links": {
          "anyOf": [
            {
              "$ref": "#/$defs/RelationshipLinks"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a links object containing at least one of the following: self, related"
        },
        "data": {
          "anyOf": [
            {
              "$ref": "#/$defs/BaseRelationshipResource"
            },
            {
              "items": {
                "$ref": "#/$defs/BaseRelationshipResource"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Resource linkage",
          "title": "Data",
          "uniqueItems": true
        },
        "meta": {
          "anyOf": [
            {
              "$ref": "#/$defs/Meta"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a meta object that contains non-standard meta-information about the relationship."
        }
      },
      "title": "StructureRelationship",
      "type": "object"
    },
    "StructureResource": {
      "description": "Representing a structure.",
      "properties": {
        "id": {
          "description": "An entry's ID as defined in section Definition of Terms.\n\n- **Type**: string.\n\n- **Requirements/Conventions**:\n    - **Support**: MUST be supported by all implementations, MUST NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - **Response**: REQUIRED in the response.\n\n- **Examples**:\n    - `\"db/1234567\"`\n    - `\"cod/2000000\"`\n    - `\"cod/2000000@1234567\"`\n    - `\"nomad/L1234567890\"`\n    - `\"42\"`",
          "title": "Id",
          "type": "string",
          "x-optimade-queryable": "must",
          "x-optimade-support": "must"
        },
        "type": {
          "const": "structures",
          "default": "structures",
          "description": "The name of the type of an entry.\n\n- **Type**: string.\n\n- **Requirements/Conventions**:\n    - **Support**: MUST be supported by all implementations, MUST NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - **Response**: REQUIRED in the response.\n    - MUST be an existing entry type.\n    - The entry of type `<type>` and ID `<id>` MUST be returned in response to a request for `/<type>/<id>` under the versioned base URL.\n\n- **Examples**:\n    - `\"structures\"`",
          "pattern": "^structures$",
          "title": "Type",
          "type": "string",
          "x-optimade-queryable": "must",
          "x-optimade-support": "must"
        },
        "links": {
          "anyOf": [
            {
              "$ref": "#/$defs/ResourceLinks"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a links object containing links related to the resource."
        },
        "meta": {
          "anyOf": [
            {
              "$ref": "#/$defs/Meta"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "a meta object containing non-standard meta-information about a resource that can not be represented as an attribute or relationship."
        },
        "attributes": {
          "$ref": "#/$defs/StructureResourceAttributes"
        },
        "relationships": {
          "anyOf": [
            {
              "$ref": "#/$defs/EntryRelationships"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A dictionary containing references to other entries according to the description in section Relationships encoded as [JSON API Relationships](https://jsonapi.org/format/1.0/#document-resource-object-relationships).\nThe OPTIONAL human-readable description of the relationship MAY be provided in the `description` field inside the `meta` dictionary of the JSON API resource identifier object."
        }
      },
      "required": ["id", "type", "attributes"],
      "title": "StructureResource",
      "type": "object"
    },
    "StructureResourceAttributes": {
      "additionalProperties": true,
      "description": "This class contains the Field for the attributes used to represent a structure, e.g. unit cell, atoms, positions.",
      "properties": {
        "immutable_id": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The entry's immutable ID (e.g., an UUID). This is important for databases having preferred IDs that point to \"the latest version\" of a record, but still offer access to older variants. This ID maps to the version-specific record, in case it changes in the future.\n\n- **Type**: string.\n\n- **Requirements/Conventions**:\n    - **Support**: OPTIONAL support in implementations, i.e., MAY be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n\n- **Examples**:\n    - `\"8bd3e750-b477-41a0-9b11-3a799f21b44f\"`\n    - `\"fjeiwoj,54;@=%<>#32\"` (Strings that are not URL-safe are allowed.)",
          "title": "Immutable Id",
          "x-optimade-queryable": "must",
          "x-optimade-support": "optional"
        },
        "last_modified": {
          "anyOf": [
            {
              "format": "date-time",
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "description": "Date and time representing when the entry was last modified.\n\n- **Type**: timestamp.\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - **Response**: REQUIRED in the response unless the query parameter `response_fields` is present and does not include this property.\n\n- **Example**:\n    - As part of JSON response format: `\"2007-04-05T14:30:20Z\"` (i.e., encoded as an [RFC 3339 Internet Date/Time Format](https://tools.ietf.org/html/rfc3339#section-5.6) string.)",
          "title": "Last Modified",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "elements": {
          "anyOf": [
            {
              "items": {
                "type": "string"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The chemical symbols of the different elements present in the structure.\n\n- **Type**: list of strings.\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - The strings are the chemical symbols, i.e., either a single uppercase letter or an uppercase letter followed by a number of lowercase letters.\n    - The order MUST be alphabetical.\n    - MUST refer to the same elements in the same order, and therefore be of the same length, as `elements_ratios`, if the latter is provided.\n    - Note: This property SHOULD NOT contain the string \"X\" to indicate non-chemical elements or \"vacancy\" to indicate vacancies (in contrast to the field `chemical_symbols` for the `species` property).\n\n- **Examples**:\n    - `[\"Si\"]`\n    - `[\"Al\",\"O\",\"Si\"]`\n\n- **Query examples**:\n    - A filter that matches all records of structures that contain Si, Al **and** O, and possibly other elements: `elements HAS ALL \"Si\", \"Al\", \"O\"`.\n    - To match structures with exactly these three elements, use `elements HAS ALL \"Si\", \"Al\", \"O\" AND elements LENGTH 3`.\n    - Note: length queries on this property can be equivalently formulated by filtering on the `nelements`_ property directly.",
          "title": "Elements",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "nelements": {
          "anyOf": [
            {
              "type": "integer"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Number of different elements in the structure as an integer.\n\n- **Type**: integer\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - MUST be equal to the lengths of the list properties `elements` and `elements_ratios`, if they are provided.\n\n- **Examples**:\n    - `3`\n\n- **Querying**:\n    - Note: queries on this property can equivalently be formulated using `elements LENGTH`.\n    - A filter that matches structures that have exactly 4 elements: `nelements=4`.\n    - A filter that matches structures that have between 2 and 7 elements: `nelements>=2 AND nelements<=7`.",
          "title": "Nelements",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "elements_ratios": {
          "anyOf": [
            {
              "items": {
                "type": "number"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Relative proportions of different elements in the structure.\n\n- **Type**: list of floats\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - Composed by the proportions of elements in the structure as a list of floating point numbers.\n    - The sum of the numbers MUST be 1.0 (within floating point accuracy)\n    - MUST refer to the same elements in the same order, and therefore be of the same length, as `elements`, if the latter is provided.\n\n- **Examples**:\n    - `[1.0]`\n    - `[0.3333333333333333, 0.2222222222222222, 0.4444444444444444]`\n\n- **Query examples**:\n    - Note: Useful filters can be formulated using the set operator syntax for correlated values.\n      However, since the values are floating point values, the use of equality comparisons is generally inadvisable.\n    - OPTIONAL: a filter that matches structures where approximately 1/3 of the atoms in the structure are the element Al is: `elements:elements_ratios HAS ALL \"Al\":>0.3333, \"Al\":<0.3334`.",
          "title": "Elements Ratios",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "chemical_formula_descriptive": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The chemical formula for a structure as a string in a form chosen by the API implementation.\n\n- **Type**: string\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - The chemical formula is given as a string consisting of properly capitalized element symbols followed by integers or decimal numbers, balanced parentheses, square, and curly brackets `(`,`)`, `[`,`]`, `{`, `}`, commas, the `+`, `-`, `:` and `=` symbols. The parentheses are allowed to be followed by a number. Spaces are allowed anywhere except within chemical symbols. The order of elements and any groupings indicated by parentheses or brackets are chosen freely by the API implementation.\n    - The string SHOULD be arithmetically consistent with the element ratios in the `chemical_formula_reduced` property.\n    - It is RECOMMENDED, but not mandatory, that symbols, parentheses and brackets, if used, are used with the meanings prescribed by [IUPAC's Nomenclature of Organic Chemistry](https://www.qmul.ac.uk/sbcs/iupac/bibliog/blue.html).\n\n- **Examples**:\n    - `\"(H2O)2 Na\"`\n    - `\"NaCl\"`\n    - `\"CaCO3\"`\n    - `\"CCaO3\"`\n    - `\"(CH3)3N+ - [CH2]2-OH = Me3N+ - CH2 - CH2OH\"`\n\n- **Query examples**:\n    - Note: the free-form nature of this property is likely to make queries on it across different databases inconsistent.\n    - A filter that matches an exactly given formula: `chemical_formula_descriptive=\"(H2O)2 Na\"`.\n    - A filter that does a partial match: `chemical_formula_descriptive CONTAINS \"H2O\"`.",
          "title": "Chemical Formula Descriptive",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "chemical_formula_reduced": {
          "anyOf": [
            {
              "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The reduced chemical formula for a structure as a string with element symbols and integer chemical proportion numbers.\nThe proportion number MUST be omitted if it is 1.\n\n- **Type**: string\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property.\n      However, support for filters using partial string matching with this property is OPTIONAL (i.e., BEGINS WITH, ENDS WITH, and CONTAINS).\n      Intricate queries on formula components are instead suggested to be formulated using set-type filter operators on the multi valued `elements` and `elements_ratios` properties.\n    - Element symbols MUST have proper capitalization (e.g., `\"Si\"`, not `\"SI\"` for \"silicon\").\n    - Elements MUST be placed in alphabetical order, followed by their integer chemical proportion number.\n    - For structures with no partial occupation, the chemical proportion numbers are the smallest integers for which the chemical proportion is exactly correct.\n    - For structures with partial occupation, the chemical proportion numbers are integers that within reasonable approximation indicate the correct chemical proportions. The precise details of how to perform the rounding is chosen by the API implementation.\n    - No spaces or separators are allowed.\n\n- **Examples**:\n    - `\"H2NaO\"`\n    - `\"ClNa\"`\n    - `\"CCaO3\"`\n\n- **Query examples**:\n    - A filter that matches an exactly given formula is `chemical_formula_reduced=\"H2NaO\"`.",
          "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
          "title": "Chemical Formula Reduced",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "chemical_formula_hill": {
          "anyOf": [
            {
              "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The chemical formula for a structure in [Hill form](https://dx.doi.org/10.1021/ja02046a005) with element symbols followed by integer chemical proportion numbers. The proportion number MUST be omitted if it is 1.\n\n- **Type**: string\n\n- **Requirements/Conventions**:\n    - **Support**: OPTIONAL support in implementations, i.e., MAY be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n      If supported, only a subset of the filter features MAY be supported.\n    - The overall scale factor of the chemical proportions is chosen such that the resulting values are integers that indicate the most chemically relevant unit of which the system is composed.\n      For example, if the structure is a repeating unit cell with four hydrogens and four oxygens that represents two hydroperoxide molecules, `chemical_formula_hill` is `\"H2O2\"` (i.e., not `\"HO\"`, nor `\"H4O4\"`).\n    - If the chemical insight needed to ascribe a Hill formula to the system is not present, the property MUST be handled as unset.\n    - Element symbols MUST have proper capitalization (e.g., `\"Si\"`, not `\"SI\"` for \"silicon\").\n    - Elements MUST be placed in [Hill order](https://dx.doi.org/10.1021/ja02046a005), followed by their integer chemical proportion number.\n      Hill order means: if carbon is present, it is placed first, and if also present, hydrogen is placed second.\n      After that, all other elements are ordered alphabetically.\n      If carbon is not present, all elements are ordered alphabetically.\n    - If the system has sites with partial occupation and the total occupations of each element do not all sum up to integers, then the Hill formula SHOULD be handled as unset.\n    - No spaces or separators are allowed.\n\n- **Examples**:\n    - `\"H2O2\"`\n\n- **Query examples**:\n    - A filter that matches an exactly given formula is `chemical_formula_hill=\"H2O2\"`.",
          "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
          "title": "Chemical Formula Hill",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional"
        },
        "chemical_formula_anonymous": {
          "anyOf": [
            {
              "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The anonymous formula is the `chemical_formula_reduced`, but where the elements are instead first ordered by their chemical proportion number, and then, in order left to right, replaced by anonymous symbols A, B, C, ..., Z, Aa, Ba, ..., Za, Ab, Bb, ... and so on.\n\n- **Type**: string\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property.\n      However, support for filters using partial string matching with this property is OPTIONAL (i.e., BEGINS WITH, ENDS WITH, and CONTAINS).\n\n- **Examples**:\n    - `\"A2B\"`\n    - `\"A42B42C16D12E10F9G5\"`\n\n- **Querying**:\n    - A filter that matches an exactly given formula is `chemical_formula_anonymous=\"A2B\"`.",
          "pattern": "(^$)|^([A-Z][a-z]?([2-9]|[1-9]\\d+)?)+$",
          "title": "Chemical Formula Anonymous",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "dimension_types": {
          "anyOf": [
            {
              "items": {
                "$ref": "#/$defs/Periodicity"
              },
              "maxItems": 3,
              "minItems": 3,
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "List of three integers.\nFor each of the three directions indicated by the three lattice vectors (see property `lattice_vectors`), this list indicates if the direction is periodic (value `1`) or non-periodic (value `0`).\nNote: the elements in this list each refer to the direction of the corresponding entry in `lattice_vectors` and *not* the Cartesian x, y, z directions.\n\n- **Type**: list of integers.\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n    - MUST be a list of length 3.\n    - Each integer element MUST assume only the value 0 or 1.\n\n- **Examples**:\n    - For a molecule: `[0, 0, 0]`\n    - For a wire along the direction specified by the third lattice vector: `[0, 0, 1]`\n    - For a 2D surface/slab, periodic on the plane defined by the first and third lattice vectors: `[1, 0, 1]`\n    - For a bulk 3D system: `[1, 1, 1]`",
          "title": "Dimension Types",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "should"
        },
        "nperiodic_dimensions": {
          "anyOf": [
            {
              "type": "integer"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "An integer specifying the number of periodic dimensions in the structure, equivalent to the number of non-zero entries in `dimension_types`.\n\n- **Type**: integer\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n    - The integer value MUST be between 0 and 3 inclusive and MUST be equal to the sum of the items in the `dimension_types` property.\n    - This property only reflects the treatment of the lattice vectors provided for the structure, and not any physical interpretation of the dimensionality of its contents.\n\n- **Examples**:\n    - `2` should be indicated in cases where `dimension_types` is any of `[1, 1, 0]`, `[1, 0, 1]`, `[0, 1, 1]`.\n\n- **Query examples**:\n    - Match only structures with exactly 3 periodic dimensions: `nperiodic_dimensions=3`\n    - Match all structures with 2 or fewer periodic dimensions: `nperiodic_dimensions<=2`",
          "title": "Nperiodic Dimensions",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "lattice_vectors": {
          "anyOf": [
            {
              "items": {
                "items": {
                  "anyOf": [
                    {
                      "type": "number"
                    },
                    {
                      "type": "null"
                    }
                  ]
                },
                "maxItems": 3,
                "minItems": 3,
                "type": "array"
              },
              "maxItems": 3,
              "minItems": 3,
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The three lattice vectors in Cartesian coordinates, in \u00e5ngstr\u00f6m (\u00c5).\n\n- **Type**: list of list of floats or unknown values.\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n      If supported, filters MAY support only a subset of comparison operators.\n    - MUST be a list of three vectors *a*, *b*, and *c*, where each of the vectors MUST BE a list of the vector's coordinates along the x, y, and z Cartesian coordinates.\n      (Therefore, the first index runs over the three lattice vectors and the second index runs over the x, y, z Cartesian coordinates).\n    - For databases that do not define an absolute Cartesian system (e.g., only defining the length and angles between vectors), the first lattice vector SHOULD be set along *x* and the second on the *xy*-plane.\n    - MUST always contain three vectors of three coordinates each, independently of the elements of property `dimension_types`.\n      The vectors SHOULD by convention be chosen so the determinant of the `lattice_vectors` matrix is different from zero.\n      The vectors in the non-periodic directions have no significance beyond fulfilling these requirements.\n    - The coordinates of the lattice vectors of non-periodic dimensions (i.e., those dimensions for which `dimension_types` is `0`) MAY be given as a list of all `null` values.\n        If a lattice vector contains the value `null`, all coordinates of that lattice vector MUST be `null`.\n\n- **Examples**:\n    - `[[4.0,0.0,0.0],[0.0,4.0,0.0],[0.0,1.0,4.0]]` represents a cell, where the first vector is `(4, 0, 0)`, i.e., a vector aligned along the `x` axis of length 4 \u00c5; the second vector is `(0, 4, 0)`; and the third vector is `(0, 1, 4)`.",
          "title": "Lattice Vectors",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "should",
          "x-optimade-unit": "\u00c5"
        },
        "cartesian_site_positions": {
          "anyOf": [
            {
              "items": {
                "items": {
                  "type": "number"
                },
                "maxItems": 3,
                "minItems": 3,
                "type": "array"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Cartesian positions of each site in the structure.\nA site is usually used to describe positions of atoms; what atoms can be encountered at a given site is conveyed by the `species_at_sites` property, and the species themselves are described in the `species` property.\n\n- **Type**: list of list of floats\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n      If supported, filters MAY support only a subset of comparison operators.\n    - It MUST be a list of length equal to the number of sites in the structure, where every element is a list of the three Cartesian coordinates of a site expressed as float values in the unit angstrom (\u00c5).\n    - An entry MAY have multiple sites at the same Cartesian position (for a relevant use of this, see e.g., the property `assemblies`).\n\n- **Examples**:\n    - `[[0,0,0],[0,0,2]]` indicates a structure with two sites, one sitting at the origin and one along the (positive) *z*-axis, 2 \u00c5 away from the origin.",
          "title": "Cartesian Site Positions",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "should",
          "x-optimade-unit": "\u00c5"
        },
        "nsites": {
          "anyOf": [
            {
              "type": "integer"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "An integer specifying the length of the `cartesian_site_positions` property.\n\n- **Type**: integer\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: MUST be a queryable property with support for all mandatory filter features.\n\n- **Examples**:\n    - `42`\n\n- **Query examples**:\n    - Match only structures with exactly 4 sites: `nsites=4`\n    - Match structures that have between 2 and 7 sites: `nsites>=2 AND nsites<=7`",
          "title": "Nsites",
          "x-optimade-queryable": "must",
          "x-optimade-support": "should"
        },
        "species": {
          "anyOf": [
            {
              "items": {
                "$ref": "#/$defs/Species"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A list describing the species of the sites of this structure.\nSpecies can represent pure chemical elements, virtual-crystal atoms representing a statistical occupation of a given site by multiple chemical elements, and/or a location to which there are attached atoms, i.e., atoms whose precise location are unknown beyond that they are attached to that position (frequently used to indicate hydrogen atoms attached to another element, e.g., a carbon with three attached hydrogens might represent a methyl group, -CH3).\n\n- **Type**: list of dictionary with keys:\n    - `name`: string (REQUIRED)\n    - `chemical_symbols`: list of strings (REQUIRED)\n    - `concentration`: list of float (REQUIRED)\n    - `attached`: list of strings (REQUIRED)\n    - `nattached`: list of integers (OPTIONAL)\n    - `mass`: list of floats (OPTIONAL)\n    - `original_name`: string (OPTIONAL).\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n        If supported, filters MAY support only a subset of comparison operators.\n    - Each list member MUST be a dictionary with the following keys:\n        - **name**: REQUIRED; gives the name of the species; the **name** value MUST be unique in the `species` list;\n        - **chemical_symbols**: REQUIRED; MUST be a list of strings of all chemical elements composing this species.\n          Each item of the list MUST be one of the following:\n            - a valid chemical-element symbol, or\n            - the special value `\"X\"` to represent a non-chemical element, or\n            - the special value `\"vacancy\"` to represent that this site has a non-zero probability of having a vacancy (the respective probability is indicated in the `concentration` list, see below).\n\n          If any one entry in the `species` list has a `chemical_symbols` list that is longer than 1 element, the correct flag MUST be set in the list `structure_features`.\n\n        - **concentration**: REQUIRED; MUST be a list of floats, with same length as `chemical_symbols`.\n          The numbers represent the relative concentration of the corresponding chemical symbol in this species.\n          The numbers SHOULD sum to one. Cases in which the numbers do not sum to one typically fall only in the following two categories:\n\n            - Numerical errors when representing float numbers in fixed precision, e.g. for two chemical symbols with concentrations `1/3` and `2/3`, the concentration might look something like `[0.33333333333, 0.66666666666]`. If the client is aware that the sum is not one because of numerical precision, it can renormalize the values so that the sum is exactly one.\n            - Experimental errors in the data present in the database. In this case, it is the responsibility of the client to decide how to process the data.\n\n            Note that concentrations are uncorrelated between different sites (even of the same species).\n\n        - **attached**: OPTIONAL; if provided MUST be a list of length 1 or more of strings of chemical symbols for the elements attached to this site, or \"X\" for a non-chemical element.\n\n        - **nattached**: OPTIONAL; if provided MUST be a list of length 1 or more of integers indicating the number of attached atoms of the kind specified in the value of the `attached` key.\n\n          The implementation MUST include either both or none of the `attached` and `nattached` keys, and if they are provided, they MUST be of the same length.\n          Furthermore, if they are provided, the `structure_features` property MUST include the string `site_attachments`.\n\n        - **mass**: OPTIONAL. If present MUST be a list of floats, with the same length as `chemical_symbols`, providing element masses expressed in a.m.u.\n          Elements denoting vacancies MUST have masses equal to 0.\n\n        - **original_name**: OPTIONAL. Can be any valid Unicode string, and SHOULD contain (if specified) the name of the species that is used internally in the source database.\n\n          Note: With regards to \"source database\", we refer to the immediate source being queried via the OPTIMADE API implementation.\n\n          The main use of this field is for source databases that use species names, containing characters that are not allowed (see description of the list property `species_at_sites`).\n\n    - For systems that have only species formed by a single chemical symbol, and that have at most one species per chemical symbol, SHOULD use the chemical symbol as species name (e.g., `\"Ti\"` for titanium, `\"O\"` for oxygen, etc.)\n      However, note that this is OPTIONAL, and client implementations MUST NOT assume that the key corresponds to a chemical symbol, nor assume that if the species name is a valid chemical symbol, that it represents a species with that chemical symbol.\n      This means that a species `{\"name\": \"C\", \"chemical_symbols\": [\"Ti\"], \"concentration\": [1.0]}` is valid and represents a titanium species (and *not* a carbon species).\n    - It is NOT RECOMMENDED that a structure includes species that do not have at least one corresponding site.\n\n- **Examples**:\n    - `[ {\"name\": \"Ti\", \"chemical_symbols\": [\"Ti\"], \"concentration\": [1.0]} ]`: any site with this species is occupied by a Ti atom.\n    - `[ {\"name\": \"Ti\", \"chemical_symbols\": [\"Ti\", \"vacancy\"], \"concentration\": [0.9, 0.1]} ]`: any site with this species is occupied by a Ti atom with 90 % probability, and has a vacancy with 10 % probability.\n    - `[ {\"name\": \"BaCa\", \"chemical_symbols\": [\"vacancy\", \"Ba\", \"Ca\"], \"concentration\": [0.05, 0.45, 0.5], \"mass\": [0.0, 137.327, 40.078]} ]`: any site with this species is occupied by a Ba atom with 45 % probability, a Ca atom with 50 % probability, and by a vacancy with 5 % probability. The mass of this site is (on average) 88.5 a.m.u.\n    - `[ {\"name\": \"C12\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"mass\": [12.0]} ]`: any site with this species is occupied by a carbon isotope with mass 12.\n    - `[ {\"name\": \"C13\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"mass\": [13.0]} ]`: any site with this species is occupied by a carbon isotope with mass 13.\n    - `[ {\"name\": \"CH3\", \"chemical_symbols\": [\"C\"], \"concentration\": [1.0], \"attached\": [\"H\"], \"nattached\": [3]} ]`: any site with this species is occupied by a methyl group, -CH3, which is represented without specifying precise positions of the hydrogen atoms.",
          "title": "Species",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "should"
        },
        "species_at_sites": {
          "anyOf": [
            {
              "items": {
                "type": "string"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "Name of the species at each site (where values for sites are specified with the same order of the property `cartesian_site_positions`).\nThe properties of the species are found in the property `species`.\n\n- **Type**: list of strings.\n\n- **Requirements/Conventions**:\n    - **Support**: SHOULD be supported by all implementations, i.e., SHOULD NOT be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n      If supported, filters MAY support only a subset of comparison operators.\n    - MUST have length equal to the number of sites in the structure (first dimension of the list property `cartesian_site_positions`).\n    - Each species name mentioned in the `species_at_sites` list MUST be described in the list property `species` (i.e. for each value in the `species_at_sites` list there MUST exist exactly one dictionary in the `species` list with the `name` attribute equal to the corresponding `species_at_sites` value).\n    - Each site MUST be associated only to a single species.\n      **Note**: However, species can represent mixtures of atoms, and multiple species MAY be defined for the same chemical element.\n      This latter case is useful when different atoms of the same type need to be grouped or distinguished, for instance in simulation codes to assign different initial spin states.\n\n- **Examples**:\n    - `[\"Ti\",\"O2\"]` indicates that the first site is hosting a species labeled `\"Ti\"` and the second a species labeled `\"O2\"`.\n    - `[\"Ac\", \"Ac\", \"Ag\", \"Ir\"]` indicating the first two sites contains the `\"Ac\"` species, while the third and fourth sites contain the `\"Ag\"` and `\"Ir\"` species, respectively.",
          "title": "Species At Sites",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "should"
        },
        "assemblies": {
          "anyOf": [
            {
              "items": {
                "$ref": "#/$defs/Assembly"
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A description of groups of sites that are statistically correlated.\n\n- **Type**: list of dictionary with keys:\n    - `sites_in_groups`: list of list of integers (REQUIRED)\n    - `group_probabilities`: list of floats (REQUIRED)\n\n- **Requirements/Conventions**:\n    - **Support**: OPTIONAL support in implementations, i.e., MAY be `null`.\n    - **Query**: Support for queries on this property is OPTIONAL.\n        If supported, filters MAY support only a subset of comparison operators.\n    - The property SHOULD be `null` for entries that have no partial occupancies.\n    - If present, the correct flag MUST be set in the list `structure_features`.\n    - Client implementations MUST check its presence (as its presence changes the interpretation of the structure).\n    - If present, it MUST be a list of dictionaries, each of which represents an assembly and MUST have the following two keys:\n        - **sites_in_groups**: Index of the sites (0-based) that belong to each group for each assembly.\n\n            Example: `[[1], [2]]`: two groups, one with the second site, one with the third.\n            Example: `[[1,2], [3]]`: one group with the second and third site, one with the fourth.\n\n        - **group_probabilities**: Statistical probability of each group. It MUST have the same length as `sites_in_groups`.\n            It SHOULD sum to one.\n            See below for examples of how to specify the probability of the occurrence of a vacancy.\n            The possible reasons for the values not to sum to one are the same as already specified above for the `concentration` of each `species`.\n\n    - If a site is not present in any group, it means that it is present with 100 % probability (as if no assembly was specified).\n    - A site MUST NOT appear in more than one group.\n\n- **Examples** (for each entry of the assemblies list):\n    - `{\"sites_in_groups\": [[0], [1]], \"group_probabilities: [0.3, 0.7]}`: the first site and the second site never occur at the same time in the unit cell.\n        Statistically, 30 % of the times the first site is present, while 70 % of the times the second site is present.\n    - `{\"sites_in_groups\": [[1,2], [3]], \"group_probabilities: [0.3, 0.7]}`: the second and third site are either present together or not present; they form the first group of atoms for this assembly.\n        The second group is formed by the fourth site.\n        Sites of the first group (the second and the third) are never present at the same time as the fourth site.\n        30 % of times sites 1 and 2 are present (and site 3 is absent); 70 % of times site 3 is present (and sites 1 and 2 are absent).\n\n- **Notes**:\n    - Assemblies are essential to represent, for instance, the situation where an atom can statistically occupy two different positions (sites).\n\n    - By defining groups, it is possible to represent, e.g., the case where a functional molecule (and not just one atom) is either present or absent (or the case where it it is present in two conformations)\n\n    - Considerations on virtual alloys and on vacancies: In the special case of a virtual alloy, these specifications allow two different, equivalent ways of specifying them.\n        For instance, for a site at the origin with 30 % probability of being occupied by Si, 50 % probability of being occupied by Ge, and 20 % of being a vacancy, the following two representations are possible:\n\n        - Using a single species:\n            ```json\n            {\n              \"cartesian_site_positions\": [[0,0,0]],\n              \"species_at_sites\": [\"SiGe-vac\"],\n              \"species\": [\n              {\n                \"name\": \"SiGe-vac\",\n                \"chemical_symbols\": [\"Si\", \"Ge\", \"vacancy\"],\n                \"concentration\": [0.3, 0.5, 0.2]\n              }\n              ]\n              // ...\n            }\n            ```\n\n        - Using multiple species and the assemblies:\n            ```json\n            {\n              \"cartesian_site_positions\": [ [0,0,0], [0,0,0], [0,0,0] ],\n              \"species_at_sites\": [\"Si\", \"Ge\", \"vac\"],\n              \"species\": [\n                { \"name\": \"Si\", \"chemical_symbols\": [\"Si\"], \"concentration\": [1.0] },\n                { \"name\": \"Ge\", \"chemical_symbols\": [\"Ge\"], \"concentration\": [1.0] },\n                { \"name\": \"vac\", \"chemical_symbols\": [\"vacancy\"], \"concentration\": [1.0] }\n              ],\n              \"assemblies\": [\n                {\n              \"sites_in_groups\": [ [0], [1], [2] ],\n              \"group_probabilities\": [0.3, 0.5, 0.2]\n                }\n              ]\n              // ...\n            }\n            ```\n\n    - It is up to the database provider to decide which representation to use, typically depending on the internal format in which the structure is stored.\n        However, given a structure identified by a unique ID, the API implementation MUST always provide the same representation for it.\n\n    - The probabilities of occurrence of different assemblies are uncorrelated.\n        So, for instance in the following case with two assemblies:\n        ```json\n        {\n          \"assemblies\": [\n            {\n              \"sites_in_groups\": [ [0], [1] ],\n              \"group_probabilities\": [0.2, 0.8],\n            },\n            {\n              \"sites_in_groups\": [ [2], [3] ],\n              \"group_probabilities\": [0.3, 0.7]\n            }\n          ]\n        }\n        ```\n\n        Site 0 is present with a probability of 20 % and site 1 with a probability of 80 %. These two sites are correlated (either site 0 or 1 is present). Similarly, site 2 is present with a probability of 30 % and site 3 with a probability of 70 %.\n        These two sites are correlated (either site 2 or 3 is present).\n        However, the presence or absence of sites 0 and 1 is not correlated with the presence or absence of sites 2 and 3 (in the specific example, the pair of sites (0, 2) can occur with 0.2*0.3 = 6 % probability; the pair (0, 3) with 0.2*0.7 = 14 % probability; the pair (1, 2) with 0.8*0.3 = 24 % probability; and the pair (1, 3) with 0.8*0.7 = 56 % probability).",
          "title": "Assemblies",
          "x-optimade-queryable": "optional",
          "x-optimade-support": "optional"
        },
        "structure_features": {
          "description": "A list of strings that flag which special features are used by the structure.\n\n- **Type**: list of strings\n\n- **Requirements/Conventions**:\n    - **Support**: MUST be supported by all implementations, MUST NOT be `null`.\n    - **Query**: MUST be a queryable property.\n    Filters on the list MUST support all mandatory HAS-type queries.\n    Filter operators for comparisons on the string components MUST support equality, support for other comparison operators are OPTIONAL.\n    - MUST be an empty list if no special features are used.\n    - MUST be sorted alphabetically.\n    - If a special feature listed below is used, the list MUST contain the corresponding string.\n    - If a special feature listed below is not used, the list MUST NOT contain the corresponding string.\n    - **List of strings used to indicate special structure features**:\n        - `disorder`: this flag MUST be present if any one entry in the `species` list has a `chemical_symbols` list that is longer than 1 element.\n        - `implicit_atoms`: this flag MUST be present if the structure contains atoms that are not assigned to sites via the property `species_at_sites` (e.g., because their positions are unknown).\n           When this flag is present, the properties related to the chemical formula will likely not match the type and count of atoms represented by the `species_at_sites`, `species` and `assemblies` properties.\n        - `site_attachments`: this flag MUST be present if any one entry in the `species` list includes `attached` and `nattached`.\n        - `assemblies`: this flag MUST be present if the property `assemblies` is present.\n\n- **Examples**: A structure having implicit atoms and using assemblies: `[\"assemblies\", \"implicit_atoms\"]`",
          "items": {
            "$ref": "#/$defs/StructureFeatures"
          },
          "title": "Structure Features",
          "type": "array",
          "x-optimade-queryable": "must",
          "x-optimade-support": "must"
        }
      },
      "required": ["last_modified", "structure_features"],
      "title": "StructureResourceAttributes",
      "type": "object"
    }
  },
  "properties": {
    "forces": {
      "description": "The forces on the atoms.",
      "items": {
        "type": "number"
      },
      "title": "Forces",
      "type": "array",
      "units": "eV/\u00c5"
    },
    "relaxed_structure": {
      "anyOf": [
        {
          "$ref": "#/$defs/StructureResource"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "The relaxed structure, if relaxation was performed.",
      "units": "\u00c5"
    },
    "total_energy": {
      "description": "The total energy associated to the relaxed structure (or initial structure in case no relaxation is performed.",
      "title": "Total Energy",
      "type": "number",
      "units": "eV"
    },
    "stress": {
      "anyOf": [
        {
          "items": {
            "type": "number"
          },
          "type": "array"
        },
        {
          "type": "null"
        }
      ],
      "description": "The final stress tensor in eV/\u00c5^3, if relaxation was performed.",
      "title": "Stress",
      "units": "eV/\u00c5^3"
    },
    "total_magnetization": {
      "anyOf": [
        {
          "description": "The total magnetization of a system.",
          "type": "number",
          "units": "\u03bcB"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "title": "Total Magnetization"
    },
    "hartree_potential": {
      "anyOf": [
        {
          "items": {
            "type": "number"
          },
          "type": "array"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "The Hartree potential.",
      "title": "Hartree Potential",
      "units": "Rydberg"
    },
    "charge_density": {
      "anyOf": [
        {
          "items": {
            "type": "number"
          },
          "type": "array"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "The total magnetization of the system in \u03bcB.",
      "title": "Charge Density",
      "units": "Rydberg"
    }
  },
  "required": ["forces", "total_energy", "stress"],
  "title": "RelaxOutputs",
  "type": "object",
  "@type": "RelaxOutputs",
  "forces": [[0.0, 0.0, 0.0]],
  "relaxed_structure": {
    "id": "",
    "type": "structures",
    "links": null,
    "meta": null,
    "attributes": {
      "immutable_id": null,
      "last_modified": null,
      "elements": ["Si"],
      "nelements": 1,
      "elements_ratios": [1.0],
      "chemical_formula_descriptive": "Si",
      "chemical_formula_reduced": "Si",
      "chemical_formula_hill": null,
      "chemical_formula_anonymous": "A",
      "dimension_types": [1, 1, 1],
      "nperiodic_dimensions": 3,
      "lattice_vectors": [
        [0.0, 1.934867891363, 1.934867891363],
        [1.934867891363, 0.0, 1.934867891363],
        [1.934867891363, 1.934867891363, 0.0]
      ],
      "cartesian_site_positions": [[0.0, 0.0, 0.0]],
      "nsites": 1,
      "species": [
        {
          "name": "Si",
          "chemical_symbols": ["Si"],
          "concentration": [1.0],
          "mass": null,
          "original_name": null,
          "attached": null,
          "nattached": null
        }
      ],
      "species_at_sites": ["Si"],
      "assemblies": null,
      "structure_features": []
    },
    "relationships": null
  },
  "total_energy": -153.80238980594,
  "stress": [
    [-0.00014506888966986438, -0.0, -0.0],
    [-0.0, -0.00014506888966986438, 0.0],
    [-0.0, -0.0, -0.00014506888966986438]
  ],
  "total_magnetization": null,
  "hartree_potential": null,
  "charge_density": null
}
````

</details>

## 8. Composite Workflows

Beyond relaxation workflows, the same schema architecture supports higher-level workflows that build on relaxation as a base. For example, the computation of the equation of state involves relaxing the same structure under different volumes, while the bond dissociation energy workflow involves the relaxation of molecular fragments. These are implemented as [composite workflows](https://aiida-common-workflows.readthedocs.io/en/latest/workflows/composite/index.html) in the AiiDA Common Workflows. We provide in the [source code repository](https://github.com/edan-bainglass/common-workflow-schemas) OO-LD-exportable Pydantic models for inputs/outputs of these composite workflows by extending the models of the common relaxation workflow. This demonstrates that all workflows, whether atomic or composite, benefit from semantic interoperability and the extensibility of the OO-LD approach.

## 9. Conclusion

The development of OO-LD-compliant schemas for common workflows in computational materials science represents a powerful and extensible solution to long-standing interoperability challenges. By combining the structural rigor of JSON Schema with the semantic capabilities of JSON-LD, we have created a flexible mechanism for defining, sharing, and interpreting input/output data across heterogeneous systems.

This approach, based on Python and Pydantic for developer accessibility and performance, ensures that schemas are both human-friendly and machine-actionable. The resulting documents support dynamic UI generation (future feature described in the next section), automated validation, and platform-independent semantic enrichment of data. The source code behind this work is open and available on [GitHub](https://github.com/edan-`bainglass/common-workflow-schemas).

This technical specification is a living document and will continue to evolve during the remaining lifetime of the PREMISE project. It aims to reflect the ongoing expansion of schema coverage to new workflows, enhancements in tooling, and feedback from real-world use in cross-platform materials discovery.

## 10. Future Work

This specification lays the groundwork for a broader ecosystem of tools and services around semantically-enriched schemas. In the short term, we plan to address open issues, such as automatic data/type coversions (leveraging tools such as [dlite](https://sintef.github.io/dlite/)) and string references not immediately captured by JSON-LD operations (e.g., [JSON-Schema `required` field](https://json-schema.org/understanding-json-schema/reference/object#required)).

In the long term, one major avenue for future work is the automatic generation of user interfaces (UIs) that enable researchers to build, inspect, and submit common workflow inputs using dynamic forms derived directly from OO-LD documents. These UIs would enhance usability and accessibility of complex workflows and help unify user interactions across different platforms.

Additional areas of exploration include schema-driven validation services, integration with lab hardware, and workflow translation services across engines via a standard workflow language schema. These will be explored in part in the PREMISE-sponsored [MADICES](https://madices.github.io/docs/2025/) workshops, where we will work with the community to develop a roadmap for the future of OO-LD and its applications in materials science.

## 11. Resources

- JSON Schema: [https://json-schema.org](https://json-schema.org)
- JSON-LD: [https://json-ld.org](https://json-ld.org)
- OO-LD: [https://github.com/OO-LD](https://github.com/OO-LD)
- Pydantic: [https://docs.pydantic.dev](https://docs.pydantic.dev)
- OPTIMADE: [https://www.optimade.org](https://www.optimade.org)
- PREMISE: [https://ord-premise.org](https://ord-premise.org)
- MADICES: [https://madices.github.io](https://madices.github.io)
- AiiDA: [https://aiida.net](https://aiida.net)
- AiiDA Common Workflows: [https://aiida-common-workflows.readthedocs.io](https://aiida-common-workflows.readthedocs.io)
- Source code repository: [https://github.com/edan-bainglass/common-workflow-schemas](https://github.com/edan-bainglass/common-workflow-schemas)

## 12. Acknowledgements

This work was carried out as part of the PREMISE project ([https://ord-premise.org](https://ord-premise.org)), an Establish Project of the ETH Board's ORD program. The authors thank [Dr. Simon Stier](https://www.isc.fraunhofer.de/en/fields-of-activity/applications/digital-transformation.html) of the Digital Transformation team at Fraunhofer ISC for his development of the OO-LD specification, collaborators and contributors from the AiiDA team, and members of the materials science community for valuable discussions and feedback throughout the development of these specifications.
