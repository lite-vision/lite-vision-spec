# LV-SPEC-0202 — Intelligence Kernel Interface (Rust ABI + Sandbox)
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0200, LV-SPEC-0201  
Language Target: Rust  

---

# 1. Purpose

This specification defines the Intelligence Kernel Interface:

- The Rust ABI contract for kernel execution
- State and message input/output structure
- Versioning and reproducibility requirements
- Sandbox isolation models (WASM / native)
- Resource caps and enforcement rules

An Intelligence Kernel is a deterministic or probabilistic compute module that:

- Accepts structured state and inputs
- Produces structured outputs (RPACK or intermediate artifacts)
- Executes within defined resource constraints
- Operates under economic accountability

The Truth Plane never executes kernels.

---

# 2. Design Goals

The Kernel Interface MUST:

1. Be language-agnostic at the binary boundary
2. Be deterministic when required
3. Be sandboxable
4. Enforce resource limits
5. Be versioned and hash-addressable
6. Support GPU acceleration outside consensus

---

# 3. Kernel Identity

Each kernel is uniquely identified by:

KernelID = H(kernel_binary || kernel_metadata)

Kernel metadata includes:

- semantic_version
- deterministic_mode_supported (bool)
- input_schema_hash
- output_schema_hash
- resource_profile_hash

KernelID MUST be immutable once published.

---

# 4. Kernel Trait Interface (Rust)

All kernels MUST implement the following trait (conceptual definition):

```rust
pub trait IntelligenceKernel {
    type Input;
    type Output;
    type State;
    type Message;

    fn init(config: KernelConfig) -> Result<Self, KernelError>;

    fn execute(
        &mut self,
        state: Self::State,
        input: Self::Input,
        messages: Vec<Self::Message>,
        context: ExecutionContext,
    ) -> Result<KernelResult<Self::State, Self::Output>, KernelError>;

    fn version() -> KernelVersion;
}