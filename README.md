# Livox MID360 + KISS-SLAM (ROS2 Jazzy) Startup Guide

Three terminals, run in this order. **Note:** Terminal 1 will fail with `bind failed`
the first time unless Terminal 2's network setup has already been done — read through
once before starting.

---

## Terminal 1 — Livox driver + RViz

Starts the driver that talks to the physical lidar and publishes its data as ROS2 topics.

**Directory:** `~/ws_livox/src/livox_ros_driver2`

```bash
cd ~/ws_livox/src/livox_ros_driver2
source /opt/ros/jazzy/setup.bash
./build.sh jazzy
```

Only re-run `./build.sh` after pulling new code or changing config — not every launch.
A successful build ends with:

```
Finished <<< livox_ros_driver2 [15.0s]
Summary: 1 package finished [15.1s]
```

Now source the install and launch. This must be done from `livox_ros_driver2` (one
level above `launch_ROS2`), since the install path is relative:

```bash
source ../../install/setup.sh
cd launch_ROS2
ros2 launch livox_ros_driver2 rviz_MID360_launch.py
```

**If the network isn't set up yet**, you'll see:

```
[livox_ros_driver2_node-1] bind failed
[livox_ros_driver2_node-1] Failed to init livox lidar sdk.
```

Go do Terminal 2, then come back here.

**Before relaunching**, always clear out any leftover process first — this is the single
most useful command in this whole workflow, since Ctrl+C can leave the driver hanging:

```bash
pkill -f livox_ros_driver2_node
```

**Success looks like:**

```
[livox_ros_driver2_node-1] [INFO] [livox_lidar_publisher]: Init lds lidar success!
[livox_ros_driver2_node-1] successfully set lidar attitude, ip: 192.168.1.145
[livox_ros_driver2_node-1] successfully enable Livox Lidar imu, ip: 192.168.1.145
[livox_ros_driver2_node-1] [INFO] [livox_lidar_publisher]: livox/imu publish use imu format
[livox_ros_driver2_node-1] [INFO] [livox_lidar_publisher]: livox/lidar publish use PointCloud2 format
```

Once you see `livox/lidar publish use PointCloud2 format`, move on to Terminal 3.

You may also see a QoS warning once RViz/SLAM connect — harmless, just a mismatch in
delivery-strictness settings between nodes:

```
[rviz2-2] [WARN] [rviz]: New publisher discovered on topic '/odometry', offering incompatible QoS.
```

**Shortcut:** a script in the home directory (`~`) wraps build + source + launch:

```bash
cd ~
./lidar_ros_setup.sh
```

---

## Terminal 2 — Network setup 

The MID360 lives on `192.168.1.x`. Your wired Ethernet port needs a static IP on that
subnet — this doesn't persist across reboots, so redo it each session.

**Directory:** anywhere (`~` is fine)

Check current IPs — if only a wifi address shows up, the wired port has no IP yet:

```bash
hostname -I
```

Find the wired interface name (ignore `wlp2s0` / anything labeled wireless):

```bash
ip link show
```

Bring it up and assign it an address (replace `enx00e04c680ede` with your interface name):

```bash
sudo ip link set enx00e04c680ede up
sudo ip addr add 192.168.1.50/24 dev enx00e04c680ede
```

Confirm it took:

```bash
ip addr show enx00e04c680ede
hostname -I
```

`hostname -I` should now show both your wifi and wired addresses.

Verify with a ping:

```bash
ping 192.168.1.50
```

If it fails, re-run the `ip addr add` command and ping again — sometimes the interface
needs a moment to settle.

Once ping succeeds, go back to Terminal 1, clear any leftover process, and relaunch:

```bash
pkill -f livox_ros_driver2_node
```

then re-run the `ros2 launch` command from Terminal 1.

---

## Terminal 3 — KISS-SLAM

Only start this once Terminal 1 has shown `livox/lidar publish use PointCloud2 format`.

**Directory:** `~/slam_ws`

```bash
cd ~/slam_ws
source /opt/ros/jazzy/setup.bash
ros2 launch kiss_slam_ros slam.launch.py \
  topic:=/livox/lidar \
  visualize:=true \
  use_sim_time:=true
```

`topic:=/livox/lidar` must match the topic the driver is publishing (confirmed in
Terminal 1's log).

**Success looks like:**

```
[odometry_node-2] [INFO] [odometry_node]: KISS-ICP ROS2 node configured successfully.
[odometry_node-2] [INFO] [odometry_node]: Odometry Node initialized. Waiting for data...
[slam_node-1] [INFO] [slam_node]: SLAM node initialized and waiting for keyframes.
```

RViz opens subscribed to `/global_voxel_map`, `/deskewed_points`, `/odometry`, and
`/global_pose`.

QoS mismatch warnings on these topics are expected and harmless — SLAM still runs fine:

```
[slam_node-1] [WARN] [slam_node]: New subscription discovered on topic 'global_voxel_map', requesting incompatible QoS.
```

**Shutdown:** `Ctrl+C` once and let it terminate on its own (ROS escalates
SIGINT → SIGTERM → SIGKILL automatically over ~15s if needed). A `rcl_shutdown already
called` error during shutdown is a known harmless quirk — not a real problem.

---

## Quick recap

1. **Terminal 1** (`~/ws_livox/src/livox_ros_driver2`): source ROS2 → `./build.sh
   jazzy` → `source install/setup.sh` → `cd launch_ROS2` → `ros2 launch
   livox_ros_driver2 rviz_MID360_launch.py`. `bind failed` → go to step 2.
2. **Terminal 2**: `ip link show` to find your wired interface → `sudo ip link set
   <interface> up` → `sudo ip addr add 192.168.1.50/24 dev <interface>` → confirm with
   `ping 192.168.1.50` → back to Terminal 1: `pkill -f livox_ros_driver2_node` → relaunch.
3. **Terminal 3** (`~/slam_ws`), once Terminal 1 is publishing: source ROS2 → `ros2
   launch kiss_slam_ros slam.launch.py topic:=/livox/lidar visualize:=true
   use_sim_time:=true`.

**Remember:** `pkill -f livox_ros_driver2_node` clears a stuck driver process any time
a launch looks hung or a previous shutdown didn't fully complete.
