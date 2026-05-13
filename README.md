# Universal Message Manager (UMM)

> *UMM allows any computer or device to receive intelligent information from any other computer or device.*
>
> — Phil Braham, Realtime Software Pty Ltd

UMM is a **typed publish/subscribe middleware** for connecting devices, processes and systems without any of them knowing about each other. Originally designed for manufacturing plant control (robots, conveyors, sensors, databases), it is general enough for any distributed system.

---

## The core idea

For example: A temperature sensor publishes a reading. An alarm console subscribes to readings above 80°C. A database logger subscribes to everything. **None of them know about each other.** The router matches and delivers.

```
Temperature Sensor ──► Router ──► Alarm Console   (only if temp > 80°C)
                          │
                          └────► Database Logger   (all readings)
```

The magic is in the **TLV (Type/Length/Match-word/Value)** message format. Every piece of data carries its type, and subscriptions carry matching criteria. The router's match engine calls the right handler for each type — so matching "temperature above 80°C" uses the same infrastructure as matching "audio contains the word Australia".

---

## Requirements

- Python 3.8 or later (3.12 recommended)
- No external dependencies for core operation

---

## Quick start

```bash
# Install and start everything
./runumm start

# Open the GUI
# http://localhost:8765

# Start the sorting machine demo
./runumm start demo

# Check status
./runumm status OR
./runumm agents

# Stop everything
./runumm stop
```

---

## System management — `umm_ctl.py`

`umm_ctl.py` manages the UMM process stack. It reads `umm.conf` for configuration.

```bash
python3 umm_ctl.py [--config FILE] [--debug] COMMAND [target]
```

**Commands:**

| Command | Description |
|---------|-------------|
| `start [target]` | Kill stale processes, roll logs, start core (or named group/agent) |
| `stop [target]` | Stop agents |
| `restart [target]` | Stop then start |
| `status [target]` | Show running status, router/GUI ports, bridge connections |
| `show` | Show parsed configuration including bridge env var status |
| `validate` | Check configuration for errors |
| `logs [list\|reset]` | Tail current log, list all logs, or start fresh |
| `version` | Show version of running agents |
| `killall` | Kill all UMM-related processes |
| `cleanstart [target]` | Alias for `start` (kept for compatibility) |

**Options:**

| Option | Description |
|--------|-------------|
| `--config FILE` | Config file path (default: `umm.conf` next to the script) |
| `--debug` | Pass `--debug` to all started agent processes |

**Targets** are group names (`core`, `demo`, `bridge`), agent names (`bridge-ubuntu`), or `all`.

**Examples:**
```bash
./runumm start                     # start core stack
./runumm start demo                # start sorting demo
./runumm start bridge-ubuntu       # start one bridge
./runummy start bridge              # start all bridges
./runumm stop demo
./runumm --config umm-ubuntu.conf start    # use alternate config
./runumm start parser -- --show-control    # pass args to agent
./runumm start send               # start the interactive send agent
```

### Configuration — `umm.conf`

```ini
[settings]
router    = localhost      # client connect address (0.0.0.0 to accept external connections)
port      = 5300
# router_id auto-detected from primary IP (last two octets: 192.168.1.31 → 1.31)
# router_id = 1.1          # override if needed
gui_port  = 8765
log_dir   = logs
python    = python3

[agent.my-agent]
module    = my.module           # Python module (run with -m)
group     = mygroup             # logical group name
after     = router              # start after this agent (or 'manual', '-')
name      = MyAgent             # display name (passed as --name)
type      = 1010                # agent mtype (passed as --type)
restart   = always              # always | never | on-failure
restart_delay = 5               # seconds before restart
options   = --my-flag value     # extra CLI arguments
foreground = false              # run in terminal (dev tools)
```

### Environment variables

| Variable | Description |
|----------|-------------|
| `UMM_ROUTER` | Override router address: `host:port` or `host` |
| `UMM_ROUTER_ID` | Override router ID: `site.node` |
| `UMM_BRIDGE_<NAME>` | Configure a bridge (see Bridge section) |

### Log management

On each full `start`, existing logs are moved to `logs/backup/` with a timestamp. The `logs/current` symlink always points to the active log file.

### Running on multiple machines

Use a separate config file per machine to avoid overwriting machine-specific settings when updating:

```bash
# On the remote machine
python3 umm_ctl.py --config umm-ubuntu.conf start
```

