# Pico-ROS on the Waveshare ESP32-P4-NANO (ESP-IDF, Ethernet)

Barebones Pico-ROS node built straight from this repository's `src/` and submodules (zenoh-pico, Micro-CDR)
as an ESP-IDF component. Nothing is vendored or patched.

* Ethernet: on-board IP101 PHY (MDC 31, MDIO 52, reset 51, addr 1), DHCP
* zenoh client to a ROS 2 `rmw_zenohd` router
* `/picoros/chatter` (std_msgs/String, 1 Hz) published by the board
* `/picoros/listener` (std_msgs/String) subscribed and logged by the board

## Build and flash (ESP-IDF v5.5.x)

```sh
cd examples/esp32p4_nano
idf.py set-target esp32p4
idf.py menuconfig        # "Pico-ROS example": router locator, node name, domain
idf.py -p <port> build flash monitor
```

The component runs a configure-only CMake pass of zenoh-pico to generate `zenoh-pico/config.h` in the build
directory (espidf profile, TCP client, no scouting/multicast, lease 60000 ms x2 to match `rmw_zenohd`).

## ROS 2 side (Lyrical + rmw_zenoh)

```sh
export RMW_IMPLEMENTATION=rmw_zenoh_cpp
ros2 run rmw_zenoh_cpp rmw_zenohd                     # router on tcp/[::]:7447
ros2 node list                                        # /picoros_p4
ros2 topic echo /picoros/chatter
ros2 topic pub /picoros/listener std_msgs/msg/String "{data: hi}"
```

## zenoh-pico version: ESP-IDF needs the TCP timeout fix

Stock zenoh-pico (upstream up to at least 1.10.2, and the `g4sp3r/zenoh-pico` 1.9.0 fork previously pinned
here) ignores the socket timeout in `src/link/transport/tcp/tcp_esp32.c` (`_ZP_UNUSED(tout)`). The single
zenoh-pico executor task then blocks in `recv()` and cannot send keep-alives or flush batches. Observed on
this board against ROS 2 Lyrical / rmw_zenoh_cpp 0.10.6 with the old pin:

* a 30 s stall inside `picoros_interface_init()` after `z_open()`;
* the router expiring the board's lease (`expired after 60000 milliseconds`) ~90 s after connecting and
  every ~60 s after each reconnect, with ~60 s offline each time while `picoros_publish()` still returned
  `PICOROS_OK`.

The submodule therefore points at `BonROV/zenoh-pico` branch `bonrov/1.10.2-espidf`: upstream 1.10.2
(which also contains the ESP-IDF condvar clock fix, eclipse-zenoh/zenoh-pico#1231) plus three patches:

1. `espidf: honour the TCP socket timeout and bound connect()` (cf. Pico-ROS issue #23)
2. `executor hygiene on ESP-IDF: bounded reopen retry, no zero-tick sleeps`
3. `executor: 64-bit wake-up arithmetic on 32-bit targets`
