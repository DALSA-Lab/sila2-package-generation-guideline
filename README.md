# How to Create a SiLA2 Server and Client Integration for Any Lab Instrument

This guide walks through building a SiLA2 integration from scratch. A plate reader is used as an
example device, but the same steps apply to any instrument (liquid handler, centrifuge, incubator,
spectrophotometer, etc.).

---

## Background: What is a Feature Definition?

A **Feature Definition** is the contract between your instrument and the rest of the lab automation
world. It describes, in a high-level, device-neutral way, what the instrument can *do* — not *how*
it does it internally.

From the SiLA 2 specification:

> *"The Feature Definition describes a certain behavior of a Feature in an exact and very specific
> way. It MUST contain an extensive Feature Description of the behavior it models. Constraints,
> like under which preconditions a Command should be called, valid Parameter ranges and any other
> dependencies MUST be detailed. In case the behavior that the Feature is describing is exporting
> a state, all states and their transitions MUST be described."*

A Feature Definition contains:

- **Feature details** — identifier, display name, description
- **Commands** — actions the instrument can perform (unobservable or observable)
- **Properties** — values the instrument exposes (static or observable/streaming)
- **SiLA Data Types** — custom structured types used by commands/properties
- **Execution Errors** — specific error conditions that can occur
- **Client Metadata** — context the client can attach to any request

