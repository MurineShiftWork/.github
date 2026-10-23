# MurineShiftWork

**Tools for standardised, reproducible behavioural neuroscience acquisition.**

A rig is a pile of heterogeneous instruments: behaviour hardware, cameras, ephys,
stimulators, scales, stages. MurineShiftWork is an extendable framework that drives
them through one acquisition controller and writes the same self-describing,
analysis-ready output every time. Modalities align by design, not by post-hoc
bookkeeping, and every instrument is a pluggable driver: Bpod is one option, not a
requirement.

## The acquisition cycle

Every session flows through the same loop, and each package owns one stage of it:

```
              subject + rig + task config  (layered YAML: defaults > rig > task > subject > CLI)
                              │
                              ▼
        ┌─────────────────────────────────────────────┐
        │             ACQUISITION CONTROL              │   murineshiftwork
        │   task state machine · hooks · config merge  │   (extendable framework)
        └─────────────────────────────────────────────┘
             │                  │                   │
     names every run     drives the task      standardises output
             ▼                  ▼                   ▼
   ┌───────────────┐   ┌──────────────────┐   ┌────────────────────┐
   │   NAMESPACE   │   │     HARDWARE      │   │   STANDARD OUTPUT  │
   │ acquisition-  │   │ parallel, typed   │   │ msw-io: session +  │
   │ namespace     │   │ acquisitions:     │   │ acquisition        │
   │ subject >     │   │ behaviour ·       │   │ manifests, per-    │
   │ session >     │   │ cameras · ephys · │   │ trial JSONL,       │
   │ acquisition > │   │ stim · stage ·    │   │ reader API         │
   │ file          │   │ scale             │   │                    │
   └───────────────┘   └──────────────────┘   └────────────────────┘
             │                  │                   │
             └──────── TTL barcodes stamp a shared clock ────────┘
                          (ttl-barcoder: align every stream offline)
                              │
                              ▼
                 analysis-ready, self-describing dataset
```

Config resolves, the namespace mints the paths, the controller runs the task and
fans out to whatever hardware is attached, each modality writes a typed acquisition,
TTL barcodes stamp a shared clock so the streams align offline, and `msw-io` writes
standardised manifests and trial data that the reader API loads straight back for
analysis.

For analysis you only need the readers, which install standalone and lean:

```bash
pip install msw-io   # readers + namespace, no acquisition stack
```

To run a rig, install the framework
(**[murineshiftwork](https://github.com/MurineShiftWork/murineshiftwork)**) with the
full stack, then add whatever hardware the rig has:

```bash
pip install "murineshiftwork[full]"        # framework + tasks + plugins
pip install "murineshiftwork[full,rce]"    # + Raspberry Pi cameras
msw run --setup rig-a --subject mouse001 --task sequence
```

Hardware and plugins are a la carte extras:

- `[oe]` Open Ephys / ephys
- `[rce]` Raspberry Pi cameras
- `[pulsepal]` PulsePal stim
- `[calibration]` weighing scales

## The acquisition namespace

The heart of MSW is
**[acquisition-namespace](https://github.com/MurineShiftWork/acquisition-namespace)**
([PyPI](https://pypi.org/project/acquisition-namespace/)): an abstract, versioned
grammar for *where data lives and what it is*, independent of the instrument that
produced it.

```
subject > session > acquisition > file
{subject}__{datetime}__{acq_type}[__{task}][__v{n}]

# behaviour acquisition (v4.x)
t004_m2045__20260514_143022_123456__msw__sequence__v1
#  subject       datetime          type   task   ver

# a synchronised video sibling of the same session
t004_m2045__20260514_143022_123456__video_flir
```

Every modality drops its data into the same `subject > session > acquisition`
hierarchy under a typed name (`msw`, `video_flir`, `video_rce`, `pxi`, ...), stamped
with the namespace version and paired with a YAML manifest. A dataset is therefore
**self-describing**: you can identify it, load it, and align it from the structure
alone, with no reliance on a given rig's tribal knowledge. That is what makes MSW
output portable, re-readable, and reprocessable years later.

The package is standalone and framework-agnostic (no MSW dependency), so any project
that needs structured, validated directory layouts can use it:

- Pydantic models for the full spec (`NamespaceSpec`, `NamespaceLevelSpec`,
  `NamespaceValidatorSpec`)
- Named validators with `${name}` interpolation: define a pattern once, reference it
  across levels
- YAML-backed schema with round-trip serialisation (`from_yaml` / `write_yaml`)
- `NamespaceBuilder` API: `build_path`, `generate_path`, `validate_path`,
  `extract_level_values`, `validate_field`

```yaml
# MSW namespace spec (v4.x)
validators:
  subject_id: { pattern: "[A-Za-z]+[0-9]+" }   # t004, seq001
  animal_id:  { pattern: "[A-Za-z][0-9]+" }    # m2045, r001
levels:
  acquisition:
    template: "{subject}__{datetime}__{acq_type}__{task}__v{version}"
    optional_fields: [task, version]
```

## Hardware

MSW abstracts each instrument behind a small, single-purpose package, so a rig mixes
and matches backends without changing the task logic. Behaviour control (Bpod state
machine, reward and valve logic) is built into the core framework.

**MSW plugins** extend the acquisition framework:

| Package | What it adds |
|---|---|
| msw-open-ephys | Open Ephys / Neuropixels electrophysiology with probe metadata capture |
| [msw-flir-bonsai](https://github.com/MurineShiftWork/msw-flir-bonsai) | FLIR camera capture via a Bonsai workflow runner |
| rpi-camera-ensemble | Multi-RPi camera conductor: trigger, record, TTL sync |

**Standalone hardware tools** are framework-agnostic drivers, usable on their own with no MSW dependency:

| Package | Device |
|---|---|
| [ttl-barcoder](https://github.com/MurineShiftWork/ttl-barcoder) | TTL barcode encode/decode for cross-stream alignment |
| [pypulsepal](https://github.com/MurineShiftWork/pypulsepal) | PulsePal 4-channel pulse generator (USB serial) |
| [one-axis-stage](https://github.com/MurineShiftWork/one-axis-stage) | Serial linear stage: jog, positioning, interactive CLI |
| [serial-scale-hx711](https://github.com/MurineShiftWork/serial-scale-hx711) | HX711 load-cell weighing scale (serial) |
| [serial-scale-bench](https://github.com/MurineShiftWork/serial-scale-bench) | RS-232 bench-scale adapter |

---

Packages and issues on GitHub under [MurineShiftWork](https://github.com/MurineShiftWork).
Tested on Unix and Windows, cross-platform.