The `umm-ubuntu.conf` file stays on that machine permanently and is not included in project updates.

---

## Modules

All modules are plain UMM clients — they connect to a router just like any other agent. All accept `--router`, `--port`, `--name`, `--debug`, and `--version` unless noted otherwise.

---

### Router — `umm.router`

The central message broker. All agents connect to it; it matches and delivers messages.

```bash
python3 -m umm.router [options]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--port N` | 5300 | TCP port to listen on |
| `--host ADDR` | (all interfaces) | Bind address. Omit or use `0.0.0.0` to accept external connections. `localhost` restricts to local only. |
| `--id SITE.NODE` | auto from IP | Router ID in `site.node` format. Auto-detected from primary IP if not set (e.g. `192.168.1.31` → `1.31`) |
| `--name NAME` | UMMRouter | Router process name |
| `--typedir DIR` | (auto) | Extra type plugin directory. Can be repeated. |
| `--plugin-port N` | (auto) | Port for match plugins (default: port+1) |
| `--debug` | off | Verbose logging |

The router publishes lifecycle events on `mtype=1`:
- `(1, AgentAttached)` — agent joined the bus
- `(1, AgentDetached)` — agent left (subscriptions removed, Unregister published for each)
- `(1, AgentReattached)` — reconnecting agent resumed
- `(1, Register)` — new subscription stored
- `(1, Unregister)` — subscription removed

---

### Supervisor — `umm.modules.supervisor`

Process manager. Starts, monitors and restarts agents defined in `supervisor.conf`. Started automatically by `umm_ctl.py`.

```bash
python3 -m umm.modules.supervisor [options]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--config FILE` | supervisor.conf | Agent definition file |
| `--max-restarts N` | 10 | Maximum restart attempts before giving up |
| `--name NAME` | Supervisor | Process name |

**mtype 12** — responds to GetCapability, GetVersion, SetDebug, Shutdown.

---

### Timer — `umm.modules.timer`

Scheduled message delivery. Fire a message once, N times, or forever at a given interval.

```bash
python3 -m umm.modules.timer [options]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--type N` | 2 | mtype to register as. Use a custom value to run multiple independent timer instances (e.g. `--type 1050`) |
| `--name NAME` | Timer | Process name |

**mtype 2** (or custom) — standard subtypes:

| Subtype | Direction | Description |
|---------|-----------|-------------|
| 1 | in | Timer request: `UTime(interval) Int(count) Msg(mtype,subtype) [MsgRef(corr)]` |
| 10 | out | Confirmed: `Int(ref) MsgRef(corr)` — ref is used to cancel |
| 11 | out | Error: `Char(message) MsgRef(corr)` |
| 2 | in | Cancel: `Int(ref)` |
| -2 | out | Fired: `Int(ref) Int(fire_count)` — published on each fire |
| 3 | in | GetVersion |
| 4 | in | Shutdown |

**Example — fire mtype=100 subtype=1 every 60 seconds forever:**
```python
from umm.core.tlv import TLVBuffer, TLV, TLV_MSG
from umm.types.builtin import UTimeType, IntType
import struct

buf = TLVBuffer()
buf.add(UTimeType().make_tlv(60.0))               # interval
buf.add(IntType().make_tlv(-1))                    # count: -1 = forever
buf.add(TLV(TLV_MSG, struct.pack('!ii', 100, 1))) # message to fire
agent.send(mtype=2, subtype=1, tlvs=list(buf))
```

**Running multiple timer instances:**
```bash
python3 -m umm.modules.timer --type 1050 --name ConveyorTimer
python3 -m umm.modules.timer --type 1051 --name BarcodeTimer
```

---

### Type Directory — `umm.modules.typedir`

Registry of all known TLV types. Agents can query for type definitions, allocate new type IDs, and browse the type hierarchy.

```bash
python3 -m umm.modules.typedir [options]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--min-id N` | 100 | Minimum ID for user-allocated types |
| `--typedir DIR` | (auto) | Extra type plugin directory. Can be repeated. |
| `--name NAME` | TypeDirectory | Process name |

**mtype 10** — responds to GetCapability, GetVersion, SetDebug, Shutdown, plus type query subtypes.

---

### Finite State Machine — `umm.modules.fsm`

Multi-dimensional FSM. Supports a three-dimensional state vector `(primary, control, aux)`. Transitions are registered as UMM subscriptions — the router handles message delivery, the FSM handles state logic.

