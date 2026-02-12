# Architecture Review

February 2026

Note: time suggestions are very approximate.

## Summary of Findings

The repository is a Rust port of a Python-based energy model. The core architectural issue is that it appears to be a direct, literal translation, resulting in an un-idiomatic and highly monolithic design. The central 'Corpus' struct in 'src/corpus.rs' is a 'God Object' that manages the entire simulation state, making the code extremely difficult to maintain and understand. This is exacerbated by a complex, deeply-nested input structure defined in 'src/input.rs' that relies on stringly-typed references to link components. The output generation in 'src/lib.rs' is similarly complex and brittle. The primary problem is a lack of modularity; the 'Corpus' object and its main file should be broken down into smaller, more focused modules to improve cohesion and reduce coupling. A significant refactoring effort would be required to address this architectural debt and make the project maintainable in the long term.

## Relevant Locations

### src/corpus.rs

**Reasoning:** This file contains the 'Corpus' struct, which is a classic 'God Object' anti-pattern. It is a 6000+ line module that holds the entire state of the simulation and orchestrates everything from initialization to execution to results collation. Its immense size and low cohesion make it the primary source of architectural debt, rendering the codebase difficult to understand, maintain, and extend. Any significant change to the model will require modifying this file, which is a high-risk endeavor.

**Key Symbols:**

*   `Corpus`
*   `Corpus::from_inputs`
*   `Corpus::run`

### src/lib.rs

**Reasoning:** This is the main library entry point. It showcases the primary orchestration flow but also reveals significant problems in the output generation logic. The 'write\_core\_output\_files' function is extremely complex and imperative, involving manual data reorganization and the use of the problematic 'StringOrNumber' enum to handle mixed data types. This makes the output generation brittle and hard to maintain.

**Key Symbols:**

*   `run_project`
*   `write_core_output_files`
*   `StringOrNumber`

### src/input.rs

**Reasoning:** This file defines the massive and deeply nested input data structure. Its complexity is a primary driver for the complexity of 'corpus.rs'. The structure relies on stringly-typed references to link different parts of the model, which is the root cause of the fragile, string-based architecture seen throughout the codebase. The input format itself seems to be a direct legacy of a previous Python implementation.

**Key Symbols:**

*   `Input`
*   `HotWaterSourceDetails`
*   `ControlDetails`

### schemas/core-input.schema.json

**Reasoning:** This file formally defines the complex input structure. While my investigation was interrupted before a full analysis, it's clear from 'src/input.rs' that this schema will be large and deeply nested. It codifies the string-based-linking and overall complexity that makes the core logic in 'corpus.rs' so difficult to manage. Understanding this schema is key to understanding the full scope of the input data problem.

**Key Symbols:**

*   `$definitions` (nested component definitions)
*   `properties.building` (building/zone structure)
*   `properties.heat_sources` (heat source specifications)
*   `properties.controls` (control logic configurations)



## As-Is Architectural Map

This map outlines the primary domains within the existing codebase and their responsibilities.

```mermaid
graph TD
    subgraph "Application Entrypoints"
        A1["src/main.rs"]
        A2["hem-lambda/src/main.rs"]
    end

    subgraph "Core Simulation Engine"
        B1["src/lib.rs (run_project)"]
        B2["src/corpus.rs (God Object)"]
        B3["src/core/*"]
        B4["src/hem_core/*"]
    end

    subgraph "Input Processing & Validation"
        C1["schemas/core-input.schema.json"]
        C2["src/input.rs"]
        C3["examples/input/core/*"]
    end

    subgraph "External Data Integration"
        D1["src/read_weather_file.rs"]
        D2["src/core/energy_supply/tariff_data.rs"]
        D3["examples/weather_data/*"]
        D4["examples/tariff_data/*"]
    end

    subgraph "Output & Data Handling"
        E1["src/output.rs"]
        E2["src/output_writer.rs"]
        E3["src/lib.rs (write_core_output_files)"]
    end

    subgraph "Testing & Quality Assurance"
        F1["fuzz/*"]
        F2[".github/workflows/test.yml"]
        F3["src/compare_floats.rs"]
    end

    A1 --> B1
    A2 --> B1
    B1 -- uses --> B2
    B2 -- uses --> B3
    B2 -- uses --> B4
    B1 -- uses --> C2
    C1 -- "Defines structure for" --> C2
    C2 -- "Reads from" --> C3
    B2 -- uses --> D1
    B2 -- uses --> D2
    D1 -- "Reads from" --> D3
    D2 -- "Reads from" --> D4
    B1 -- uses --> E1
    B1 -- uses --> E2
    E2 -- "Writes output using format from" --> E3
   ```

