# Documentation

Five documents, in the order they are worth reading.

| | What it covers | Read it when |
|---|---|---|
| [01 - High-level design](01-high-level-design.md) | What the system is, why it is split the way it is, how a message travels end to end | First. Everything else assumes it |
| [02 - Low-level design](02-low-level-design.md) | Inside each service: the mechanics that are easy to get wrong, and the bugs that proved it | You are about to change a service |
| [03 - Data model](03-data-model.md) | Every table and collection, and why each column exists | You are adding a field or a query |
| [04 - Event catalogue](04-events.md) | Every Kafka topic and event, its payload, who consumes it, and the consumer-group strategy | You are adding a producer or a consumer |
| [05 - Testing strategy](05-testing-strategy.md) | Four layers, what each one is for, how the two QA roles divide the work | You are writing a test, or wondering why one exists |

Operational instructions - how to run it, the ports, the troubleshooting - are in the
repository's top-level `README.md` rather than here, so there is one place to look for them.
The CI and CD pipelines are in `.github/workflows/` and the Kubernetes manifests in
`infra/k8s/`, both commented in the same style as these documents.

## A note on how these are written

They record **why**, not **what**. What the code does is readable from the code, and a document
that restates it is wrong within a month and nobody notices.

What is not recoverable from the code is the reasoning: which alternatives were considered, what
each one cost, and which failure a particular line is guarding against. That is what these
contain, including the parts that did not work - several sections exist because something broke,
and the explanation of why is more useful than the fix.

They also say plainly what is **not** built. A document that quietly implies completeness is
worse than one that lists the gaps, because the gaps are what a reader most needs to know before
relying on something.
