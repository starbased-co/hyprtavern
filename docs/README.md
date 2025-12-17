# `hyprtavern`

## Dependencies

```bash
paru -S cmake gcc
paru -S hyprutils-git hyprlang-git hyprwire-git hyprtoolkit-git
```

### Install `hyprwire-protocols`

```bash
git clone -b hyprtavern https://github.com/hyprwm/hyprwire-protocols
cd hyprwire-protocols
cmake -B build
sudo cmake --install build # installs /usr/share/hyprwire-protocols/hyprtavern/
```

## Build & Run

```bash
git clone git@github.com:hyprwm/hyprtavern.git
cd hyprtavern
cmake -B build
cmake --build build
PATH="$PWD/build/barmaids/hyprtavern-kv:$PATH" ./build/hyprtavern
# build/
# ├── hyprtavern (daemon)
# ├── barmaids/ (helpers)
# │   └── hyprtavern-kv/
# │       └── hyprtavern-kv (store)
# └── tools/
#     └── hyprtavern-spy/
#         └── hyprtavern-spy (debugging)
```

## Architecture

- Applications register themselves on the bus as `objects`, each exposing a set of protocols it implements along with discoverable properties.
- Other applications query the bus to find objects matching specific protocols or properties, receiving object handles that reveal metadata and enable connection.
- When connecting, hyprtavern passes a file descriptor directly between the two applications, establishing a peer-to-peer hyprwire channel that bypasses the bus entirely for subsequent communication.
- Applications request permission groups that persist either for the session or permanently, with non-sandboxed apps optionally skipping checks.
<!-- yes is ai summary, generated from my voice notes -->

### Registration Flow

```mermaid
%%{init: {'sequence': {'mirrorActors': false, 'noteAlign': 'left'}}}%%
sequenceDiagram
    participant AppA as App A
    participant Tavern as hyprtavern

    AppA->>Tavern: get_bus_object
    Tavern-->>AppA: object_id: 42

    Note over AppA: #nbsp;#nbsp;#nbsp;#nbsp;#nbsp;Bus Object (id=42)<br/>- expose_protocol("my_protocol_v1")<br/>- expose_property("APP:TYPE=daemon")<br/>- require_permissions([21000])
```

### Discovery & Connection

```mermaid
%%{init: {'sequence': {'mirrorActors': false}}}%%
sequenceDiagram
    participant AppB as App B
    participant Tavern as hyprtavern
    participant AppA as App A

    AppB->>Tavern: get_query_object<br/>(protocol_names, props)
    Tavern-->>AppB: results: [42]

    AppB->>Tavern: get_object_handle(42)
    Tavern-->>AppB: name, protocols, props
    Tavern-->>AppB: done

    AppB->>Tavern: connect()
    Tavern->>AppA: new_fd<br/>(wire_fd, security_token)
    Tavern-->>AppB: socket(fd)

    Note over AppB,AppA: direct hyprwire channel
```

### Permission Request Flow

```mermaid
%%{init: {'sequence': {'mirrorActors': false}}}%%
sequenceDiagram
    participant AppB as App B
    participant Tavern as hyprtavern
    participant Policy as UI/Policy

    AppB->>Tavern: get_security_object
    Tavern-->>AppB: token

    AppB->>Tavern: set_identity(name, desc)

    AppB->>Tavern: obtain_permission<br/>(monitoring_basic, session)
    Tavern->>Policy: prompt/check_policy
    Policy-->>Tavern: granted/denied

    Tavern-->>AppB: permission_result<br/>(granted/denied)
```

### KV Store Operations

```mermaid
%%{init: {'sequence': {'mirrorActors': false}}}%%
sequenceDiagram
    participant App as App
    participant KV as hyprtavern-kv

    App->>KV: set_value(key, val, app_value)
    Note right of KV: ~/.local/share/hyprtavern/<br/>hyprtavern-kv.dat
    KV-->>App: value_set

    App->>KV: get_value(key, app_value)
    KV-->>App: value_obtained
```

### Protocols

Located in hyprwire-protocols (`hyprtavern` branch):

| Protocol                                     | Purpose             |
| -------------------------------------------- | ------------------- |
| `hp_hyprtavern_core_v1`                      | Core bus operations |
| `hp_hyprtavern_kv_store_v1`                  | Key-value storage   |
| `hp_hyprtavern_permission_authentication_v1` | Security/auth       |

## Common Issues

> "Couldn't load proto: File was not found"

You're missing hyprwire-protocols or using the wrong branch

> Missing type `HP_HYPRTAVERN_CORE_V1_FD`

Your `hyprwire` version is too old. Install `hyprwire-git`

> `CSetupLineEdit` has no member `password`

Your `hyprtoolkit` version is too old. Install `hyprtoolkit-git`