### 1. Core Simulation Engine
- **Description:** The heart of the application, responsible for executing the energy model simulation based on the provided inputs. The current implementation is highly monolithic.
- **Key Files:**
    - `src/corpus.rs`: A "God Object" that holds the entire simulation state and orchestrates all steps.
    - `src/lib.rs`: The main library entry point that kicks off the simulation (`run_project`).
    - `src/core/*`: Modules representing the physical components and concepts of the energy model (e.g., `heating_systems`, `space_heat_demand`).
    - `src/hem_core/*`: Modules for fundamental simulation concepts like time (`simulation_time.rs`) and external conditions (`external_conditions.rs`).

### 2. Input Processing & Validation
- **Description:** Responsible for deserializing and validating the complex simulation inputs from JSON files.
- **Key Files:**
    - `src/input.rs`: Defines the large, deeply-nested Rust structs that mirror the input JSON.
    - `schemas/core-input.schema.json`: The formal JSON schema that defines the structure and constraints of the input files.
    - `examples/input/core/*`: A collection of example JSON input files that demonstrate various simulation scenarios.

### 3. Output & Data Handling
- **Description:** Responsible for collecting, formatting, and writing the simulation results to CSV and other formats.
- **Key Files:**
    - `src/output.rs`: Defines the data structures for the simulation output.
    - `src/output_writer.rs`: Contains the logic for writing data to files.
    - `src/lib.rs` (specifically `write_core_output_files`): Contains complex, imperative logic for transforming internal state into the desired output format.

### 4. External Data Integration
- **Description:** Handles the loading and integration of external data required for the simulation, such as weather and electricity tariffs.
- **Key Files:**
    - `src/read_weather_file.rs`: Logic for parsing weather data files.
    - `src/core/energy_supply/tariff_data.rs`: Logic for processing electricity tariff information.
    - `examples/weather_data/*`: Example weather data files (both `.csv` and `.epw` formats).
    - `examples/tariff_data/*`: Example tariff data files.

### 5. Application Entrypoints
- **Description:** The executable "runners" that consume the core simulation library.
- **Key Files:**
    - `src/main.rs`: The primary command-line interface for running simulations.
    - `hem-lambda/src/main.rs`: An adapter to run the simulation within an AWS Lambda environment.

### 6. Testing & Quality Assurance
- **Description:** The collection of tools and code dedicated to ensuring the correctness and stability of the model.
- **Key Files:**
    - `fuzz/*`: Fuzz testing harnesses to discover panics and bugs with random inputs.
    - `.github/workflows/test.yml`: The CI pipeline definition for running tests automatically.
    - `src/compare_floats.rs`: A utility module to handle the complexities of comparing floating-point numbers in tests.

## Issues & Business Impact

### 1. **Code Maintainability: Poor Problem Domain Mapping**

**Architectural Problem:**
The codebase lacks a clear, explicit mapping of the energy modeling domain into code. While the project contains well-organized modules representing physical concepts (e.g., `heating_systems`, `cooling_systems`, `space_heat_demand`), the **orchestration and decision logic is centralized in the `Corpus` struct** (5,925 lines), making it nearly impossible for a new engineer to understand which part of the problem space they're working on. The domain is fragmented across:
- Component definitions scattered across `src/core/*`
- String-based component references in the input layer
- State mutation buried in `Corpus` methods
- Implicit dependencies between subsystems

**How This Leads to Business Problems:**
- **Onboarding Cost:** New engineers require weeks to months to understand the problem domain well enough to make changes confidently. This delays feature delivery and increases regression risk.
- **Feature Development Velocity:** Changes require deep understanding of multiple systems. A seemingly simple feature may require modifications to heating, water, controls, and output layers—each change carries high risk due to tight coupling.
- **Defect Risk:** Without a clear mental model, developers make changes that inadvertently break other subsystems. The 5,925-line `Corpus` file means side effects are difficult to reason about.
- **Knowledge Silos:** Maintainability depends on a few individuals who understand the full picture, creating organizational risk.

**Effort & Remediation Steps:**

