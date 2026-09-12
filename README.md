# ESP32 Wi-Fi TCP Socket Server (Echo-style)

This project connects an ESP32 to a Wi-Fi network and then runs an actual
**TCP server** on it: it listens for a single incoming connection, receives
data from that client, prints it, and sends a reply back.

Unlike a client that reaches out to a server, here the ESP32 itself is the
one listening — a PC or another device connects *to* the ESP32's IP address.

---

## How it works (high level)

1. `app_main()` initializes NVS storage, brings up Wi-Fi, and waits until an
   IP address is obtained.
2. `start_socket_server()` opens a TCP socket, binds it to `PORT` on all
   interfaces, and `listen()`s for a connection.
3. It `accept()`s exactly one client connection.
4. It then loops forever: `recv()` from that client, print what arrived, and
   `send()` a reply back.

```
 [PC / client] --connect()--> [ESP32:8080] --accept()--> connected socket
 [PC / client] --send()----->  recv() -> printf()
 [PC / client] <--send()----   ESP32 echoes/replies
```

---

## Configuration constants

| Macro | Purpose |
|---|---|
| `WIFI_SSID` / `WIFI_PASSWORD` | Wi-Fi credentials the ESP32 connects to as a station. |
| `PORT` | TCP port the ESP32 listens on (`8080`). |
| `rx_buffer[128]` | Local (stack) receive buffer, sized for up to 127 bytes of payload plus a null terminator. |

---

## Function-by-function explanation

### `wifi_event_handler(...)`
Callback registered with the ESP-IDF event loop. It reacts to three events:
- **`WIFI_EVENT_STA_START`** – station mode has started, so it calls
  `esp_wifi_connect()` to begin connecting.
- **`WIFI_EVENT_STA_DISCONNECTED`** – Wi-Fi dropped. Clears the "connected"
  bit in the event group, waits 1 second, then retries
  `esp_wifi_connect()`.
- **`IP_EVENT_STA_GOT_IP`** – DHCP assigned an address. Logs the IP and sets
  `WIFI_CONNECTED_BIT` so other tasks know the network is ready.

### `wifi_init(void)`
Brings up Wi-Fi station mode:
1. Creates the event group used to signal connection state.
2. Initializes the network interface and default event loop.
3. Creates the default Wi-Fi station netif.
4. Initializes the Wi-Fi driver with default config.
5. Registers `wifi_event_handler` for both Wi-Fi and IP events.
6. Applies the SSID/password from the macros.
7. Sets station mode and starts Wi-Fi (`esp_wifi_start()`), which triggers
   `WIFI_EVENT_STA_START` and begins the connection attempt.

### `wifi_wait_connected(void)`
Blocks the calling task using `xEventGroupWaitBits()` until
`WIFI_CONNECTED_BIT` is set — i.e., until a real IP address has been
obtained — instead of guessing a fixed startup delay.

### `start_socket_server(void)`
The actual TCP server logic:
1. Builds a `sockaddr_in` bound to `INADDR_ANY` (all interfaces) on `PORT`.
2. **Create**: `socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)` creates a TCP
   socket.
3. **Bind**: `bind()` attaches that socket to the chosen port.
4. **Listen**: `listen(sock, 1)` puts the socket into listening mode with a
   backlog of 1 pending connection.
5. **Accept**: `accept()` blocks until a client connects, returning a new
   socket descriptor (`accepted`) dedicated to that client.
6. Enters an infinite loop:
   - `recv()` reads up to 127 bytes from the client into `rx_buffer`.
   - If data arrived, it's null-terminated and printed.
   - A reply is sent back to the client with `send()`.
   - `vTaskDelay(500 ms)` paces the loop when a message was successfully
     handled.
7. (Unreachable in normal operation — see **Issues** below) closes both
   sockets and returns.

### `app_main(void)`
Entry point called by the ESP-IDF startup code:
1. Initializes NVS flash (erasing/reinitializing it if it's a fresh or
   incompatible version — required by the Wi-Fi driver).
2. Calls `wifi_init()` then `wifi_wait_connected()` to bring up and confirm
   connectivity.
3. Calls `start_socket_server()`, which sets up the listening socket,
   accepts one client, and then serves it forever.

---

## Build & flash (ESP-IDF)

```bash
idf.py set-target esp32
idf.py menuconfig      # optional
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

Update `WIFI_SSID`/`WIFI_PASSWORD` before building. After flashing, check
the serial monitor for the assigned IP (`Got IP: ...`), then connect to it
from your PC:

```bash
nc <esp32-ip> 8080
```

Type something and press enter — you should see it echoed back.

---

## Known issues / things worth fixing
- **Single connection backlog** — `listen(sock, 1)` means only one pending
  connection is queued; fine for testing, but increase it if you expect
  multiple near-simultaneous connection attempts.
- **Hardcoded Wi-Fi credentials/port** — fine for a quick test, but move to
  `menuconfig`/NVS for real deployments.