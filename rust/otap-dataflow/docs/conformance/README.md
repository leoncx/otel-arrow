# Engine conformance

From `rust/otap-dataflow`, run the candidate on your working tree:

```sh
cargo xtask conformance --open
cargo xtask conformance --component urn:otel:receiver:syslog_cef
cargo xtask conformance --base origin/main --metrics target/live.json --html target/live.html
```

## How the checker works

The validation library starts a native engine. The conformance driver discovers
registered factories through linkme and selects the same YAML rollout fixture
for a full run or a single URN. Each fixture changes a running pipeline and
checks its effect on traffic. An independent observer must keep delivering
through its original generation. The suite also checks identical updates,
rejected configurations and rollback after an accepted replacement fails to start.

The report names failed operations first. A failure stops later stages of that
fixture. Unlinked components, missing fixtures and unmet prerequisites remain
visible. Controller extensions get startup configuration preflight. The
[check catalog](checks.yaml) describes each observation. The generated
[support table](live-support.md) lists known gaps and their required environment.

Use `--base REF` to identify new nodes without rollout evidence and lost
coverage. Documented inherited gaps remain visible. The command writes requested
reports before exiting unsuccessfully for a regression. Local runs require the
normal Rust build prerequisites. Python and Docker are unnecessary.

### Checking one component

Use the full registered URN to check its factory's rejection of invalid
configuration and its add, change, and remove fixture. The observer pipeline
and delivery checks run at each stage. Focused JSON reports include
`behavior_checks` and `deployment_gaps` with an owner and follow-up.

```sh
cargo xtask conformance --component urn:otel:processor:transform
cargo xtask conformance --component urn:otel:exporter:otlp_http --metrics target/otlp-http.json
```

A controller extension receives startup configuration validation. For a
pipeline component, a missing fixture, missing prerequisite or unlinked
factory makes the focused check exit unsuccessfully and explains the gap.
Unknown URNs are rejected against the source inventory. Use the full suite
to review a change against a base revision.

Each catalog fixture also reapplies the changed configuration with the subject
installed. The admin operation must complete as a no-op, retain the committed
generation and pass the normal delivery probe. A focused report records
`reconfiguration-noop` when the fixture reaches this step. A failed add leaves
the no-op in `unexercised_checks` and later operations unexercised.
Independent behavior checks still run.

### Add a live fixture for a component

Choose an existing probe that can observe the operation. Add a YAML entry
keyed by the factory URN with the component configuration and the change to
exercise. Remove its prerequisite-only entry when executable coverage replaces
the gap. The source inventory supplies its row automatically. Run the focused
check and regenerate the operator table with `--write-support`.

Add one entry to the conformance crate's [fixture catalog][fixtures]. The
runner matches its URN against the production factory maps and applies one
lifecycle to each linked component. Each fixture gets an isolated engine
with a target and an observer pipeline. The runner adds the subject, changes
its configuration, and removes it. Each step checks the committed topology,
the observer's generation and delivery, and the subject's data-plane effect.
The rejected-update regression checks that the committed generation keeps serving
after an invalid graph and after startup-owned changes, including a request
to install an internal telemetry receiver in the observability pipeline.

For example, the transform processor reuses the same marked-log capture as
the attribute processor. The catalog entry supplies its query and assertion.

```yaml
- urn: urn:otel:processor:transform
  probe:
    kind: log-attribute
    key: guarantee.phase
  config:
    opl_query: 'logs | set attributes["guarantee.phase"] = "{{ phase }}"'
```

For a processor that should pass a marked log, use `probe: {kind: pipeline,
role: processor}`. Its default graph is `input -> subject -> out`.
Use `peers` and `connections` to supply a producer for a receiver, a consumer
for an exporter, or both halves of a protocol bridge. The peer and subject
use ordinary DFE node configurations. Declare required local services, such as
`services: [kafka]`, on the fixture. Templates expand only in string values.
Use `{{ phase }}` for the add or change value, `{{ address }}` for a reserved
loopback address, and `{{ dir }}` for the session directory. Use `change_config`
when the second configuration changes fields or typed values.
For the configuration-free noop exporter, the change enables its node-level
message telemetry policy.

| Probe | What an add or change must demonstrate |
| --- | --- |
| `pipeline` | The declared graph delivers the marked record, transformed body, named metric or phase-specific signal. Sinks can require a downstream OTLP acknowledgement or refusal, a Parquet row, an authenticated upload or a ClickHouse ArrowStream insert. |
| `syslog-tcp` | The listener delivers a marked record and its retired address refuses connections. |
| `log-attribute` | A fresh record carries the configured attribute value. After removal it arrives without that attribute. |
| `jsonl-file` | Each marked log appears at its active destination. The original destination resumes after removal. |
| `otlp-http-header` | The capture receives the configured test header on the marked request and observes its absence after removal. |
| `otlp-authorization` | The receiver rejects an unauthorized token, delivers a log for the allowed audience and accepts an unauthenticated request after unbinding. |

