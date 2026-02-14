# Home Assistant Add-on: KDB-X Tick

## Overview

KDB-X Tick is a high-performance time-series database add-on for Home Assistant. It captures and stores all Home Assistant events, allowing you to query historical data with millisecond precision.

## Installation

### Prerequisites

Before installing this add-on, you need:

1. **KX License**: A valid KDB-X license from KX Systems
2. **kdbx-tick Docker Image**: The base KDB-X tick image must be built locally

### Building the kdbx-tick Image

```bash
cd kdb-tick-docker
source kdbx.env  # Contains KX_BEARER_TOKEN and KX_LICENSE_B64
docker build \
  --build-arg KX_BEARER_TOKEN="${KX_BEARER_TOKEN}" \
  --build-arg KX_LICENSE_B64="${KX_LICENSE_B64}" \
  -t kdbx-tick \
  -f docker/Dockerfile .
```

### Installing the Add-on

1. Add this repository to your Home Assistant add-on repositories
2. Install the "KDB-X Tick" add-on
3. Configure the license (see Configuration section)
4. Start the add-on

## Configuration

### Option: `kx_license_b64`

Your KDB-X license file encoded as base64. To encode your license:

```bash
base64 -w0 < /path/to/kc.lic
```

Copy the output and paste it into the `kx_license_b64` configuration option.

Example configuration:

```yaml
kx_license_b64: "base64-encoded-license-content-here"
```

## Ports

The add-on exposes the following ports:

| Port | Service      | Description                          |
|------|--------------|--------------------------------------|
| 5010 | Tickerplant  | Receives events, maintains log       |
| 5011 | RDB          | Real-time database (today's data)    |
| 5012 | HDB          | Historical database (past data)      |
| 5013 | Gateway      | Unified query interface              |

## Data Storage

All data is stored in the `/data` volume:

- `/data/hass/` - Historical database partitions
- `/data/tplog/` - Tickerplant transaction logs
- `/data/log/` - Application logs

## Schema

Events are stored in the `hass_event` table with the following schema:

```
hass_event:([]
  time:`timestamp$();      / Event timestamp
  sym:`$();                / Domain (sensor, binary_sensor, etc.)
  entity_id:`$();          / Entity identifier
  nvalue:`float$();        / Numeric state value
  svalue:();               / String state value
  eattr:())                / Entity attributes
```

## Publishing Events

Home Assistant events are published to the tickerplant using the `.u.updjson` function:

```q
/ Connect to tickerplant
h:hopen `:localhost:5010

/ Publish an event
payload:"{\"time\": 1704877539.956, \"host\": \"hass_event\", \"event\": {\"domain\": \"sensor\", \"entity_id\": \"temperature\", \"attributes\": {\"unit\": \"C\"}, \"value\": 22.5, \"svalue\": \"22.5\"}}"
h (`.u.updjson; `hass_event; payload)
```

## Querying Data

### Connect to Gateway

The gateway (port 5013) provides unified access to both real-time and historical data:

```q
h:hopen `:localhost:5013
```

### Available Functions

```q
/ Check connection status
h "status[]"

/ Get RDB statistics (record counts, current date)
h "rdbStats[]"

/ Query data (combines RDB and HDB automatically)
/ getData[table; startDate; endDate; symbols]
h "getData[`hass_event; .z.D-7; .z.D; `]"  / Last 7 days, all entities

/ Trigger end-of-day save manually
h "triggerEOD[]"

/ Reconnect to databases
h "reconnect[]"
```

### Direct Queries

You can also connect directly to the RDB or HDB:

```q
/ Connect to RDB for today's data
rdb:hopen `:localhost:5011
rdb "select from hass_event where entity_id=`temperature"

/ Connect to HDB for historical data
hdb:hopen `:localhost:5012
hdb "select from hass_event where date=.z.D-1"
```

## Troubleshooting

### Check Logs

Logs are available in the add-on log viewer or at:
- `/data/log/tick.log` - Tickerplant log
- `/data/log/rdb.log` - RDB log
- `/data/log/hdb.log` - HDB log
- `/data/log/gw.log` - Gateway log

### Verify Services

```bash
# Check if tickerplant is running
echo "1+1" | nc localhost 5010

# List tables
echo "tables[]" | nc localhost 5011
```

### License Issues

If you see license errors:
1. Verify your license is valid and not expired
2. Check the base64 encoding is correct
3. Ensure the license is for the correct architecture (amd64)

## Support

For issues with this add-on, please open an issue at:
https://github.com/peteclarkez/hassio-kdb/issues
