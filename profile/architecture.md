# Architecture diagram

The thermal-label stack, in dependency order. Arrows point from consumer to
producer (i.e. "depends on").

```mermaid
flowchart TB
  subgraph App["Your application"]
    APP[Node app or browser app]
  end

  subgraph Cli["CLI"]
    CLI["thermal-label-cli<br/><sub>list · status · print</sub>"]
  end

  subgraph Drivers["Device families"]
    BQ["@thermal-label/brother-ql-*<br/><sub>core · node · web</sub>"]
    LM["@thermal-label/labelmanager-*<br/><sub>core · node · web</sub>"]
    LW["@thermal-label/labelwriter-*<br/><sub>core · node · web</sub>"]
  end

  subgraph Plumbing["Plumbing"]
    TR["@thermal-label/transport<br/><sub>USB · TCP · WebUSB · Web Bluetooth · Web Serial</sub>"]
    CT["@thermal-label/contracts<br/><sub>Transport · PrinterAdapter · PrinterDiscovery · MediaDescriptor · PrinterStatus</sub>"]
  end

  APP --> CLI
  APP --> BQ
  APP --> LM
  APP --> LW

  CLI --> BQ
  CLI --> LM
  CLI --> LW

  BQ --> TR
  LM --> TR
  LW --> TR

  BQ --> CT
  LM --> CT
  LW --> CT
  TR --> CT

  classDef app fill:#fef3c7,stroke:#b45309,color:#1c1917
  classDef cli fill:#fce7f3,stroke:#9d174d,color:#1c1917
  classDef driver fill:#e0f2fe,stroke:#075985,color:#1c1917
  classDef plumbing fill:#dcfce7,stroke:#166534,color:#1c1917

  class APP app
  class CLI cli
  class BQ,LM,LW driver
  class TR,CT plumbing
```

## How to read it

- **`@thermal-label/contracts`** at the bottom is types-only. Everything else
  imports from it; it imports nothing internal.
- **`@thermal-label/transport`** sits one level up and provides the byte
  channels. Drivers compose it; they don't reimplement USB/TCP/WebUSB.
- **Each driver** is published as `core` + `node` + `web` packages. `core` is
  pure protocol; `node` and `web` add the integration with a transport.
- **`thermal-label-cli`** auto-discovers any installed driver with a
  `discovery` named export. Adding a driver doesn't require a CLI change.
- **Your application** imports drivers directly (the typed API) or shells out
  to the CLI (for ops, CI, "is this cable working" moments).