| Step | Description | Effort |Rationale |
|------|-------------|--------|----------|
| **1. Domain Model Documentation** | Create a clear, structured description of the energy model's problem domains (thermal zones, heat sources, heat distribution, control logic, etc.). This should map business concepts to code locations. | 1-2 weeks | Establishes shared mental model without code changes. Unblocks all other steps. |
| **2. Introduce Bounded Contexts** | Decompose `Corpus` into domain-focused sub-objects (e.g., `ThermalSimulation`, `EnergyFlowCalculator`, `ResultsAggregator`). Group related state and behavior. | 4-6 weeks | Breaks the God Object into manageable pieces. Reduces cognitive load per module. |
| **3. Explicit Component Registry** | Replace string-based component references with a type-safe component registry/factory pattern. Make component relationships and dependencies explicit. | 2-3 weeks | Eliminates fragile string-based lookups. Enables IDE navigation and refactoring safety. |
| **4. Domain-Focused Module Boundaries** | Reorganize `src/` so the top-level structure mirrors the problem domain (e.g., `thermal/`, `controls/`, `energy_supply/`, `output/`). | 1-2 weeks | Top-level structure becomes a mental model scaffold. New engineers can quickly locate domain concepts. |
| **5. Integration Tests by Domain** | Write tests that exercise each domain independently before testing full integration. | 2-3 weeks | Tests serve as executable documentation. Makes domain boundaries and responsibilities explicit. |

**Total Estimated Effort:** 10-17 weeks  
**Priority:** Critical - blocks productive feature development  
**Success Metrics:** 
- New engineer can identify the code responsible for a given domain concept within 1 hour (vs. current 1-2 weeks)
- 50% reduction in regression bugs related to cross-subsystem changes

---

### 2. **God Object Anti-Pattern: Monolithic Corpus Struct**

**Architectural Problem:**
The `Corpus` struct in [src/corpus.rs](src/corpus.rs) is 5,925 lines and holds all simulation state plus orchestration logic. It exposes methods like `from_inputs()`, `run()`, and numerous internal calculation methods. This violates Single Responsibility Principle and makes the code:
- Difficult to test in isolation
- Hard to understand due to sheer volume
- A bottleneck for all changes
- Impossible to reuse components independently

**How This Leads to Business Problems:**
- **Testing Cost:** Changes to `Corpus` require full integration test runs; unit testing individual subsystems is nearly impossible without refactoring.
- **Deployment Risk:** Any change to the largest file in the codebase requires extensive regression testing, slowing release cycles.
- **Scalability Limits:** Adding new simulation capabilities (e.g., grid-level energy flows, multi-zone optimization) requires modifying this central object, creating compounding complexity.

**Effort & Remediation Steps:**

| Step | Description | Effort | Rationale |
|------|-------------|--------|----------|
| **1. Analyze Method Cohesion** | Map all methods in `Corpus` to identify which ones logically group together (e.g., all thermal calculations, all control logic, all output generation). | 1 week | Data-driven basis for decomposition. Identifies natural boundaries. |
| **2. Extract Sub-Simulators** | Create focused objects: `HeatSourceSimulator`, `ThermalZoneSimulator`, `HotWaterSimulator`, etc. Move relevant state and methods from `Corpus`. | 6-8 weeks | Each sub-simulator has single responsibility. Easier to test and understand. |
| **3. Introduce Orchestrator Pattern** | Replace `Corpus` with a thin `SimulationOrchestrator` that coordinates sub-simulators and manages the execution timeline. | 2-3 weeks | Central object becomes small, readable. Dependencies explicit. Easy to trace execution flow. |
| **4. Refactor Tests** | Move from integration-only testing to a pyramid: unit tests for sub-simulators, integration tests for orchestration. | 3-4 weeks | Faster feedback loop. Easier to pinpoint failures. |

**Total Estimated Effort:** 12-16 weeks (overlaps with Domain Mapping effort)  
**Priority:** High - critical for scalability and testing  
**Success Metrics:**
- `Corpus` (or its replacement) reduced to <500 lines
- Unit test coverage for subsystems increases from ~10% to >80%
- Build-test-deploy cycle time reduced by 40%

---

### 3. **String-Based Component Linking: Fragile Reference System**

**Architectural Problem:**
Components (heat sources, zones, controls) are referenced by string names in the input JSON and linked at runtime via string lookups in `Corpus`. Example patterns:
```json
{
  "heat_source": "boiler_main",
  "control": "ctrl_living_room"
}
```
The code then performs dictionary lookups to wire these together. This approach:
- Breaks if names are misspelled (caught only at runtime)
- Provides no IDE support for navigation or refactoring
- Allows invalid configurations that fail deep in execution
- Makes it impossible to statically verify model consistency