```bash
python3 -m umm.modules.fsm --config plant.fsm [options]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--config FILE` | (required) | FSM definition file (`.fsm`) |
| `--name NAME` | FSM | Process name |

**mtype 5** — responds to GetCapability, GetVersion, SetDebug, Shutdown.

---

### Parser — `umm.modules.parser`

Bus monitor and debugger. Subscribes to messages and displays them in a human-readable format. Primarily a development tool; run in foreground.

```bash
python3 -m umm.modules.parser [options]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--mtype N` | all | Filter by mtype. Can be repeated for multiple mtypes. |
| `--subtype N` | all | Filter by subtype |
| `--channel N` | all | Filter by channel. Can be repeated. |
| `--show-channel` | off | Show channel number in output |
| `--show-control` | off | Include UMM control messages (Register, Unregister, lifecycle) |
| `--heading` | off | Show column headings |
| `--hex` | off | Show raw hex dump of TLV payloads |
| `--flags` | off | Show message flags |
| `--raw` | off | Raw mode — minimal formatting |
| `--interactive` | off | Enable runtime commands (see below) |
| `--names FILE` | (auto) | Custom names config file |
| `--log FILE` | none | Log output to file |
| `--replay FILE` | none | Replay a log file |
| `--replay-delay N` | 0 | Delay between replayed messages (seconds) |
| `--typedir DIR` | (auto) | Extra type plugin directory |

**Interactive commands** (when `--interactive`):
- `subs` — list all current subscriptions
- `subs <mtype>` — filter by mtype
- `who <mtype> <subtype>` — show who is subscribed
- `q` — quit

**Examples:**
```bash
# Monitor everything including control messages
python3 -m umm.modules.parser --heading --show-control

# Watch only timer traffic
python3 -m umm.modules.parser --mtype 2 --heading

# Watch bridge channel
python3 -m umm.modules.parser --channel 6 --heading
```

---

### Bridge — `umm.modules.bridge`

Inter-router bridge. Connects two UMM routers so agents on either side can subscribe to messages from the other side. The bridge appears as a standard UMM agent on each router — no router changes required.

```bash
python3 -m umm.modules.bridge [options]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--router-a HOST:PORT` | localhost:5300 | Router A (local/home router) |
| `--router-b HOST:PORT` | (required) | Router B (remote router) |
| `--mtype-a N` | 11 | mtype on Router A |
| `--mtype-b N` | 111 | mtype on Router B |
| `--name NAME` | UMMBridge | Process name |
| `--debug` | off | Verbose logging |

**mtype 11** on Router A, **mtype 111** on Router B (defaults). Override if these conflict with existing agents.

**Configuration via environment variable:**
```bash
# Minimal — address only, mtypes default to 11/111
export UMM_BRIDGE_UBUNTU=192.168.64.3:5300

# With custom remote mtype
export UMM_BRIDGE_UBUNTU=192.168.64.3:5300,mtype_b=112

# With both mtypes specified
export UMM_BRIDGE_UBUNTU=192.168.64.3:5300,mtype_a=12,mtype_b=112
```

If `UMM_BRIDGE_<NAME>` is set, a bridge agent is auto-created even without a `[agent.bridge-<name>]` section in `umm.conf`. The config section is only needed to customise restart policy.

**Starting bridges:**
```bash
python3 umm_ctl.py start bridge             # all bridge agents
python3 umm_ctl.py start bridge-ubuntu      # one bridge by name
```

**Receiving remote messages — method 1: ForwardRequest**

Send `(11, 101)` to the bridge with `Int(mtype) Int(subtype)` payload. The bridge mirrors the subscription on the far router and forwards matching messages back on `channel=BRIDGE(6)`. The bridge replies with `(11, 102) ForwardAck` containing the far router's ID.

Then subscribe locally to `(mtype, subtype, channel=6)` to receive forwarded messages.

```python
# Ask bridge to forward all timer-fired messages from remote router
buf = TLVBuffer()
buf.add(IntType().make_tlv(2))    # mtype = Timer
buf.add(IntType().make_tlv(-2))   # subtype = Fired
agent.send(mtype=11, subtype=101, tlvs=list(buf))

# Subscribe to receive them on the bridge channel
agent.register(mtype=2, subtype=-2, callback=on_remote_timer,
               channel=Channel.BRIDGE)
```

