+++
title = "So, What about the nmstate YAML API?"
date = 2026-11-13
description = "A dive into the YAML properties of nmstate"
[taxonomies]
tags = ["linux", "API", "yaml", "nmstate", "networking"]
+++

In this post I want to go into properties of the nmstate YAML schema
and some thoughts about how we could do it in Netfyr.
**Warning**: This might just be a bunch of BS. Be patient in reading it, and you *may*
find something interesting. Not a promise.

## Quick Comment on Netfyr

Netfyr is a declarative network configuration utility for Linux that aims
to make linux networking easier, more intuitive and explainable. Spiritually,
it is the combination of NetworkManager and nmstate written in Rust.  It allows a YAML configuration frontend and itself applies the configuration at the kernel layer.

See more about it netfyr.io.
See my DevConf talk about what and why we are doing it [here](https://youtu.be/-bWW5NLkUSs?si=H02uLQXe0NYDaJaF)

## NMstate

NMstate [1] is a declarative YAML API for configuring host networking.

```yaml
---
dns-resolver:
  config:
    server:
      - 192.0.2.1
    search:
      - example.org
routes:
  config:
    - destination: 0.0.0.0/0
      next-hop-interface: eth1
      next-hop-address: 192.0.2.1
interfaces:
  - name: eth1
    type: ethernet
    description: Main-NIC
    state: up
    ipv4:
      enabled: true
      dhcp: false
      address:
        - ip: 192.0.2.9
          prefix-length: 24
    ipv6:
      enabled: false
```

Above is an example of a YAML that configures an ethernet interface
with a static IPv4 address, IPv6 disabled, a default route, and
a DNS server and search domains.

Nmstate at its core is a Rust library [2], which exposes C bindings through Rust FFI. This allows
us to create bindings into Python and Go, which are used in our CI and in kubernetes
respectively.

There is also an accompanied CLI called nmstatectl.
```
# Show the current network state
nmstatectl show

# Apply a network state
nmstatectl apply foo.yml
```

### The Schema

Tools like [Kubernetes](https://github.com/kubernetes/kube-openapi) [3] and [cloud-init](https://cloudinit.readthedocs.io/en/latest/reference/schema.html) [4] generate types from API specs. However,
nmstate does no such thing. NMstate does not export a schema artifact like [JSON Schema](https://json-schema.org/) [5]
for example. The "schema" of NMstate is defined by the Rust types within the source code
that can be deserialized from and serialized to YAML with [serde.rs](https://serde.rs/) [6]. This is the standard
way to do YAML / JSON representation in Rust, and it is made super convenient with default
implmentation macros provided for the serde `Serialize` and `Deserialize` traits.

#### Serde rs

TLDR; Serde saves us work in custom serialization / deserialization logic.

Using `serde_yaml`, we can deserialize and serelize rust types into YAML / JSON.
Meaning, the Rust types themselves serve as the source of truth for the JSON
scema of the network state. So pretty much any data structure that implements
serde's `Serialize` / `Deserialize` trait is public and exposed to external consumers.

For example, take an basic interface configuration:
```
interfaces:
- name: eth1
  type: ethernet
  state: up
# Ignore the invalid mac address
  mac-address: LO:LZ:HA:HA:LO:LZ
```

It will can be represented as:

```rust
/// Top level NetworkState interfaces list
struct NetworkState {
  interfaces: Vec<Interface>
}

/// Data type for each interface
struct Interface {
  name: String
  type: String
  state: String
  #[serde(rename = "mac-address")]
  mac_address: String
}
```

However, it is more complicated than this in reality.

This is the real Interfaces struct:

```rust
/// Represent a list of [Interface].
#[derive(Clone, Debug, Default, PartialEq, Eq)]
#[non_exhaustive]
pub struct Interfaces {
    pub(crate) kernel_ifaces: HashMap<String, Interface>,
    pub(crate) user_ifaces: HashMap<(String, InterfaceType), Interface>,
    // The insert_order is allowing user to provided ordered interface
    // to support 5+ nested dependency.
    pub(crate) insert_order: Vec<(String, InterfaceType)>,
}
```

Which has custom serialization and deserialization.

It is wrapped in this big NetworkState struct:
```rust
pub struct NetworkState {
    // ------- Redacted -------
    /// Route rule, deserialize and serialize from/to `route-rules`.
    #[serde(
        rename = "route-rules",
        default,
        skip_serializing_if = "RouteRules::is_empty"
    )]
    pub rules: RouteRules,
    /// Route
    #[serde(default, skip_serializing_if = "Routes::is_empty")]
    pub routes: Routes,
    /// Network interfaces
    #[serde(default)]
    pub interfaces: Interfaces,
    // ------- Redacted -------
}
```

### Underlying Data Structures: public and private

We hide certain information from external consumers with Rust's visibility tools:
`pub` for public fields and `pub(crate)` for internal fields.
`#[serde(skip)]` is used to skip deserialization for certain fields in `NetworkState`.

Anything that is exposed publicly is subject to Semantic Versioning consideration. Don't
change it on a patch release. Exceptions need to be carefully considered.

Backwards compatibility is maintained with most fields being optional, with new fields
not an exception. We make fields optional with `#[serde(skip_serializing_if = "Option::is_none")]`

### Evolution and Versioning

So, the schema evolves naturally with additional optional fields as new features are added.
Unlike some projects, we do not support multiple versions of the schema at once. Th

### Quirks

Some de-serialization we do with custom functions. For example, we want to support all
these boolean strings: "1|0|true|false|yes|no|on|off|y|n". We do it with
`option_bool_or_string` in `rust/src/lib/deserializer.rs`.

Another one was mentioned earlier (struct Interfaces).

#### But it wasn't always this way

When NMstate was in it's Python implementation days, it started out with a 
JSON schema and validated with the `jsonschema` python library [7]. It later moved 
to doing custom validation logic instead of relying on the python `jsonschema` library [8].

Looking through the history, there was a manual edit needed in the schema every
time you needed to support a new field in the network state, meaning it was another
document nmstate needed to maintain along with the source code.

This was later dropped entirely when nmstate was ported from Python to Rust [9].
Gris, a core maintainer of nmstate who was there at the start of the project,
said that JSON schema was dropped because there was not much of a need for it.
Morever, there did not exist a light weight crate that achieved this at the time.

The way consumers of nmstate will have to validate their YAMLs now is to call
the `nmstate_net_state_format` function. Although it works, it can be argued
that an export API spec in the form of specification can help clients validate YAMLs
earlier and in a cleaner fashion.

#### Netfyr and Schema

From NetworkManager, we know that there will be many different clients that
attempt to configure networking: desktop settings applets, and CLIs.
If we want to use YAML as the language to configure
networking, then we can't have the clients rely on the daemon to validate
YAMLs.

For this reason, exporting a schema object can be of use to all the clients
that want to communicate with the Netfyr daemon. To reduce toil in maintaining
schemas, we must programatically generate them from Rust types.

I don't think this will allow easy use everywhere. For example, Linux desktop
uses DBus heavily for IPC, and is what NetworkManager user. DBus does have a typed
API description format in xml. Potentially will have to maintain a DBus to YAML
translation layer.

Kubernetes CRD does not support full JSON schema, it is actually a subset of OpenAPI,
which is a subset of JSON Schema. This will require some work to translate into. Another
direction we can take is developing our schema in a CRD compatible way, however that
might put more work on us unnecessarily since netfyr is not a Kubernetes first project.


# References

[1] - [nmstate.io](https://nmstate.io/)

[2] - [nmstate Rust core](https://github.com/nmstate/nmstate/tree/main/rust/src/lib) — the Rust library at the heart of nmstate

[3] - [kube-openapi](https://github.com/kubernetes/kube-openapi) — Kubernetes' OpenAPI spec generator, which derives the API schemas from Go types

[4] - [cloud-init JSON Schema](https://cloudinit.readthedocs.io/en/latest/reference/schema.html) — generated from cloud-init's Python dataclass definitions, usable via `cloud-init schema --config-file`

[5] - [json-schema.org](https://json-schema.org/)

[6] - [serde.rs](https://serde.rs/)

[7] - [91f97f14](https://github.com/nmstate/nmstate/commit/91f97f14) — Introduce a basic schema with validation tests (2018)

[8] - [2599d3af](https://github.com/nmstate/nmstate/commit/2599d3af) — validator: drop jsonschema dependency (2021)

[9] - [e7da681f](https://github.com/nmstate/nmstate/commit/e7da681f) — Switch to all rust based (2022)