Reference: [SiLA Standard Documentation](https://sila-standard.com/standards/)

---

## Important Design Principle: High-Level vs. Low-Level

Feature Definitions model **instrument behavior from the user's perspective**, not the vendor API.

| Feature Definition (high-level) | Vendor SDK (low-level) |
|---|---|
| `StartRead(ProtocolName, PlateId)` | `sdk.run_assay("SID_001", plate=4, gain=100, ...)` |
| `OpenTray()` | `sdk.send_command(0x03, 0x01)` |
| `SetTemperature(TemperatureCelsius)` | `sdk.write_register(REG_TEMP, value * 10)` |

Your feature implementations bridge the two: they accept high-level SiLA calls and translate them
into specific vendor SDK calls inside an **adapter/client module**.

---

## Step 1 — Write Your Feature Definition File

Start by creating a `.sila.xml` file for each logical capability of your instrument. The XML files
in `feature_definitions/` in this folder are **demonstration examples only** — do not use them
directly. Use them as a reference for the XML structure, then write your own files that reflect
your instrument's actual capabilities.

**Recommended editors:**

- Visual Studio (with XML schema support) — gives inline validation and autocomplete
- VS Code with the [XML extension](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-xml)
- Any XML editor that can load the SiLA2 XSD schema

Because the example files in this folder were generated from and validated against the SiLA2 schema,
your editor will flag any structural mistakes (wrong element names, missing required fields, etc.)
as soon as you deviate from the specification — which is the fastest way to learn the format.

### Anatomy of a Feature Definition

```xml
<Feature SiLA2Version="1.0" FeatureVersion="1.0" MaturityLevel="Draft"
         Originator="org.example" Category="examples"
         xmlns="http://www.sila-standard.org"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.sila-standard.org
           https://gitlab.com/SiLA2/sila_python/-/raw/main/schema/FeatureDefinition.xsd">

  <Identifier>PlateReaderDeviceControl</Identifier>
  <DisplayName>Plate Reader Device Control</DisplayName>
  <Description>
    Controls the physical state of a plate reader instrument.
    Call InitializeDevice once at startup before using any other command.
    OpenTray and CloseTray move the sample carrier. SetTemperature configures
    the incubation temperature; valid range is 4–45 °C.
  </Description>

  <!-- Commands -->
  <Command>
    <Identifier>InitializeDevice</Identifier>
    <DisplayName>Initialize Device</DisplayName>
    <Description>Initializes the device and homes all axes. Must be called before any read.</Description>
    <Observable>No</Observable>
  </Command>

  <!-- Observable Command (long-running, client polls for completion) -->
  <Command>
    <Identifier>StartRead</Identifier>
    <DisplayName>Start Read</DisplayName>
    <Description>Starts a plate read using the given protocol. Returns a result ID when done.</Description>
    <Observable>Yes</Observable>
    <Parameter>
      <Identifier>ProtocolName</Identifier>
      <DisplayName>Protocol Name</DisplayName>
      <Description>Name of the measurement protocol configured in the reader software.</Description>
      <DataType><Basic>String</Basic></DataType>
    </Parameter>
    <Response>
      <Identifier>ResultId</Identifier>
      <DisplayName>Result ID</DisplayName>
      <Description>Identifier for the completed read result.</Description>
      <DataType><Basic>String</Basic></DataType>
    </Response>
  </Command>

  <!-- Observable Property (streaming value) -->
  <Property>
    <Identifier>DeviceStatus</Identifier>
    <DisplayName>Device Status</DisplayName>
    <Description>Current operational status of the device (IDLE, BUSY, ERROR).</Description>
    <Observable>Yes</Observable>
    <DataType><Basic>String</Basic></DataType>
  </Property>

</Feature>
```

**Key rules:**
- `<Observable>Yes</Observable>` on a Command means it is a long-running operation; the client
  receives a handle and must poll for completion.
- `<Observable>Yes</Observable>` on a Property means the client can subscribe to a live stream
  of values.
- `<Description>` fields should include pre-conditions, valid ranges, and error states — not just
  a one-liner. This is what operators and integrators read to understand safe usage.

---

## Step 2 — Generate the Python Package

Once your `.sila.xml` files are ready, use `sila2-codegen` to generate the full server/client
scaffolding:

```powershell
sila2-codegen new-package `
  --package-name plate_reader_sila `
  PlateReaderAcquisition.sila.xml PlateReaderDeviceControl.sila.xml
```

Replace `plate_reader_sila` with your desired Python package name and list all your feature
definition files as positional arguments.

This generates:

```
plate_reader_sila/
  __init__.py
  __main__.py          ← CLI entry point (--ip-address, --port, etc.)
  server.py            ← Wires feature implementations into the SiLA server
  generated/           ← Auto-generated gRPC/protobuf + base classes (DO NOT EDIT)
    platereaderacquisition/
      platereaderacquisition_base.py   ← Abstract base class you must implement
      platereaderacquisition_client.py
      platereaderacquisition_types.py
    platereaderdevicecontrol/
      ...
  feature_implementations/
    platereaderacquisition_impl.py     ← YOUR implementation code goes here
    platereaderdevicecontrol_impl.py
```

> Re-run codegen whenever you change a feature definition file. The `generated/` directory is
> always overwritten; your code in `feature_implementations/` is preserved.

---

## Step 3 — Implement the Feature Methods

Open each file in `feature_implementations/` and fill in the abstract methods. Each method
corresponds to a Command or Property you defined in the XML.

If your instrument has a vendor SDK or service client (SOAP, REST, serial, etc.), create an
**adapter module** (e.g., `plate_reader_service/client.py`) that wraps it, then inject it into
the implementations:

```python
# feature_implementations/platereaderdevicecontrol_impl.py

from ..generated.platereaderdevicecontrol.platereaderdevicecontrol_base import PlateReaderDeviceControlBase
from ..generated.platereaderdevicecontrol.platereaderdevicecontrol_types import (
    InitializeDevice_Responses, OpenTray_Responses, SetTemperature_Responses
)

class PlateReaderDeviceControlImpl(PlateReaderDeviceControlBase):
    def __init__(self, device_client):
        self._client = device_client  # your adapter

    def InitializeDevice(self, *, metadata):
        self._client.initialize()         # vendor SDK call
        self._client.wait_until_idle()    # block until motion stops
        return InitializeDevice_Responses()

    def OpenTray(self, *, metadata):
        self._client.open_tray()
        self._client.wait_until_idle()
        return OpenTray_Responses()

    def SetTemperature(self, *, metadata, TemperatureCelsius):
        self._client.set_temperature(TemperatureCelsius.value)
        return SetTemperature_Responses()
```

For observable commands, `begin_execution()` must be called first:

```python
def StartRead(self, *, metadata, instance, ProtocolName, PlateId):
    instance.begin_execution()
    self._client.start_read(ProtocolName.value, PlateId.value)
    self._client.wait_until_idle()
    return StartRead_Responses(ResultId=Typed.String(self._client.get_last_result_id()))
```

---

## Step 4 — Wire Into the Server

In `server.py`, instantiate your adapter and pass it to each feature implementation:

```python
from .plate_reader_service.client import PlateReaderClient
from .feature_implementations.platereaderdevicecontrol_impl import PlateReaderDeviceControlImpl

device = PlateReaderClient(host="192.168.1.10", port=8080)
server.set_feature_implementation(PlateReaderDeviceControlImpl(device))
```

---

## Step 5 — Run the Server

```powershell
python -m plate_reader_sila --insecure --ip-address 127.0.0.1 --port 50052
```

---

## Step 6 — Build a Client Script

```python
from sila2.client import SilaClient

client = SilaClient("127.0.0.1", 50052, insecure=True)
client.PlateReaderDeviceControl.InitializeDevice()
client.PlateReaderDeviceControl.OpenTray()

# observable command — server blocks; client polls for completion
cmd = client.PlateReaderAcquisition.StartRead(ProtocolName="Assay_001", PlateId="P001")
while not cmd.done:
    pass
result = cmd.get_responses()
print("Result ID:", result.ResultId)
```

---

## Further Reading

- [SiLA 2 Standard Documentation](https://sila-standard.com/standards/)
- [sila_python library (GitLab)](https://gitlab.com/SiLA2/sila_python)
- The `mantis_sila` project in this repository is a complete real-world example of this pattern.
