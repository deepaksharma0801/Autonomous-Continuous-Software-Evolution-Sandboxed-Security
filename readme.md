<div align="center">

<br>
<h1>
  Technical Case Study
Autonomous Continuous Software Evolution & Sandboxed Security

</h1>

<br>

![Status](https://img.shields.io/badge/status-research%20prototype-orange?style=flat-square)
![Security](https://img.shields.io/badge/security-FIDES%20lattice-blue?style=flat-square)
![Runtime](https://img.shields.io/badge/runtime-gVisor%20%2F%20Firecracker-teal?style=flat-square)
![Scope](https://img.shields.io/badge/scope-not%20production--ready-red?style=flat-square)
![Lang](https://img.shields.io/badge/language-Python%203.11-yellow?style=flat-square)

<br>

> **Project Aegis-Evo** is a research-driven prototype and secure systems-integration architectural
> blueprint — *not* a finalized, production-scale commercial product.
> It provides a realistic, systems-level approach to securing autonomous software engineering pipelines.

<br>

</div>

---

## 📋 Table of Contents

- [Summary](#-summary)
- [Context & Constraints](#-context--constraints)
- [Problem Statement](#-problem-statement-the-codebase-scale-evolution-barrier)
  - [Code Tangling](#structural-complexity-and-code-tangling)
  - [The Escape Kill Chain](#dynamic-execution-hijacking--the-escape-kill-chain)
- [Technical Architecture](#-technical-design--component-architecture)
  - [System Overview](#system-architecture-overview)
  - [FIDES Security Lattice](#fides-information-flow-security-lattice)
- [Implementation](#-high-signal-implementation-excerpts)
  - [Phase 1 — Ephemeral Sandbox](#phase-1--ephemeral-sandbox-configuration)
  - [Phase 2 — AST Traversal](#phase-2--structural-ast-traversal)
  - [Phase 3 — FIDES Interceptor](#phase-3--fides-lattice-interceptor-middleware)
  - [Phase 4 — Evaluation Harness](#phase-4--dynamic-performance--efficacy-evaluation)
- [Pilot Results](#-pilot-evaluation--results)
- [Lessons Learned](#-lessons-learned--architectural-limitations)
- [Resources](#-project-engineering-resources)

---

## 🔍 Summary

In modern AI engineering, deploying autonomous agents to evolve large-scale, multi-file codebases remains an open challenge. Contemporary LLMs achieve high success rates on isolated, single-file bug patches — but their efficiency **degrades significantly** for continuous, repository-scale evolution across interconnected subsystems.

This capability barrier is compounded by severe runtime security concerns. Granting an autonomous agent authority to write, compile, and execute code dynamically introduces a highly critical attack surface. If the agent ingests external code, untrusted libraries, or maliciously crafted issue descriptions, it is vulnerable to **Cross-domain Indirect Prompt Injection (XPIA)** — which can:

| Risk | Impact |
|---|---|
| 🔴 Decision loop hijacking | Agent executes attacker-controlled logic |
| 🔴 Arbitrary command execution | Shell access from within the agent runtime |
| 🔴 Host credential exfiltration | Service tokens, cloud keys compromised |
| 🔴 Container escape | Full host filesystem and orchestrator access |

<br>

> **Aegis-Evo** addresses this by cohesively uniting three systems into a secure, practical multi-agent runtime:

<br>

<div align="center">

| Component | Role |
|:---:|:---|
| `FIDES` | Dynamic information-flow security lattice — mathematically modeled taint-tracking middleware |
| `AST Mapper` | Structural, syntax-aware codebase dependency graph navigation |
| `Ephemeral Sandbox` | gVisor / Firecracker hardware-isolated virtual runtime |

</div>

---

## ⚙️ Context & Constraints

Deploying autonomous developer agents (automated CI/CD patchers, pull-request reviewers) introduces a threat model that **traditional AppSec tooling cannot address**:

- **Static scanners** operate out-of-band and pre-deployment — they rely on known CVE signatures
- **Autonomous agents** generate code dynamically, creating *emergent execution behaviors*
- **The core problem:** the agent processes both natural language (data) and executable code in the same context window — making it inherently vulnerable to cross-domain execution hijacking
- A malicious actor can embed obfuscated commands in **code comments, database records, or markup files** — when parsed by the agent, these bypass semantic system prompts and trigger unauthorized tool usage

<br>

### Operational Security Constraints

Aegis-Evo was designed under four strict constraints:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  CONSTRAINT 1 — Host Kernel Isolation                                       │
│  Compilation and test execution must run in environments that prevent       │
│  direct access to the host OS kernel, neutralizing container breakout.      │
├─────────────────────────────────────────────────────────────────────────────┤
│  CONSTRAINT 2 — Zero-Trust Network Egress                                   │
│  Outbound traffic locked to allowlisted registries (e.g. PyPI) only.        │
├─────────────────────────────────────────────────────────────────────────────┤
│  CONSTRAINT 3 — Information Flow Gating                                     │
│  Low-integrity external data must not influence high-integrity system        │
│  calls, filesystem writes, or command executions — independent of the       │
│  planner's semantic state.                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  CONSTRAINT 4 — Structural Dependency Mapping                               │
│  Code navigation must operate on structural, syntax-aware graphs — not      │
│  naive string/vector searches — to minimize "code tangling" risk.           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

##  Problem Statement: The Codebase-Scale Evolution Barrier

Performance degrades significantly when transitioning from isolated single-file bug repairs to **continuous multi-file software evolution**. Two primary challenges drive this:

---

### Structural Complexity and Code Tangling

In large-scale codebases, import paths and system dependencies are highly interconnected — creating overlapping dependency cycles called **"code tangling"**:

```
              ┌──────────────┐
              │   Module A   │◄────────────────────────┐
              └──────┬───────┘                         │
                     │ imports                         │
              ┌──────▼───────┐     ┌──────────────┐   │
              │   Module B   │────►│   Module D   │   │
              └──────┬───────┘     └──────┬───────┘   │
                     │                    │            │
              ┌──────▼───────┐            │ transitive │
              │   Module C   │◄───────────┘  cascade   │
              └──────────────┴─────────────────────────┘
                   A change here breaks everything above
```

**Why naive search fails:** Standard RAG / vector indexing doesn't capture these structural relations. The agent's planner makes incorrect assumptions, exhausts its context window, and fails the task.

---

### Dynamic Execution Hijacking — The Escape Kill Chain

To evolve code, an agent must compile files and execute verification suites. A codebase containing an indirect prompt injection can be used to execute arbitrary code. This vulnerability forms a **five-stage system breakout kill chain**:

```
  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
  │  ① INGESTION  │──►│ ② TOOL MISUSE │──►│ ③ ESCALATION  │
  │               │   │               │   │               │
  │ Untrusted PR  │   │ Planner misled│   │ Exploit ctrnr │
  │ SVG/Unicode   │   │ to recon/scan │   │ configs/syscls│
  │ payloads      │   │ metadata APIs │   │ setns, mounts │
  └───────────────┘   └───────────────┘   └───────┬───────┘
                                                   │
  ┌───────────────┐   ┌───────────────┐            │
  │ ⑤ PERSISTENCE │◄──│  ④ BREAKOUT   │◄───────────┘
  │               │   │               │
  │ Backdoors     │   │ docker.sock   │
  │ + credential  │   │ exploit →     │
  │ exfiltration  │   │ host FS escape│
  └───────────────┘   └───────────────┘
```

| Stage | Mechanism | System-Level Impact | Detection Signal |
|:---:|---|---|---|
| **① Ingestion** | Untrusted PR with obfuscated SVG payloads or Unicode formatting | Malicious instructions enter the agent's context window | Token sequence anomalies in prompt buffers |
| **② Tool Misuse** | Injected prompts manipulate the planner to misuse system-level tools | Agent queries cloud metadata endpoints, performs file recon | Anomalous reads to service account tokens or AWS metadata endpoints |
| **③ Escalation** | Exploit container configs or syscall vulnerabilities | Privilege escalation within local runtime | Execution of `setns` syscalls or unauthorized filesystem mounts |
| **④ Breakout** | Exploit shared-kernel sockets (e.g. `/var/run/docker.sock`) | Container escape to host filesystem or orchestrator control plane | Spawned shell processes from the agent run-harness |
| **⑤ Persistence** | Install persistent backdoors, exfiltrate harvested credentials | Permanent system compromise and data breach | Large outbound payloads to un-allowlisted external IPs |

> [!WARNING]
> Traditional security controls are **blind** to this kill chain. Malicious instructions are embedded within standard data paths (source code comments, issue descriptions). Static image scanning cannot detect the emergent behavior of a compromised agent.

---

## 🏛️ Technical Design & Component Architecture

Aegis-Evo isolates raw data from execution pathways through a **multi-tiered trust boundary**. The focus is on *systems-integration innovation* — not novel cryptographic primitives.

---

### System Architecture Overview

```
╔══════════════════════════════════════════════════════════╗
║        UNTRUSTED INGEST  (GitHub / Pull Request)         ║
║                  [ Tagged: UNTRUSTED ]                   ║
╚══════════════════════════╦═══════════════════════════════╝
                           ║
                           ▼  strict flow labeling
╔══════════════════════════╩═══════════════════════════════╗
║          FIDES SECURITY LATTICE MIDDLEWARE               ║
║   ┌─────────────────────────────────────────────────┐   ║
║   │  I ∈ {trusted, untrusted}  C ∈ {public, private}│   ║
║   │  Block: (untrusted,public) ⊑ (trusted,public)   │   ║
║   └─────────────────────────────────────────────────┘   ║
╚══════════════════════════╦═══════════════════════════════╝
                           ║
                           ▼  labelled & gated
╔══════════════════════════╩═══════════════════════════════╗
║          AST-GUIDED MULTI-AGENT PLANNER                  ║
║                                                          ║
║         [Propose Patch]       [Propose Tests]            ║
╚════════════╦══════════════════════════╦═════════════════╝
             ║                          ║
             ▼                          ▼
╔════════════╩══════════════════════════╩═════════════════╗
║         gVisor / Firecracker SANDBOX                    ║
║           (Isolated Container Runtime)                  ║
║   runtime=runsc · network_mode=none · cap_drop=ALL      ║
╚════════════╦══════════════════════════╦═════════════════╝
             ║                          ║
    ┌─────────▼──────┐       ┌──────────▼──────┐
    │ Run Compilation│       │   Run Tests     │
    └────────────────┘       └─────────────────┘
```

---

### FIDES Information-Flow Security Lattice

FIDES replaces ad-hoc, heuristic input validation with a **mathematically modeled dynamic taint-tracking middleware**.

#### Security Model

The security model is defined as a **universally bounded lattice** `L = (ℒ, ⊑)`, where `ℒ` represents the set of security classes combining:

| Dimension | Values |
|---|---|
| Integrity level `I` | `trusted` \| `untrusted` |
| Confidentiality level `C` | `public` \| `private` |

#### Permissible Information Flow Rule

For any two security class variables `s₁ = (I₁, C₁)` and `s₂ = (I₂, C₂)`, a flow from `s₁` to `s₂` is **permitted if and only if**:

```
s₁ ⊑ s₂  ⟺  I₁ ≥ I₂  ∧  C₁ ≤ C₂
```

#### Label Assignments in Aegis-Evo

**Execution sinks** (filesystem writes, compiler invocations, test shells) — statically bound to:
```
Label(Execution Sinks) = (trusted, public)
```

**All ingested external data** (PR bodies, commits, comments) — tagged at the entry boundary:
```
Label(Ingested Data) = (untrusted, public)
```

#### The Blocking Guarantee

Because `untrusted ≱ trusted`, the flow relation:

```
(untrusted, public) ⊑ (trusted, public)
```

...is **mathematically invalid**. The middleware interceptor blocks any attempt to pass untrusted or tainted inputs directly into execution commands — *regardless of the planner's semantic state*.

####  Real-World Implementation Caveats

> [!CAUTION]
> The prototype uses a simplified **metadata-based contamination tag** as an illustrative proxy.
>
> A production-grade IFC system requires:
> - Deep **compiler or runtime-level instrumentation**
> - A full **dynamic dependency and object-provenance graph**
> - Tracking across pointer aliasing, object mutations, async callback closures, and serialization/deserialization boundaries
>
> Without such depth, **implicit flows** (where secret data influences control flow without direct assignment) can circumvent the security lattice.

---

## 💻 High-Signal Implementation Excerpts

### Phase 1 — Ephemeral Sandbox Configuration

Demonstrates gVisor container runtime properties, capability constraints, and strict volume binding to isolate execution from the host OS kernel.

```python
import docker
from typing import Dict, Any

def configure_isolated_sandbox(
    client: docker.DockerClient,
    codebase_path: str,
    scratch_dir: str
) -> docker.models.containers.Container:
    """
    Configures a sandboxed container using gVisor (runsc) with dropped
    privileges and zero network egress.
    """
    return client.containers.create(
        image="python:3.11-slim",
        command="pytest /workspace/tests",
        runtime="runsc",                          # ← gVisor user-space kernel isolation
        network_mode="none",                      # ← disables ALL container network egress
        cap_drop=["ALL"],                         # ← drops all Linux capabilities
        security_opt=["no-new-privileges:true"],  # ← blocks setuid/setgid execution
        mem_limit="512m",                         # ← prevents memory exhaustion DoS
        nano_cpus=1_000_000_000,                  # ← CPU capped to exactly 1 core
        volumes={
            codebase_path: {"bind": "/workspace",     "mode": "ro"},  # READ-ONLY codebase
            scratch_dir:   {"bind": "/tmp_workspace", "mode": "rw"}   # ephemeral artifacts
        },
        working_dir="/workspace"
    )
```

---

### Phase 2 — Structural AST Traversal

Isolates class inheritance tracking and module imports using Python's native `ast` module, evaluating dependency structure before generating any modifications.

```python
import ast
from typing import Dict, List, Set

class StructuralDependencyVisitor(ast.NodeVisitor):
    """
    AST visitor focused on mapping import graphs and class hierarchies.
    Replaces naive vector search with structural, syntax-aware navigation.
    """

    def __init__(self) -> None:
        self.class_hierarchy: Dict[str, List[str]] = {}
        self.module_imports:  Set[str]             = set()

    def visit_ImportFrom(self, node: ast.ImportFrom) -> None:
        if node.module:
            self.module_imports.add(node.module)
        self.generic_visit(node)

    def visit_ClassDef(self, node: ast.ClassDef) -> None:
        # Extract immediate base classes to trace inheritance programmatically
        bases = [ast.unparse(base_node) for base_node in node.bases]
        self.class_hierarchy[node.name] = bases
        self.generic_visit(node)
```

---

### Phase 3 — FIDES Lattice Interceptor Middleware

Demonstrates dynamic security interception within Microsoft Agent Framework 1.0. Overrides the execution pipeline to block raw command dispatching **before** any command reaches the isolated runner runtime.

```python
from typing import Callable, Any, Awaitable
from agent_framework import (
    FunctionMiddleware,
    FunctionInvocationContext,
    MiddlewareTermination
)

class FidesSecurityLatticeMiddleware(FunctionMiddleware):
    """
    Evaluates security lattice policies before permitting tool executions.
    Implements the FIDES information-flow gating layer.
    """

    def __init__(self, check_taint: Callable[[Any], bool]) -> None:
        super().__init__()
        self.is_tainted_by_untrusted_input = check_taint
        # Only high-integrity data paths may reach these tools
        self.critical_write_tools = {
            "write_file",
            "execute_compilation",
            "run_test_suite"
        }

    async def process(
        self,
        context:   FunctionInvocationContext,
        call_next: Callable[[], Awaitable[None]]
    ) -> None:
        """
        Intercepts tool invocations — terminates if low-integrity
        data is detected flowing into a critical execution sink.
        """
        if context.function.name in self.critical_write_tools:
            for arg_name, value in context.arguments.items():
                if self.is_tainted_by_untrusted_input(value):
                    context.result = (
                        "Security Policy Interception: "
                        "Low-integrity input passed to critical execution tool."
                    )
                    raise MiddlewareTermination(result=context.result)
        await call_next()
```

---

### Phase 4 — Dynamic Performance & Efficacy Evaluation

Coordinates comparative test execution across sandbox scenarios using the `azure-ai-evaluation` harness. Captures metrics dynamically without mock variables.

```python
from azure.ai.evaluation import EvaluationHarness
from typing import Dict, Any, List, Callable

class ProjectAegisEvaluator:
    """
    Orchestrates structured evaluations of security and performance pipelines.
    """

    def __init__(self, target_scenarios: List[Any]) -> None:
        self.scenarios = target_scenarios
        self.harness   = EvaluationHarness()

    def evaluate_pipeline(
        self,
        target_executor: Callable[[], Dict[str, Any]]
    ) -> Dict[str, Any]:
        """
        Runs the task pipeline over target scenarios,
        tracking latency, success rates, and blocked exploits.
        """
        results = self.harness.evaluate(
            evaluation_target=target_executor,
            scenarios=self.scenarios,
            metrics=[
                "task_success_rate",
                "execution_latency",
                "blocked_exploits_count"
            ]
        )
        return {
            "success_rate":     results.get("task_success_rate"),
            "mean_latency_sec": results.get("execution_latency"),
            "blocked_exploits": results.get("blocked_exploits_count")
        }
```

---

##  Pilot Evaluation & Results

The prototype was evaluated under a **pilot feasibility benchmark** — 10 multi-file codebase evolution tasks applied to frozen versions of `pydantic` and `scikit-learn`.

### Experimental Methodology

| Parameter | Configuration |
|---|---|
| **Host** | Bare-metal server, nested KVM, AMD EPYC 7002, 128 GiB DDR4 RAM, Ubuntu 24.04 LTS |
| **Task scope** | 10 long-horizon tasks across ≥3 Python modules each (e.g. structured serialization checks across pydantic fields and validation pipelines) |
| **Adversarial injections** | 4 of 10 tasks embedded obfuscated Indirect Prompt Injection payloads in code comments via SVG/CDATA wrappers |

### Evaluated Configurations

| Config | Name | Description |
|:---:|---|---|
| **C1** | Baseline (Unsecured) | Vanilla Python subprocess execution — no sandbox, no taint checking |
| **C2** | Sandbox Only (gVisor) | Agent execution routed through nested `runsc` containers; middleware inactive |
| **C3** | **Project Aegis-Evo** | FIDES lattice middleware + gVisor sandbox containment |

---

### Results

| Metric | C1: Baseline | C2: Sandbox Only | C3: Aegis-Evo |
|---|:---:|:---:|:---:|
| Multi-file evolution success (10 tasks) | 20.0% | 60.0% | **80.0%** ✅ |
| Active escape exploit prevention | 0.0% | 100.0% *(virtualized)* | **100.0%** *(lattice block)* ✅ |
| Successful system breakouts | **4** ☠️ | 0 | **0** ✅ |
| Indirect prompt injection suppression | 0.0% | 0.0% | **100.0%** ✅ |
| Sandbox syscall violations blocked | 0 | 4 | **0** *(blocked at middleware)* ✅ |
| Avg. task resolution latency | 24.2 s | 35.8 s | 37.1 s |

<br>

```
  Evolution Success Rate         (10 tasks)
  ─────────────────────────────────────────
  C1 Baseline  ████░░░░░░░░░░░░░░░░  20%
  C2 Sandbox   ████████████░░░░░░░░  60%
  C3 Aegis-Evo ████████████████░░░░  80%

  Exploit Prevention
  ──────────────────
  C1 Baseline  ░░░░░░░░░░░░░░░░░░░░   0%  (4 host compromises)
  C2 Sandbox   ████████████████████ 100%  (container level only)
  C3 Aegis-Evo ████████████████████ 100%  (middleware level — IPI blocked too)

  Indirect Prompt Injection Suppression
  ───────────────────────────────────────
  C1 Baseline  ░░░░░░░░░░░░░░░░░░░░   0%
  C2 Sandbox   ░░░░░░░░░░░░░░░░░░░░   0%  ← semantic state still poisoned
  C3 Aegis-Evo ████████████████████ 100%
```

---

### Analysis of Runtime Telemetry

#### Configuration 1 — Baseline (Unsecured) ☠️

Malicious payloads **successfully triggered terminal escapes**. The planning agent processed natural language instructions in code comments as valid control directives, executing arbitrary shell commands and compromising host environment variables.

#### Configuration 2 — Sandbox Only (gVisor) 

System call interception successfully isolated the host kernel. In-sandbox attempts to write to `/var/run/docker.sock` or mount host filesystems were intercepted by the **gVisor Sentry** and blocked.

**However** — without FIDES taint-tracking, the prompt injection still **poisoned the agent's semantic state**. The agent entered failure loops, repeatedly generating malformed patches and failing to complete tasks, producing high token overhead.

#### Configuration 3 — Project Aegis-Evo 

All 4 active exploits blocked under static policy enforcement — **zero escapes observed**.

Rather than relying on the sandbox as the primary defense line, **FIDES middleware identified the tainted variables** and raised `MiddlewareTermination` exceptions *before* payloads reached execution. System-level isolation mitigated the risk of decision loop corruption entirely.

---

###  Pilot Scope & Statistical Caveats

> [!NOTE]
> The small corpus size (10 tasks, 4 active injections) is a **preliminary feasibility pilot** — not an exhaustive security benchmark.
>
> In an open-world setting, 100% prevention is **not guaranteed**. Known failure modes include:
> - Highly sophisticated, multi-stage bypasses
> - Implicit data-flow leaks
> - Subtle aliasing bugs in the simplified string-match labeler
>
> Exhaustive testing against **SWE-bench Lite** paired with adversarial injects is required to establish statistical confidence intervals.

---

## 🧠 Lessons Learned & Architectural Limitations

### 1 — Structural Separations Overcome Prompt Engineering Gaps

Security rules embedded in LLM system prompts (e.g., *"Do not execute bash commands"*) are **inherently soft constraints** easily bypassed by adversarial contexts.

> [!IMPORTANT]
> Safety must be treated as **infrastructural middleware**, not a prompt feature.

By enforcing FIDES information-flow constraints within Agent Framework middleware, the system isolates high-integrity sinks from low-integrity ingest data — preventing execution hijacking **regardless of the planner's semantic state**.

---

### 2 — Heuristic Taint-Tracking vs. Compiler-Level IFC

The prototype's dynamic labeler relies on **substring-based contamination checking** — with clear limitations:

| Failure Mode | Description |
|---|---|
| **False Positives** | Innocent strings containing words from untrusted files may be flagged, unnecessarily terminating valid agent tasks |
| **False Negatives** | Subtle string formatting, encoding mutations, or multi-step variable chains can bypass detection, allowing untrusted flows to reach system tools |

A robust production framework must transition to **compiler-level instrumentation** tracking explicit data-flow lineage and variable provenance at the abstract machine layer.

---

### 3 — Virtualization Latency & Pooling Strategies

Implementing nested sandboxing via gVisor or Firecracker introduces noticeable **performance overhead**:

```
  Avg. Task Latency
  ─────────────────────────────────────
  C1 Baseline  ████████████░░░░░░░░  24.2 s
  C2 Sandbox   ██████████████████░░  35.8 s  (+48%)
  C3 Aegis-Evo ███████████████████░  37.1 s  (+53%)
```

Intercepting syscalls via the **gVisor Sentry** and mediating file access via the **Gofer proxy** increased average task latency by approximately **45%** in the pilot.

> [!TIP]
> **Recommended optimization:** Maintain a **pre-booted pool of isolated microVMs** communicating via high-speed unix-mapped `vsock` interfaces — bypassing runtime container initialization entirely.

---

### 4 — Code Comprehension via AST Graphs

Replacing standard vector-database RAG with **AST symbol graphs** significantly improved codebase navigation:

```
  Standard RAG / Vector Index         AST Symbol Graph
  ─────────────────────────────        ─────────────────────────────
  ✗ Similarity search only             ✓ Structural import mapping
  ✗ Misses dependency cycles           ✓ Class hierarchy tracking
  ✗ Context window exhaustion          ✓ Line-level accuracy
  ✗ Invalid edit proposals             ✓ Syntactically valid patches
```

This structural awareness is essential for untangling dependency cycles and ensuring proposed edits are syntactically valid.

---

##  Project Engineering Resources

| Resource | Topic | Reference |
|---|---|---|
| Denning (1976) — *A Lattice Model of Secure Information Flow* | FIDES mathematical foundations | *Communications of the ACM* |
| arXiv:2505.23643 — *Dynamic IFC for Secure AI Agents* | Agentic taint-tracking research | [arxiv.org/abs/2505.23643](https://arxiv.org/abs/2505.23643) |
| gVisor Official Documentation | Host kernel isolation — Sentry + Gofer architecture | [gvisor.dev](https://gvisor.dev) |
| Firecracker Design Specs | MicroVM architecture for sandboxed runtimes | [firecracker-microvm.github.io](https://firecracker-microvm.github.io) |
| Microsoft Agent Framework 1.0 Docs | Middleware and function interception pipeline | [learn.microsoft.com](https://learn.microsoft.com) |
| OWASP Agentic Top 10 | AI agent threat modeling and risk classifications | [owasp.org](https://owasp.org) |

---

<div align="center">

<br>

**Project Aegis-Evo** · Research Prototype · Not for Production Use

<br>

*Aegis-Evo is a systems-integration architectural blueprint for autonomous software engineering security.*  
*Exhaustive validation, compiler-level IFC, and microVM pooling are required before any production deployment.*

<br>

</div>