Use `Int(0) Int(0)` to forward all messages.

**Receiving remote messages — method 2: RouterId TLV**

Include a RouterId TLV (type_id=17) in subscription match criteria. The bridge detects it, strips it, and mirrors the subscription on the remote router. Use `(0, 0)` as a wildcard for any remote router.

```python
from umm.types.builtin import RouterIdType

agent.register(
    mtype      = 100,
    subtype    = 1,
    callback   = on_remote_msg,
    match_tlvs = [RouterIdType.make_tlv(site=1, node=2)],
)
```

**Bridge subtypes (mtype=11 on A, mtype=111 on B):**

| Subtype | Name | Description |
|---------|------|-------------|
| 1 | GetCapability | Query bridge capability |
| 2 | GetVersion | Query bridge version |
| 3 | SetDebug | Enable/disable debug output |
| 4 | Shutdown | Shutdown bridge (A-side only) |
| 101 | ForwardRequest | Request forwarding of `(mtype, subtype)` from far router |
| 102 | ForwardAck | Response: `Int(far_router_id)` |
| 103 | Status | Request bridge status |
| 104 | StatusReply | JSON status: addresses, sub counts, seen router IDs |
| 106 | RouterSeen | `Int(site) Int(node) Char(side)` — new router observed |
| 107 | RouterGone | `Int(site) Int(node)` — router timed out |

---

### GUI Server — `umm_gui/server.py`

WebSocket bridge between the browser GUI and the UMM bus. The GUI itself is a static HTML file (`gui.html`) served by this process.

```bash
python3 umm_gui/server.py [options]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--router HOST` | localhost | UMM router address |
| `--port N` | 5300 | UMM router port |
| `--gui-port N` | 8765 | HTTP/WebSocket port to serve on |
| `--debug` | off | Verbose logging |

**Open the GUI:** `http://localhost:8765`

**Running on a remote machine:**
```bash
# Monitor a router on another machine from any browser
python3 umm_gui/server.py --router 192.168.1.31 --port 5300 --gui-port 8765
```

The GUI can run on any machine that can reach the router — it doesn't need to be on the same host.

**GUI tabs:**

| Tab | Description |
|-----|-------------|
| Subscribe | Register subscriptions with mtype, subtype, channel, and optional TLV match criteria |
| Log | Live message stream with channel filter buttons (ch1 Confirm, ch2 Debug, ch3 Meta, ch4 Heartbeat, ch6 Bridge) |
| Send | Send arbitrary messages with TLV payloads |
| Timer | Schedule timer requests (supports custom timer mtype) |
| Agents | List all attached agents; click capability button to inspect any agent |
| Capability | Query and display full capability descriptor for any mtype |
| Types | Browse the type registry |
| Debug | Raw debug output |

**Channel subscription buttons** (in Subscribe tab):
- `ch1 Confirm` — delivery confirmations and error responses
- `ch2 Debug` — runtime debug output from agents
- `ch3 Meta` — capability and version responses
- `ch4 Heartbeat` — supervisor liveness ping/pong
- `ch6 Bridge` — messages forwarded from remote router via bridge

---

## Writing agents

### Publisher

```python
from umm.agent import UMMAgent
from umm.types.manufacturing import TemperatureType

sensor = UMMAgent(host='localhost', process_name='TempSensor', agent_mtype=100)
sensor.connect()

temp = TemperatureType()
while True:
    reading = read_hardware_sensor()
    sensor.send(mtype=100, subtype=1,
                tlvs=[temp.make_tlv(reading)])
    time.sleep(1.0)
```

### Subscriber

```python
from umm.agent import UMMAgent
from umm.types.manufacturing import TemperatureType
from umm.types._base import MatchOp

console = UMMAgent(host='localhost', process_name='AlarmConsole')
console.connect()

temp = TemperatureType()

def on_high_temp(msg):
    from umm.core.tlv import decode_all
    for tlv in decode_all(msg.tdata.value):
        if tlv.type_id == temp.TYPE_ID:
            print(f"ALARM: {temp.to_text(temp.unpack(tlv.value))}")

# Subscribe to temperature readings above 80°C
console.register(
    mtype      = 100,
    subtype    = 1,
    callback   = on_high_temp,
    match_tlvs = [temp.make_subscription_tlv(80.0, MatchOp.GT)],
)
console.run()
```