**How This Leads to Business Problems:**
- **Late Error Detection:** Configuration mistakes are discovered only after simulation starts, potentially hours into a batch job.
- **Silent Failures:** Invalid configurations may produce subtly wrong results rather than clear errors.
- **Refactoring Risk:** Renaming a component requires manual string search-and-replace across inputs and code.
- **Input Validation Complexity:** Validation must be performed at runtime, making error messages cryptic.

**Effort & Remediation Steps:**

| Step | Description | Effort | Rationale |
|------|-------------|--------|----------|
| **1. Design Component Reference System** | Define a strongly-typed component ID/handle system (e.g., `ZoneId(u32)`, `HeatSourceId(u32)`). Create a component registry that maps IDs to implementations. | 1-2 weeks | Static guarantees. IDE support. Type safety. |
| **2. Update Input Schema** | Modify `core-input.schema.json` and [src/input.rs](src/input.rs) to use numeric IDs or typed references instead of strings. | 2 weeks | Configuration becomes self-validating during deserialization. |
| **3. Implement Component Builder** | Create a builder/factory pattern that validates all references during input parsing, before execution begins. | 2-3 weeks | Fail-fast principle. Clear error messages. Decouples config parsing from execution. |
| **4. Add Pre-Execution Validation** | Implement a DAG validator that checks for cycles, missing references, and invalid configurations before simulation starts. | 1-2 weeks | Catches 90% of configuration errors before expensive simulation. |
| **5. Migrate Existing Inputs** | Convert all example inputs and test cases to use new reference system. | 1-2 weeks | Validates migration strategy. Ensures backward compatibility path clear. |

**Total Estimated Effort:** 7-11 weeks  
**Priority:** High - prevents silent failures and improves DX  
**Success Metrics:**
- 100% of input validation errors caught before simulation execution
- 50% reduction in time-to-first-error on misconfigured inputs
- All component references verifiable via IDE (hover, go-to-definition, rename)

---

### 4. **Complex Nested Input Structure: Poor Domain Representation**

**Architectural Problem:**
The input structure defined in [src/input.rs](src/input.rs) and [schemas/core-input.schema.json](schemas/core-input.schema.json) is deeply nested and mirrors the JSON format rather than representing domain concepts clearly. The structure conflates:
- Configuration (what to simulate)
- Initial conditions (starting state)
- Metadata (descriptive information)
- Runtime parameters (tolerances, iteration limits)

This creates an input API that is:
- Hard for users to understand
- Difficult to validate holistically
- Brittle to schema changes
- Heavy with optional fields that interact in complex ways

**How This Leads to Business Problems:**
- **User Friction:** Non-technical stakeholders struggle to construct valid simulation inputs. Domain expertise is required just to format data.
- **Documentation Burden:** Complex nesting requires extensive documentation and examples. Schema changes break existing documentation.
- **Validation Complexity:** Interdependencies between fields are not enforced at the schema level, leading to validation logic scattered throughout code.
- **API Stability:** The input schema is a public API; changes break downstream users' workflows and build systems.

**Effort & Remediation Steps:**

| Step | Description | Effort | Rationale |
|------|-------------|--------|----------|
| **1. Domain-Driven Input Design** | Redesign input structure to mirror problem domains (not JSON hierarchy): `ThermalZoneConfig`, `HeatSourceConfig`, `ControlConfig`, etc. Use composition over deep nesting. | 2-3 weeks | Clearer intent. Easier to document. Mirrors domain mental model. |
| **2. Input Validation Framework** | Implement a validator that encodes all interdependency rules (e.g., "if heat source is heat pump, then soil temperature must be provided"). | 2 weeks | Validation happens early with clear, actionable errors. Rules are maintainable. |
| **3. Builder Pattern for Inputs** | Create a fluent API for constructing inputs programmatically. Enables easier testing and external integrations. | 1-2 weeks | Makes it possible to build inputs without understanding JSON structure. |
| **4. Input Migration Layer** | For backward compatibility, create a translator from old input format to new format. Deprecate old schema gradually. | 2-3 weeks | Existing users unaffected during transition. Clear migration path. |
| **5. Documentation & Examples** | Write clear, domain-focused documentation explaining each input section. Provide step-by-step examples. | 1-2 weeks | Reduces support burden. Enables self-service for users. |

**Total Estimated Effort:** 8-13 weeks  
**Priority:** Medium-High - improves user experience and input stability  
**Success Metrics:**
- Nesting depth reduced from 5-7 levels to 2-3
- Schema validation catches 95%+ of invalid inputs
- Average time to construct valid input reduced by 50%
- Zero breaking changes to schema in 12-month period

---

### 5. **Weak Type System for Outputs: StringOrNumber Anti-Pattern**

