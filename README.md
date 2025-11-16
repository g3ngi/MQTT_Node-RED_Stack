
# MQTT + Node-RED Stack for IoT Garden Sprinkler

This repository contains a minimal, production-ish setup for:

- **Eclipse Mosquitto** – MQTT broker with:
  - persistent data
  - username/password authentication
  - ACL-based topic authorization
- **Node-RED** – backend + dashboard for your IoT flows (e.g. garden sprinkler control, soil moisture monitoring, scheduling).

Everything runs via **Docker Compose**.

---

## 1. Project Structure

After cloning the repo, the relevant files look like this:

```text
.
├── docker-compose.yaml
└── mosquitto/
    ├── mosquitto.conf   # Mosquitto broker configuration
    ├── passwords        # Username/password file (hashed)
    └── acl              # ACL rules for topic access
```

Named Docker volumes will be created automatically:

- `mosqdata` → `/mosquitto/data` (persistence)
- `mosqlog`  → `/mosquitto/log` (logs)
- `nodered_data` (directory) → `/data` inside Node-RED

---

## 2. Prerequisites

- Docker
- Docker Compose (or `docker compose` plugin)
- Basic terminal access

On macOS / Linux you can verify:

```bash
docker --version
docker compose version
```

---

## 3. Docker Compose Overview

The key parts of `docker-compose.yaml` (simplified):

```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mqtt_server-mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"          # MQTT (plain TCP)
      # later you can add TLS: - "8883:8883"
    volumes:
      - ./mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf:ro
      - ./mosquitto/passwords:/mosquitto/config/passwords:ro
      - ./mosquitto/acl:/mosquitto/config/acl:ro
      - mosqdata:/mosquitto/data
      - mosqlog:/mosquitto/log
    environment:
      - TZ=Asia/Jakarta

  # Example Node-RED service (adjust to match your actual compose)
  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"     # Node-RED editor & dashboard
    volumes:
      - ./nodered_data:/data
    environment:
      - TZ=Asia/Jakarta
    depends_on:
      - mosquitto

volumes:
  mosqdata: {}
  mosqlog: {}
```

> **Note:**  
> The Mosquitto config/passwords/acl files are mounted **read-only** (`:ro`) from the `mosquitto/` folder.

---

## 4. Mosquitto Configuration

### 4.1 `mosquitto.conf`

The provided `mosquitto/mosquitto.conf` should look like:

```conf
persistence true
persistence_location /mosquitto/data/

listener 1883
allow_anonymous false
password_file /mosquitto/config/passwords
acl_file /mosquitto/config/acl

# Helpful logs to stdout
log_dest stdout
log_type error
log_type warning
log_type notice
log_type information
log_type all
```

Key points:

- **No anonymous access**: `allow_anonymous false`
- **Credentials** are read from `/mosquitto/config/passwords`
- **ACL rules** are read from `/mosquitto/config/acl`
- Data is persisted under `/mosquitto/data/` (mapped to the `mosqdata` volume)

---

## 5. Setting Up Authentication

### 5.1 Create a User & Password

We’ll use the Mosquitto image itself to generate the password file.

From the repo root (where `docker-compose.yaml` lives):

```bash
# First-time creation: use -c (creates/overwrites passwords file)
docker run --rm -it \
  -v "$(pwd)/mosquitto:/mosquitto/config" \
  eclipse-mosquitto:2 \
  mosquitto_passwd -c /mosquitto/config/passwords your_username
```

You’ll be prompted:

```text
Password:
Reenter password:
```

To add more users **later** (do not use `-c`):

```bash
docker run --rm -it \
  -v "$(pwd)/mosquitto:/mosquitto/config" \
  eclipse-mosquitto:2 \
  mosquitto_passwd /mosquitto/config/passwords another_user
```

The `mosquitto/passwords` file now contains hashed credentials and will be used by the broker at startup.

---

## 6. Setting Up ACLs

The `mosquitto/acl` file controls which users can publish/subscribe to which topics.

Basic syntax:

```text
# Allow full access for a specific user
user your_username
topic readwrite #

# Another user with limited access
user sensor_node
topic read sensors/#
topic write commands/sensors/#

# Read-only monitoring user
user monitor
topic read #
```

Adjust this file for your project’s topic hierarchy.

---

## 7. Starting the Stack

From the repo root:

```bash
# Start Mosquitto and Node-RED
docker compose up -d
```

Check running containers:

```bash
docker ps
```

View Mosquitto logs:

```bash
docker compose logs -f mosquitto
```

Stop everything:

```bash
docker compose down
```

> To also remove named volumes (all data!):
> ```bash
> docker compose down -v
> ```

---

## 8. Testing the MQTT Broker

### 8.1 Using `mosquitto_pub` / `mosquitto_sub`

From your host (with `mosquitto-clients` installed) or inside a test container:

#### Subscribe:

```bash
mosquitto_sub -h localhost -p 1883 \
  -u your_username -P 'your_password' \
  -t 'test/topic' -v
```

#### Publish:

```bash
mosquitto_pub -h localhost -p 1883 \
  -u your_username -P 'your_password' \
  -t 'test/topic' -m 'hello from docker'
```

You should see the message appear in the subscriber terminal.

---

## 9. Connecting Node-RED to MQTT

In Node-RED (via `http://localhost:1880`):

1. Drag an **MQTT in** or **MQTT out** node onto the canvas.
2. Click the node → edit → click the **pencil icon** next to the Server field.
3. Configure:

   - **Server**: `mosquitto` (service name inside docker network)  
     or `host.docker.internal` / `localhost` depending on how you run it.
   - **Port**: `1883`
   - **Security**:
     - Username: `your_username`
     - Password: `your_password`

4. Deploy.
5. Use debug nodes to verify messages flow properly.

---

## 10. Troubleshooting

### 10.1 “not a directory” mount errors

If you see errors like:

> `Are you trying to mount a directory onto a file (or vice-versa)?`

Check that these host paths exist and are **files**, not directories:

```text
./mosquitto/mosquitto.conf   # file
./mosquitto/passwords        # file
./mosquitto/acl              # file
```

On macOS / Linux:

```bash
ls -l mosquitto
```

You want lines starting with `-rw` (file), **not** `drwx` (directory).

If needed, recreate:

```bash
rm -rf mosquitto/passwords mosquitto/acl
touch mosquitto/passwords mosquitto/acl
```

Then re-run the `mosquitto_passwd` command.

---

## 11. How This Fits the IoT Garden Sprinkler

This stack is intended to be the **backend** for an IoT Garden Sprinkler system:

- ESP32/MCU nodes publish:
  - soil moisture sensor data → `garden/zoneX/sensor/moisture`
  - valve status → `garden/zoneX/sensor/valve_state`
- Node-RED:
  - subscribes to those topics
  - applies rules/schedules (time-based, moisture-based)
  - publishes control commands → `garden/zoneX/control/valve`

Because the broker is secured (no anonymous access, ACLs per user/topic), you can:

- keep sensor/control topics isolated per zone
- expose a dashboard to family members
- safely integrate automation logic without exposing an open MQTT broker to the world

---