### Adding capability to an agent

```python
from umm.capability import CapabilityMixin, make_capability, SubtypeSpec

class MyAgent(CapabilityMixin):
    CAPABILITY = make_capability(
        what        = 'MyAgent',
        mtype       = 1010,
        description = 'Does useful things.',
        subtypes    = [
            SubtypeSpec(subtype=1, name='Command', direction='in',
                        description='Send a command.', response_subtype=2),
            SubtypeSpec(subtype=2, name='Ack', direction='out',
                        description='Command acknowledged.', responds_to=1),
        ],
        author = 'Realtime Software Pty Ltd',
    )

    def run(self):
        agent = UMMAgent(host='localhost', agent_mtype=1010)
        agent.connect()
        self.register_capability_handler(agent)
        self.register_version_handler(agent, mtype=1010)
        self.register_debug_handler(agent, mtype=1010)
        self.register_shutdown_handler(agent, mtype=1010)
        ...
```

---

## TLV wire format

Every piece of data in UMM is encoded as a TLV:

```
┌──────────┬──────────┬──────────┬──────────────────┬──────────┐
│  Type    │  Match   │  Length  │  Value           │ Padding  │
│  4 bytes │  4 bytes │  4 bytes │  Length bytes    │  to 4B   │
│  NBO     │  NBO     │  NBO     │                  │          │
└──────────┴──────────┴──────────┴──────────────────┴──────────┘
```

The **Match word** is owned by the type handler:
- Numeric types (Int, Float…): comparison operator index (EQ/NE/GT/LT/GE/LE)
- String (Char): operator (EQ, NE, CI_EQ, wildcard, contains)
- RouterId (type 17): bridge routing hint — site(2 bytes) + node(2 bytes)

TLVs can be nested: the value of one TLV can contain other TLVs.

---

## Built-in types

| ID | Name | Match operators |
|----|------|----------------|
| 1 | Raw | =, != |
| 2 | Int | =, !=, >, <, >=, <= |
| 3 | UInt | =, !=, >, <, >=, <= |
| 4 | Short | =, !=, >, <, >=, <= |
| 5 | UShort | =, !=, >, <, >=, <= |
| 6 | Float | =, !=, >, <, >=, <= |
| 7 | Bool | =, != |
| 8 | Char | =, !=, CI=, CI!=, \*= (wildcard), contains |
| 9 | UTime | =, !=, before, after |
| 10 | MDNum | metadata field match |
| 11 | Msg | nested message |
| 12 | AttId | =, != |
| 13 | CoOrd | =, within radius |
| 14 | FName | =, wildcard |
| 15 | LogId | = (on process name) |
| 16 | MsgRef | =, != |
| 17 | RouterId | bridge routing hint: site(2) + node(2) NBO |

### Manufacturing types (IDs 50–99)

| ID | Name | Unit | Notes |
|----|------|------|-------|
| 50 | Temperature | °C | Float-based, >/< alarms |
| 51 | Pressure | PSI | M-word = unit (PSI or Bar) |
| 52 | ConveyorSpeed | m/s | M-word = direction |
| 53 | RobotState | — | Enum: IDLE/RUNNING/FAULT/… |
| 54 | ProcessEvent | — | String + M-word severity |
| 55 | AlarmLevel | — | 0=INFO … 4=FAULT |

---

## mtype namespace

| Range | Owner | Notes |
|-------|-------|-------|
| 1 | UMM system | Router control and system services |
| 2 | Timer | Standard timer service |
| 3 | Command | Generic command/response |
| 4 | Metadata | File/record metadata |
| 5 | FSM | Finite state machine |
| 6–9 | FileSend, FileRecv, Database, MicroBill | Standard services |
| 10 | TypeDir | Type directory service |
| 11 | Bridge | Inter-router bridge (Router A side) |
| 12 | Supervisor | Process management and watchdog |
| 13 | GUI | GUI console server |
| 100–999 | Standard application range | Sensors, actuators, alarms, process control |
| 111 | Bridge (B-side default) | Bridge on remote router |
| 1000+ | Site-specific | Sorting demo uses 1001–1006, 1050–1051 |

---

## Channels

Messages carry a channel number that allows subscribers to filter by purpose:

