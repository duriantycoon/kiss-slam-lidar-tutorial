# Livox MID360 + KISS-SLAM (ROS2 Jazzy) Startup Guide

This is my personal reference so I stop getting confused about the order of operations.
There are **3 terminals**. Order matters — the network has to be set up (Terminal 2)
*before* the Livox driver will bind successfully in Terminal 1.

---

## Terminal 2 — Network setup (do this FIRST)

The Livox MID360 sits on `192.168.1.x`. Your PC's Ethernet port needs a static IP on
that same subnet before the driver can open a socket to the lidar.

```bash
# Check current IPs / confirm you're not already on the right subnet
hostname -I

# Find your Ethernet interface name (NOT the wifi one, e.g. wlp2s0)
ip link show
```

Look for the wired adapter (in my case `enx00e04c680ede`).

```bash
# Bring the interface up
sudo ip link set enx00e04c680ede up

# Give it a static IP on the lidar's subnet
sudo ip addr add 192.168.1.50/24 dev enx00e04c680ede

# Confirm the IP took
ip addr show enx00e04c680ede

# Sanity check — should get replies once the interface is really up
ping 192.168.1.50
```

**Why:** Without this, `livox_ros_driver2` prints `bind failed` / `Failed to init livox
lidar sdk` because it can't bind to the `192.168.1.x` network the lidar expects. This IP
assignment doesn't persist across reboots, so it has to be redone every time (or turned
into a netplan/systemd-networkd config later).

---

## Terminal 1 — Livox ROS2 driver + RViz

```bash
cd ~/ws_livox/src/livox_ros_driver2
source /opt/ros/jazzy/setup.bash

# Build (only needed after code changes / fresh clone)
./build.sh jazzy

cd launch_ROS2
source ../../install/setup.sh

ros2 launch livox_ros_driver2 rviz_MID360_launch.py
```

**Reasoning / gotchas:**
- Workspace is `~/ws_livox`, not `~/livox_ws` — easy to mistype.
- `source ../../install/setup.sh` must be run **after** `./build.sh` and **from inside**
  `~/ws_livox/src/livox_ros_driver2` (so `../../install` resolves to `~/ws_livox/install`).
- If you see:
  ```
  bind failed
  Failed to init livox lidar sdk.
  ```
  → the network isn't set up yet. Go do Terminal 2, then re-run this launch command.
- Success looks like:
  ```
  Init lds lidar success!
  ...
  livox/imu publish use imu format
  livox/lidar publish use PointCloud2 format
  ```
- There's also a `lidar_ros_setup.sh` convenience script in the home directory
  (`~/lidar_ros_setup.sh`) that wraps the build + launch step — run it from `~` with
  `./lidar_ros_setup.sh`.

---

## Terminal 3 — KISS-SLAM

Once Terminal 1 shows `Init lds lidar success!` and is actively publishing:

```bash
cd ~/slam_ws
source /opt/ros/jazzy/setup.bash

ros2 launch kiss_slam_ros slam.launch.py \
  topic:=/livox/lidar \
  visualize:=true \
  use_sim_time:=true
```

**Reasoning / gotchas:**
- `topic:=/livox/lidar` must match the topic the driver is actually publishing on.
- QoS mismatch warnings like:
  ```
  New publisher discovered on topic '/odometry', offering incompatible QoS.
  ```
  are cosmetic/non-fatal here (best-effort vs reliable) — SLAM still runs, RViz just
  won't display those specific topics until QoS is aligned. Safe to ignore for now.
- Shutdown errors on Ctrl+C (`rcl_shutdown already called on the given context`) are a
  known harmless quirk of `rclpy.shutdown()` being called twice during teardown.

---

## TL;DR startup order

1. **Terminal 2:** bring up Ethernet interface + assign `192.168.1.50/24`, confirm with ping.
2. **Terminal 1:** build (if needed) → source install → launch `livox_ros_driver2`.
   Confirm `Init lds lidar success!` before moving on.
3. **Terminal 3:** launch `kiss_slam_ros` pointed at `/livox/lidar`.