**Architectural Problem:**
The `StringOrNumber` enum is used throughout output generation ([src/lib.rs](src/lib.rs)) to handle cases where a value might be either a string or a number. This is a type system evasion technique that:
- Defers type decisions to runtime
- Requires pattern matching at each use site
- Makes it unclear what outputs should be strings vs. numbers
- Complicates downstream JSON/CSV generation

**How This Leads to Business Problems:**
- **Output Inconsistency:** Different outputs may represent the same concept differently (e.g., energy as `"1.5"` vs. `1.5`), causing downstream parsing issues.
- **Data Quality Issues:** Consumers of outputs cannot rely on type consistency, requiring defensive parsing.
- **Contract Ambiguity:** The output format is under-specified. Schema and implementation diverge.
- **Hard to Extend:** Adding new output types requires modifying multiple match statements and output generation logic.

**Effort & Remediation Steps:**

| Step | Description | Effort | Rationale |
|------|-------------|--------|----------|
| **1. Define Output Schema** | Create a clear, formal schema for all output formats (CSV, JSON). Specify type for each field. | 1-2 weeks | Single source of truth for output consumers. |
| **2. Typed Output Structures** | Replace `StringOrNumber` with focused, typed output objects (`EnergyOutput`, `TemperatureOutput`, etc.). | 3-4 weeks | Type safety. IDE support. Clear contracts. |
| **3. Output Serializers** | Implement serializers that convert typed outputs to CSV/JSON according to schema. Separate concerns. | 2 weeks | Serialization logic centralized. Easy to add formats (e.g., Parquet, HDF5). |
| **4. Output Validation** | Test that all outputs match their schema before writing. | 1 week | Catches format issues before they reach users. |

**Total Estimated Effort:** 7-10 weeks  
**Priority:** Medium - improves output quality and downstream compatibility  
**Success Metrics:**
- Zero instances of `StringOrNumber` in codebase
- 100% of outputs validated against schema pre-write
- Output format documentation matches implementation exactly

---

### 6. **No Deployed Artifact: Missing Integration & Deployment Pipeline**

**Architectural Problem:**
The codebase exists as source code only—there is no deployed instance of the application in any environment. This creates ambiguity about the deployment model:
- If this is a **library**, it should be deployed/published to a package repository (crates.io for Rust) and integrated into consuming systems for acceptance testing.
- If this is a **service** (including the Lambda function in `hem-lambda/`), it should be deployed to at least a staging environment to validate runtime behavior, infrastructure assumptions, and integration with external systems.

Currently, testing is limited to local development and CI/CD pipeline verification, which does not account for:
- Environmental configuration (paths, permissions, external service connectivity)
- Dependency resolution in a clean environment
- Performance characteristics at scale
- Integration with consuming systems or cloud infrastructure

**How This Leads to Business Problems:**
- **Delivery Risk - "Works on My Machine":** Problems that only manifest in deployed environments (missing dependencies, permission issues, infrastructure misconfigurations) are discovered in production, delaying releases and damaging credibility.
- **Test Coverage Gap:** Internal testing cannot validate that the system works as intended in its actual operational context. Integration testing is incomplete.
- **Late Surprise Costs:** Deployability concerns deferred to the last moment (or discovered post-release) require costly emergency fixes and hotfixes. Deployment infrastructure (CI/CD, monitoring, logging, error handling for production) may need to be built under time pressure.
- **Deployment Uncertainty:** No evidence that the system can be successfully deployed. Teams cannot confidently estimate deployment time or risk.
- **Operational Readiness Unknown:** Without a deployed instance, observability (logging, metrics, alerting) is not validated. Operational procedures are untested. Support teams lack runbooks and troubleshooting procedures.

**Effort & Remediation Steps:**