The fixtures use an in-process Kafka broker, loopback HTTP and TLS peers,
disposable credentials, and temporary files. Azure and Google token services,
Kubernetes TokenReview, and both Geneva endpoints run locally. Linux fixtures
also exercise a synthetic procfs, a two-record journal and a private CA trust
file. The journald receiver needs `libsystemd.so.0` in the engine runtime.
The journal fixture contains two records with identifiers `add` and `change`,
bodies `fixture-add-record` and `fixture-change-record`, and synthetic machine
and boot IDs. Inspect it by decompressing `sample.journal.gz` into a temporary
file and running `journalctl --file=sample.journal -o export`.
The WASM processor loads a checked-in pass-through component compiled from WAT.
On Linux, `cargo xtask conformance` selects the Linux cases automatically.

The host-metrics fixture changes between two synthetic host roots. Overlapping
generations that claim the same root can conflict. The topic fixtures bind
two topics declared at startup with `impl_selection: force_mixed`. A change
to the broker's selected implementation requires a restart. The console
fixture uses pretty output. Structured stdout must be claimed at startup.
A successful rollout through a processor establishes the delivery behavior
named by its probe. Algorithm-specific edge cases belong in its owning crate.

Run `cargo xtask conformance --write-support` after editing the
[catalog][fixtures] or the [prerequisite records][prerequisites]. The [support
table][live] links each inherited gap to its required environment and
acceptance criterion. A failed operation stops the remaining stages of that
fixture. Other fixtures continue. Controller extensions keep their separate
startup checks.

For a provisioned fixture, pass one `--with NAME` per prerequisite.

[fixtures]: ../../crates/conformance/fixtures/fixtures.yaml
[prerequisites]: ../../crates/conformance/fixtures/prerequisites.yaml
[live]: live-support.md

### Embedded dashboard

Run the dashboard deployment alone, or omit `--check embedded-ui` to run the
full suite. Both paths use the same Rust probe and [YAML signal catalog][ui-signals].

```sh
cargo xtask conformance --check embedded-ui --html target/dashboard.html
```

The probe loads the dashboard at `/` and `/dashboard`, follows its local
asset references and module imports, and checks their content types. It compares
the API paths referenced by those modules with the endpoints it exercises.

The fixture needs two available engine cores. A marked syslog record crosses a
topic and reaches the file exporter. The probe checks the following observations.

| Dashboard surface | Evidence from the running engine |
| --- | --- |
| Channel charts and DAG | Both metrics routes expose scoped senders and receivers for data and control channels. Each pair joins on a channel ID. Sent and received counts increase during the workload. Idle gauges and error counters can be zero. |
| Charts and tooltips | Each applicable instrument has the expected unit, instrument kind and temporality, and a nonempty brief in the live schema and both metrics responses. The catalog identifies platform-specific instruments. |
| All-core selector | Two reader instances supply distinct core IDs. Removing the core, thread and NUMA dimensions leaves one series. Worker counts sum to two and the mean of the two 32-slot channel capacities is 32. |
| Status overlay | During creation, a pipeline with no observed runtime reports 0/0 cores and both conditions are unknown. A held receiver port then produces a runtime error, 0/1 running cores and a failed ready quorum. The healthy and shutdown event checks run on separate pipelines. |
| Live logs | A slow WebSocket subscriber falls behind while the fixture generates local audit events. The error supplies the last delivered sequence in `lag_before_seq`. The probe backfills from it, verifies contiguous pages against the retained HTTP logs and recovers the missed events. |

The topic topology uses an empty string when a process ID is absent, matching
the UI's single-process behavior. The log viewer renders arbitrary event names.
The probe checks the fields on real entries and observes any explicitly named
events in the stream or retained snapshot.

Terminal, HTML and JSON reports list each UI consumer and signal. The
[signal evaluator][evaluator] checks schema declarations, usable samples and
scoped traffic increases. The checker uses the same catalog to reject incomplete
observations from the deployment runner.

When a dashboard change adds, renames, or removes a telemetry dependency,
update `telemetry-signals.yaml` in the same change. Reviewers check that the
list matches what the UI reads. Under `sets`, each metric maps a catalog key
to its wire name and metadata. Use `dimensions` for
scope, `series` for required label values, and `traffic: true` when the marked
workload must increment the series. Use `channel` to identify endpoint roles and
`allCores` for the per-core and aggregate values. Under `events`, list status
kinds, conditions and expected pipeline states, the lag cursor, log fields,
and any explicitly named logs. Give a newly required event a
reproducible trigger in the deployment if the existing workload does not emit it.

[ui-signals]: telemetry-signals.yaml
[evaluator]: ../../crates/conformance/model/src/ui.rs