| Channel | Name | Description |
|---------|------|-------------|
| 0 | App | Normal application traffic (default) |
| 1 | Confirm | Delivery confirmations and error responses |
| 2 | Debug | Runtime debug output from agents |
| 3 | Meta | Capability and version responses |
| 4 | Heartbeat | Supervisor liveness ping/pong |
| 5 | Supervisor | Supervisor control messages |
| 6 | Bridge | Messages forwarded from remote router via bridge |

---

## UMM system layer (mtype=1)

### Agent discovery

```python
# Ask the router who is on the bus
agent.send(mtype=1, subtype=110)
agent.register(mtype=1, subtype=111, callback=on_agents)
# Response: Char(json) with router_id, router_name, and agents list
```

### Capability queries

```python
# Query a specific agent (mtype=2 = Timer)
agent.send(mtype=1, subtype=10, tlvs=[IntType().make_tlv(2)])

# Broadcast — all capability-aware agents respond
agent.send(mtype=1, subtype=10)

# Listen for responses (on channel=META)
agent.register(mtype=1, subtype=11, callback=on_cap, channel=3)
```

---

## Adding a new type

Drop a `.py` file in any type directory and the router loads it automatically:

```python
# myplant/types/hydraulic_pressure.py
from umm.types._base import BaseUMMType, MatchOp
import struct

class HydraulicPressureType(BaseUMMType):
    TYPE_ID     = 60
    TYPE_NAME   = "HydraulicPressure"
    BASE_TYPE   = "Float"
    DESCRIPTION = "Hydraulic line pressure"
    UNIT        = "Bar"
    MATCH_OPS   = {
        MatchOp.EQ: lambda a, b: abs(a-b) < 0.01,
        MatchOp.GT: lambda a, b: a > b,
        MatchOp.LT: lambda a, b: a < b,
    }
    def pack(self, value):   return struct.pack('!d', float(value))
    def unpack(self, data):  return struct.unpack('!d', data)[0]
    def to_text(self, v):    return f"{v:.3f} Bar"
    def from_text(self, t):  return float(t.replace('Bar','').strip())
```

---

## Architecture

```
umm/
├── core/
│   ├── tlv.py          TLV wire format encode/decode
│   ├── messages.py     UMMMsg struct, MType/Channel/UMMSubType enums
│   └── registry.py     Type plugin auto-discovery and matching
├── types/
│   ├── _base.py        BaseUMMType plugin contract, MatchOp enum
│   ├── builtin.py      Built-in types (Int, Char, Bool, UTime, RouterId…)
│   └── manufacturing.py  Temperature, Pressure, ConveyorSpeed, …
├── modules/
│   ├── timer.py        Timer service (mtype=2, or custom)
│   ├── fsm.py          Multi-dimensional Finite State Machine (mtype=5)
│   ├── typedir.py      Type directory service (mtype=10)
│   ├── bridge.py       Inter-router bridge (mtype=11/111)
│   ├── supervisor.py   Process manager (mtype=12)
│   ├── parser.py       Bus monitor / debugger
│   └── watchdog.py     Agent liveness monitor
├── plugins/
│   ├── string_match.py String/regex match plugin
│   └── coord_match.py  Geographic coordinate match plugin
├── capability.py       CapabilityMixin, AgentCapability, make_capability
├── mixins.py           VersionMixin, DebugMixin, ShutdownMixin
├── names.py            Human-readable mtype/subtype name registry
├── router.py           asyncio TCP router
├── agent.py            Synchronous client API (UMMAgent)
└── __init__.py         Public API

umm_gui/
├── server.py           WebSocket bridge (browser ↔ UMM bus) (mtype=13)
├── gui.html            General-purpose console
└── conveyor.html       Sorting machine visualisation

examples/sorting/
├── sorting_demo.py     ConveyorSim, BarcodeSimulator, DiverterSim, BinMonitor
├── sorting.fsm         FSM state machine definition
├── capabilities.py     mtype/subtype constants for the sorting demo
└── umm_names.conf      Human-readable names for parser display

umm_ctl.py              System manager (start/stop/status/show/validate)
umm.conf                System configuration
supervisor.conf         Supervisor agent definitions
umm_names.conf          Global human-readable name registry
```

---

## Compatibility

- Python 3.8 or later required (3.12 recommended)
- Wire format is binary-compatible with the original C++ UMM implementation
- Original C++, Perl, and shell script clients can connect to this Python router

---

## Licence

MIT — © Phil Braham, Realtime Software Pty Ltd. Original concept and design by Phil Braham.