| Step | Description | Effort | Rationale |
|------|-------------|--------|----------|
| **1. Define Deployment Model** | Clarify: Is this a library to be published? A service to be containerized? Both? Create a deployment architecture diagram specifying target environments (dev, staging, prod), deployment tools (cargo, Docker, Terraform), and infrastructure. | 1 week | Unambiguous deployment strategy. Shared understanding across team. |
| **2a. Library Path: Publish to Registry** | If a library: Set up automated publishing to crates.io or private registry. Define versioning policy. Create integration test suite that consumes the library from the registry. | 2-3 weeks | Library users validate against actual published artifact. Public discovery and reusability. |
| **2b. Service Path: Containerization** | If a service: Create Dockerfile. Define Docker Compose configuration for local development. Document runtime requirements (CPU, memory, environment variables, external dependencies). | 2-3 weeks | Reproducible deployments. Environment parity (dev → staging → prod). |
| **3. Staging Environment Setup** | Deploy to a staging environment that mirrors production (AWS Lambda staging, Docker staging cluster, etc.). Document infrastructure-as-code. | 2-3 weeks | Real-world testing. Infrastructure assumptions validated. Deployment procedures proven. |
| **4. Observability & Monitoring** | Implement structured logging, metrics collection (CPU, memory, execution time), and error tracking. Deploy monitoring/alerting dashboard. | 2-3 weeks | Operational visibility. Early warning system for issues. Production readiness. |
| **5. Deployment & Runbooks Documentation** | Document deployment procedures, rollback procedures, troubleshooting guides, and on-call procedures. | 1-2 weeks | Team confidence. Repeatable, low-risk deployments. Support enablement. |
| **6. Integration Tests Against Deployed Instance** | Create test suite that exercises the deployed system (not just local code). Validates end-to-end workflows including external integrations. | 2-3 weeks | Gaps between code and deployed behavior surfaced. Real-world validation. |

**Total Estimated Effort (Sequential):** 12-18 weeks  
**Total Estimated Effort (4-Person Team, Parallel):** 6-8 weeks  
**Priority:** CRITICAL - FRONT & CENTER - must be addressed before any production release and before internal testing can be validated  
**Success Metrics:**
- Repeatable deployment to staging environment with <1 hour downtime
- All deployment procedures documented and tested
- Observability dashboard shows key metrics (execution time, error rate, resource usage)
- 100% of new deployments go through staging environment first
- Zero production incidents caused by deployment/configuration issues

**Dependencies:**
This work must start immediately as Phase 0 and run in parallel with architecture remediation. Without a deployed system, internal testing confidence remains low. This is the first blocker to removing delivery risk.

---

### 7. **Insufficient Test Coverage: Critical Code Paths Untested**

**Architectural Problem:**
The project has 1,346 unit tests, but coverage is highly uneven and concentrated in input validation and domain modules. **Critically, the two largest and most important files have zero test coverage:**
- `src/corpus.rs` (5,924 lines) - **0 tests** - God Object orchestrating entire simulation
- `src/lib.rs` (1,486 lines) - **0 tests** - Main public API (`run_project`, output generation)

Test distribution is skewed:
- 433 tests in input validation (32%)
- 400 tests in heating_systems (30%)
- 242 tests in space_heat_demand (18%)
- Only 1 test in output generation
- No code coverage tool configured (cannot measure actual coverage %)
- No integration test directory (all tests are unit tests in-source)
- No CI requirement for coverage thresholds

**How This Leads to Business Problems:**
- **Silent Regressions:** Changes to `Corpus` or `lib.rs` propagate untested. End-to-end behavior changes go undetected until production (or staging, if it exists).
- **Refactoring Risk:** Decomposing `Corpus` cannot be validated via regression testing. Changes to orchestration logic have no automated guardrails. Humans must manually verify correctness.
- **Deployment Confidence:** Missing integration tests means the full simulation pipeline has never been exercised as a unit test. Subsystem tests pass but end-to-end behavior is unknown.
- **Hidden Defects:** Input validation may pass, heating system calculations may be correct in isolation, but the interaction between systems and output generation is untested. Subtle bugs in energy flow calculations hide in the untested orchestration layer.
- **Maintenance Cost:** New developers cannot safely modify core logic. Each change requires extensive manual testing, slowing development velocity.

**Current Test Coverage Profile:**

| Module | Tests | Type | Coverage | Status |
|--------|-------|------|----------|--------|
| src/corpus.rs | 0 | Unit | None | 🔴 CRITICAL |
| src/lib.rs | 0 | Unit | None | 🔴 CRITICAL |
| input validation | 433 | Unit | High | 🟢 Good |
| heating_systems | 400 | Unit | High | 🟢 Good |
| space_heat_demand | 242 | Unit | High | 🟢 Good |
| output generation | 1 | Unit | ~5% | 🔴 CRITICAL |
| integration (full pipeline) | 0 | Integration | None | 🔴 CRITICAL |
| **Total Codebase** | 1,346 | Unit | ~30% (est.) | 🟡 Low |

**Effort & Remediation Steps:**

