# Architecture

Three figures, each carrying one claim. A boundary, a loop, and a gate.

A single stack diagram of the components was deliberately not drawn: it would be
an inventory, and a reviewer who wants an inventory can read `ls src/app` faster.

---

## 1. The inheritance boundary

The central claim of a golden path is that governance arrives by default rather
than by discipline. That claim lives entirely in where this line falls.

```mermaid
flowchart TB
    subgraph S["your-service &nbsp;·&nbsp; built from ai-golden-path@v1.0.0"]
        direction TB
        subgraph W["YOU WRITE"]
            direction LR
            W1["API routes"] ~~~ W2["model + retrieval calls"] ~~~ W3["guardrail implementations"]
            W4["evaluators + thresholds"] ~~~ W5["readiness checks"] ~~~ W6["service ADRs"]
        end
        subgraph I["INHERITED &nbsp;—&nbsp; present before any business logic exists"]
            direction LR
            I1["typed config"] ~~~ I2["structured logging"] ~~~ I3["correlation ids"] ~~~ I4["health + readiness"]
            I5["OTel tracing"] ~~~ I6["error contract"] ~~~ I7["guardrail contract"] ~~~ I8["eval harness"]
            I9["hardened container"] ~~~ I10["CI gate"] ~~~ I11["ADR discipline"] ~~~ I12["version pinning"]
        end
        W ~~~ I
    end

    classDef write fill:#F7EDDA,stroke:#96650F,color:#6E4A0B
    classDef inherit fill:#E1EAF3,stroke:#2B5F8C,color:#1F4568
    class W1,W2,W3,W4,W5,W6 write
    class I1,I2,I3,I4,I5,I6,I7,I8,I9,I10,I11,I12 inherit
    style S fill:#FBFCFD,stroke:#B9C2CC
    style W fill:#FFFFFF,stroke:#96650F
    style I fill:#FFFFFF,stroke:#2B5F8C
```

**Twelve of the eighteen boxes arrive for free.** A team starting here writes
routes, model calls, guardrail implementations, evaluators, readiness checks and
its own ADRs. Everything else — including the entire security and observability
posture — is inherited the moment the repository is created.

Both planes reach code the team writes, not only code it inherits: a handler's
own log line carries the correlation id without the handler doing anything.

Nothing stops a team going around the template, and that is deliberate. The
paved road is made easier than the alternative rather than mandatory, because
mandates produce shadow implementations and incentives produce adoption.

Redraw this figure for each service built from the template. The ochre boxes
change; the blue ones never do. That repetition is the argument.

---

## 2. One request, two identifiers

Why the template carries both a correlation id and a trace id is the hardest
thing here to explain in prose.

```mermaid
flowchart LR
    R["request arrives<br/><small>header, or nothing</small>"] --> C["correlation id<br/><small>accepted, else uuid4</small>"]
    C --> T["trace span opened<br/><small>trace_id + span_id</small>"]
    T --> H["handler runs<br/><small>your code</small>"]

    classDef corr fill:#E1EAF3,stroke:#2B5F8C,color:#1F4568
    classDef trc fill:#F7EDDA,stroke:#96650F,color:#6E4A0B
    classDef neutral fill:#F2F4F7,stroke:#606B79,color:#2A323B
    class C corr
    class T trc
    class R,H neutral
```

Both identifiers are then stamped onto everything the request produces:

```mermaid
flowchart LR
    C["correlation id"] --> HDR["response header<br/><small>x-correlation-id</small>"]
    C --> ERR["error response<br/><small>RFC 9457 problem+json</small>"]
    C --> LOG["every log record"]
    T["trace_id + span_id"] --> LOG
    T --> TB["trace backend<br/><small>vendor-neutral OTLP</small>"]

    classDef corr fill:#E1EAF3,stroke:#2B5F8C,color:#1F4568
    classDef trc fill:#F7EDDA,stroke:#96650F,color:#6E4A0B
    classDef both fill:#F2F4F7,stroke:#2B5F8C,color:#1F4568
    class C,HDR,ERR corr
    class T,TB trc
    class LOG both
```

They answer different questions. The **correlation id** is business-facing and
quotable — accepted from the caller, echoed on the response, present in every
error body. The **trace id** is what an observability backend indexes on.

Carrying only one breaks the support loop:

```mermaid
flowchart LR
    U["user quotes an id<br/><small>off their error screen</small>"] --> S["search the logs<br/><small>by correlation_id</small>"]
    S --> L["the line carries trace_id<br/><small>stamped at record creation</small>"]
    L --> TR["open the trace<br/><small>every hop, timed</small>"]

    classDef loop fill:#E1EAF3,stroke:#2B5F8C,color:#1F4568
    class U,S,L,TR loop
```

A trace id alone breaks this at step one — nobody reads 32 hex characters off a
screen. A correlation id alone has no call graph behind it.

**"Stamped at record creation" is load-bearing.** Reading the context variable in
the formatter looks equivalent and fails silently under a `QueueHandler`, because
formatting then happens on another thread after the request has finished. We
shipped that bug and it is written up in
[ADR-0003](adr/0003-structured-logging.md), along with the regression test.

See also [ADR-0008](adr/0008-tracing.md) and [ADR-0009](adr/0009-error-contract.md).

---

## 3. What a change has to survive

"CI is a gate, not a report" is a sentence anyone can write. This is the version
that can be checked.

```mermaid
flowchart TB
    CH["a change"] --> Q["quality<br/><small>ruff · ruff format · mypy --strict</small>"]
    CH --> TE["test<br/><small>56 tests · coverage floor 70%</small>"]
    CH --> SE["security<br/><small>pip-audit on the lockfile · gitleaks over history</small>"]
    CH --> BU["build<br/><small>image builds, and actually boots</small>"]

    Q --> CI{{"CI &nbsp;—&nbsp; the one required check"}}
    TE --> CI
    SE --> CI
    BU --> CI

    CI -->|"any job fails"| X["blocked<br/><small>merge button disabled</small>"]
    CI -->|"all four pass"| M["merge to main<br/><small>protected branch</small>"]
    M -.->|"each service adds"| EV["evaluation gate<br/><small>groundedness · pass rate · regression vs baseline</small><br/><small>p95 latency · error rate</small>"]

    classDef neutral fill:#F2F4F7,stroke:#606B79,color:#2A323B
    classDef job fill:#E1EAF3,stroke:#2B5F8C,color:#1F4568
    classDef stop fill:#F7E3E0,stroke:#9E3F36,color:#7A2F28
    classDef svc fill:#F7EDDA,stroke:#96650F,color:#6E4A0B
    class Q,TE,SE,BU,CI job
    class X stop
    class EV svc
    class CH,M neutral
```

**Branch protection points at one aggregate check, not at four.** If all four
were listed as required and someone later added a fifth job, the protection rule
would silently stop covering it. One check that `needs` the rest cannot drift.

The template ships the evaluation harness and the thresholds contract. It ships
no metrics — groundedness belongs to a retrieval service, task success to an
agent runtime ([ADR-0011](adr/0011-evaluation-harness.md)).

This gate has rejected a push twice.

See [ADR-0007](adr/0007-ci-pipeline.md).