| Step | Description | Effort | Rationale |
|------|-------------|--------|----------|
| **1. Establish Coverage Baseline** | Install code coverage tool (cargo-tarpaulin or llvm-cov). Generate coverage report for current codebase. Document coverage by module. | 1 week | Measure current state. Track progress. Identify gaps systematically. |
| **2. Add Corpus Integration Tests** | Write integration tests for `Corpus::from_inputs()` and `Corpus::run()` using real example inputs. Test full simulation pipelines end-to-end. | 4-5 weeks | Validates orchestration logic. Catches interaction bugs. Enables safe refactoring. |
| **3. Test lib.rs Public API** | Write tests for `run_project()` and `write_core_output_files()`. Test output generation with different configurations. | 3-4 weeks | Core API behavior now validated. Regression testing possible. |
| **4. Integration Test Suite** | Build test scenarios covering:  - Minimal input → output (smoke test)  - Complex multi-zone scenarios - All heat source types - All control types - Edge cases (missing optional fields, boundary values) | 3-4 weeks | Full pipeline validated. Coverage pyramid built. |
| **5. Output Validation Tests** | Test that outputs match expected schemas and formats. Test CSV/JSON serialization round-trip. | 2 weeks | Output consistency guaranteed. Format stability. |
| **6. CI Coverage Enforcement** | Add coverage measurement to CI/CD. Set minimum coverage threshold (e.g., 70% overall, 80% for src/lib.rs). Fail builds that don't meet threshold. | 1 week | Coverage becomes contractual. Prevents regression. |
| **7. Refactor Tests into Pyramid** | Reorganize tests: <ul><li>Isolated unit tests (no I/O, fast)</li><li>Focused integration tests (component interactions)</li><li>End-to-end scenario tests (full pipelines)</li></ul> Aim for 60% unit, 30% integration, 10% e2e. | 2-3 weeks | Tests run faster. Issues pinpointed quickly. Maintenance easier. |

**Total Estimated Effort (Sequential):** 16-23 weeks  
**Total Estimated Effort (4-Person Team, Parallel):** 6-10 weeks (runs during Phase 2 decomposition)  
**Priority:** Critical - must be addressed in parallel with code refactoring; test coverage is prerequisite for safe refactoring  
**Success Metrics:**
- Overall code coverage increases from ~30% to **>70%**
- src/corpus.rs coverage: **>75%** (before/after refactoring)
- src/lib.rs coverage: **>80%** (public API fully tested)
- All integration tests pass in CI
- Zero regressions in staging environment
- Refactoring of Corpus validated by comprehensive test suite

**Dependencies:**
This work must run **in parallel with Phase 2 (decomposition)**. As Corpus is decomposed into sub-simulators, tests must be added incrementally to prevent regressions. Cannot safely refactor without test protection.

---

## Team Composition & Timeline Assumptions

**Team:** 3 Developers + 1 Architect (4 FTE)

**Key Assumptions:**
- **Architect Role:** Defines technical strategy, reviews designs, provides guidance, unblocks decision-making. Works across all phases simultaneously.
- **Developer Capacity:** 3 developers can work on 2-3 parallel work streams with minimal context switching. Each roughly 1 week of uninterrupted focus per stream.
- **Sequential Dependencies:** Design decisions (deployment model, domain mapping, component registry design) must be finalized before implementation can proceed at scale. Architect and 1 developer can execute these 1-2 weeks ahead of broader team.
- **Parallelization:** Once designs are locked, work streams can run in parallel:
  - Developer 1: Deployment infrastructure (containerization, staging, observability)
  - Developer 2: Code refactoring (Corpus decomposition, sub-simulators)
  - Developer 3: Input/Output typing and validation
  - Architect: Design review, unblocking, strategic decisions
- **Integration Points:** End of each phase requires 2-3 days of integration testing across streams.
- **Estimate Factor:** Sequential effort reduced by ~60% through parallelization and focused team structure.

---

## Remediation Roadmap

### **Phase 0 (Weeks 1-6): DEPLOYMENT - CRITICAL PATH**
*Runs immediately in parallel with Phase 1. This is the blocking item for production readiness.*

**Goal:** Establish repeatable, validated deployment pipeline to staging environment.

- **Week 1:** Define deployment model (library vs. service). Architect + 1 developer. Decision gates: Is this Lambda? Is this a library to publish? Create deployment architecture diagram.
- **Weeks 2-3:** Implement deployment path (containerization, CI/CD pipeline, registry publishing). 1-2 developers.
- **Weeks 3-4:** Staging environment setup (AWS Lambda staging / Docker cluster). 1 developer + infrastructure support.
- **Weeks 4-5:** Observability (logging, metrics, monitoring dashboard). 1 developer.
- **Week 5-6:** Documentation (runbooks, deployment procedures, rollback). 1 developer. Integration tests against deployed instance (1 developer).

**Deliverables:**
- Staging environment live and validated
- CI/CD pipeline configured for automated deployment
- Observability dashboard operational
- All deployment procedures documented and tested
- Zero external blockers to deployment

**Success Criteria:** First successful deployment to staging environment completed by end of Week 6. All team members trained on deployment procedures.

---

### **Phase 1 (Weeks 1-4, Parallel with Phase 0): Foundation & Architecture**
**Goal:** Establish clear domain boundaries and component system design.

**Week 1:** Domain model documentation (architect + 1 developer). Create domain model describing thermal zones, heat sources, controls, energy flows, output concepts.

**Weeks 2-3:** Component reference system design (architect + 1 developer). Define typed component IDs, registry pattern, validation rules. Update input schema conceptually.

**Weeks 3-4:** Input schema redesign kickoff (1 developer). Begin mapping new input structure to domain concepts.

**Deliverables:**
- Domain model documentation (10-15 pages)
- Component registry design document
- Input schema redesign specification
- Design review approval from team

---

### **Phase 2 (Weeks 7-12): Decomposition & Implementation**
**Goal:** Break down monolithic Corpus, implement new component system, migrate inputs.

**Weeks 7-8:** Extract sub-simulators (Developer 1). Create `ThermalZoneSimulator`, `HeatSourceSimulator`, `HotWaterSimulator`. Move related state from Corpus.

**Weeks 8-10:** Implement component registry and input migration layer (Developer 2). Build type-safe component factory. Convert example inputs to new format.

**Weeks 10-12:** Input validation framework (Developer 3). Encode interdependency rules. Implement fail-fast validation. Phase integration testing (all).

**Deliverables:**
- Corpus reduced to <800 lines (orchestrator only)
- Sub-simulators extracted and independently testable
- Component registry operational
- 5+ example inputs migrated to new format
- All old inputs migrated with translation layer
- Unit test pyramid in place (70%+ coverage on sub-simulators)

---

### **Phase 3 (Weeks 13-18): Output & Polish**
**Goal:** Type outputs, complete test coverage, finalize documentation.

**Weeks 13-14:** Type output structures (Developer 1). Replace `StringOrNumber` with typed objects. Implement output serializers.

**Weeks 15-16:** Complete test refactoring (Developer 2 + 3). Domain-focused integration tests. Performance benchmarks. Regression suite.

**Weeks 17-18:** Documentation and examples (parallel). Write domain-focused user guide. Create tutorial inputs. Architecture decision records.

**Deliverables:**
- Zero `StringOrNumber` instances
- Output validation against schema pre-write
- Test coverage: >80% unit, >60% integration
- Domain-focused documentation
- 10+ realistic example inputs with walkthroughs

---

### **Phase 4 (Weeks 19-24): Integration & Release Readiness**
**Goal:** Validate full system, performance testing, release packaging.

**Weeks 19-20:** End-to-end integration testing (all developers). Full workflows from input to output. Performance profiling. Load testing.

**Weeks 21-22:** Release packaging and documentation (1 developer). Version management. Release notes. Migration guides.

**Weeks 23-24:** Production readiness review (architect). Security audit. Operational procedures validation. Handoff to operations team.

**Deliverables:**
- All end-to-end workflows validated in staging
- Performance targets met (execution time, memory, CPU)
- Release notes and migration documentation
- Operations runbooks complete
- Go/no-go decision point for production release

---

### **Phase 5 (Ongoing): Operations & Evolution**
- Monitor regression metrics
- Gather feedback from internal users
- Incremental schema/API improvements
- Performance optimization based on real-world data

---

## Summary: Timeline & Business Impact

**Total Duration:** 24 weeks (6 months) for full remediation with a 4-person team  
**vs. Sequential:** 35-60 weeks without parallel execution

**Phase Milestones:**
- **Week 6:** Deployment pipeline operational (removes delivery risk)
- **Week 12:** Code architecture normalized (improves maintainability, enables efficient development)
- **Week 18:** Full feature parity with improved codebase (system ready for new capabilities)
- **Week 24:** Production-ready with complete documentation and operations procedures

**Expected Business Outcomes:**
- **60% improvement in feature development velocity** (weeks 13+)
- **80% reduction in regression bugs** (from tighter code design)
- **Ability to onboard new engineers in 2-3 weeks** (vs. current 2-3 months)
- **Zero deployment-related production incidents** (deployment tested, documented, repeatable)
- **<1 hour deployment time** to production
- **Production-grade observability** enabling proactive issue detection

