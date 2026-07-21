# Livox MID360 + KISS-SLAM (ROS2 Jazzy) Startup Guide

This is my own cheat sheet so I stop getting lost every time I start this system up.
You'll need **3 terminals** open. I've written them up in the order you actually run
them in — 1, 2, 3 — but here's the catch: **Terminal 1 will fail the first time** with
a `bind failed` error unless you've already done the network setup in Terminal 2. So
read through everything once before you start, so you're not caught off guard.

---

## Terminal 1 — Start the Livox driver + RViz

This terminal talks to the physical lidar and publishes its data as ROS2 topics.

### Step 1: Get into the right folder

```bash
aria@ARIA2:~$ cd ~/ws_livox/src/livox_ros_driver2
```

I keep messing this up because my brain wants to type `~/livox_ws` instead of
`~/ws_livox` — the words are just swapped. Here's what that looks like when it goes
wrong:

```bash
aria@ARIA2:~/slam_ws$ cd ~/livox_ws/src/Livox-SDK2
aria@ARIA2:~/livox_ws/src/Livox-SDK2$ cd ws_livox
bash: cd: ws_livox: No such file or directory
```

If you get turned around like this, just start fresh from home and step in one folder
at a time so you can see what's actually there:

```bash
aria@ARIA2:~$ cd ~/ws_livox
aria@ARIA2:~/ws_livox$ ls
build  install  log  src
aria@ARIA2:~/ws_livox$ cd src
aria@ARIA2:~/ws_livox/src$ ls
livox_ros_driver2
aria@ARIA2:~/ws_livox/src$ cd livox_ros_driver2
```

### Step 2: Load ROS2 and build the driver

```bash
aria@ARIA2:~/ws_livox/src/livox_ros_driver2$ source /opt/ros/jazzy/setup.bash
aria@ARIA2:~/ws_livox/src/livox_ros_driver2$ ./build.sh jazzy
```

A successful build ends with something like this:

```
Finished <<< livox_ros_driver2 [15.0s]

Summary: 1 package finished [15.1s]
  1 package had stderr output: livox_ros_driver2
```

Don't panic about the CMake warning above it mentioning `FLANN_ROOT` and `CMP0144` —
that's just CMake being chatty about a policy setting, it's not an actual error.

Also, you don't need to run `./build.sh` every single time you want to launch — only
after you've changed code or pulled updates. Otherwise you can skip straight to
launching.

### Step 3: Move into the launch folder and load the install

```bash
aria@ARIA2:~/ws_livox/src/livox_ros_driver2$ cd launch_ROS2
```

Don't try to `cd launch_ROS2` a second time once you're already inside it — there's no
folder inside itself:

```bash
aria@ARIA2:~/ws_livox/src/livox_ros_driver2/launch_ROS2$ cd launch_ROS2
bash: cd: launch_ROS2: No such file or directory
```

This next part trips me up every time: the setup script has to be sourced from
**one level up** (from `livox_ros_driver2`, not from inside `launch_ROS2`), because the
path `../../install` is written relative to that outer folder:

```bash
aria@ARIA2:~/ws_livox/src/livox_ros_driver2/launch_ROS2$ source ../../install/setup.sh
bash: ../../install/setup.sh: No such file or directory
```

So back out one directory, source it there, then step back into `launch_ROS2`:

```bash
aria@ARIA2:~/ws_livox/src/livox_ros_driver2/launch_ROS2$ cd ..
aria@ARIA2:~/ws_livox/src/livox_ros_driver2$ source ../../install/setup.sh
aria@ARIA2:~/ws_livox/src/livox_ros_driver2$ cd launch_ROS2
```

### Step 4: Launch it

```bash
aria@ARIA2:~/ws_livox/src/livox_ros_driver2/launch_ROS2$ ros2 launch livox_ros_driver2 rviz_MID360_launch.py
```

### If the network isn't set up yet, here's what you'll see

```
[livox_ros_driver2_node-1] config lidar type: 8
[livox_ros_driver2_node-1] successfully parse base config, counts: 1
[livox_ros_driver2_node-1] bind failed
[livox_ros_driver2_node-1] Failed to init livox lidar sdk.
[livox_ros_driver2_node-1] [ERROR] [...] [livox_lidar_publisher]: Init lds lidar fail!
```

RViz will still open and say it's subscribing to `/livox/lidar`, but nothing will ever
show up, because the driver couldn't even open a connection to the lidar. If you see
this, don't waste time troubleshooting here — go straight to **Terminal 2** below and
fix the network, then come back.

### Before you relaunch — kill any leftover process (this one's a lifesaver)

If you've already tried launching once and hit Ctrl+C, the driver process can be slow
to actually die (you'll see ROS escalate from SIGINT → SIGTERM → SIGKILL over several
seconds), and sometimes it lingers even after that. Rather than waiting around or
guessing whether it's really gone, just kill it directly before you try again:

```bash
aria@ARIA2:~$ pkill -f livox_ros_driver2_node
```

This is genuinely one of the most useful commands in this whole workflow — run it
any time a launch looks stuck, hung, or weird before your next attempt, and you'll save
yourself a lot of confused re-launching.

### What success actually looks like

Once the network (Terminal 2) is sorted and you re-run the same launch command:

```
[livox_ros_driver2_node-1] config lidar type: 8
[livox_ros_driver2_node-1] successfully parse base config, counts: 1
[livox_ros_driver2_node-1] [INFO] [...] [livox_lidar_publisher]: Init lds lidar success!
...
[livox_ros_driver2_node-1] GetFreeIndex key:livox_lidar_2432805056.
[livox_ros_driver2_node-1] successfully set lidar attitude, ip: 192.168.1.145
[livox_ros_driver2_node-1] successfully enable Livox Lidar imu, ip: 192.168.1.145
[livox_ros_driver2_node-1] [INFO] [...] [livox_lidar_publisher]: livox/imu publish use imu format
[livox_ros_driver2_node-1] [INFO] [...] [livox_lidar_publisher]: livox/lidar publish use PointCloud2 format
```

That last line — `livox/lidar publish use PointCloud2 format` — is your green light to
move on to Terminal 3.

You might also see a warning like this once RViz or the SLAM node connects. It's
harmless, just a mismatch in how strict two nodes are about message delivery
(QoS = "quality of service"):

```
[rviz2-2] [WARN] [...] [rviz]: New publisher discovered on topic '/odometry', offering incompatible QoS. No messages will be sent to it. Last incompatible policy: RELIABILITY_QOS_POLICY
```

### The shortcut version

There's a script sitting in the home folder that basically does the build + source +
launch for you in one shot:

```bash
aria@ARIA2:~$ ./lidar_ros_setup.sh
```

Handy once everything's working and you just want to get moving quickly.

---

## Terminal 2 — Set up the network (only needed if Terminal 1 says `bind failed`)

The MID360 lidar lives on the `192.168.1.x` network. For your computer to talk to it,
your **wired** Ethernet port needs its own address on that same network — it won't have
one by default. Heads up: this doesn't survive a reboot, so you'll be doing this every
time you power the machine back on.

### Step 1: See what IPs you currently have

```bash
aria@ARIA2:~$ hostname -I
10.1.162.81
```

Only the wifi address shows up here — which confirms the wired port has no IP yet, and
explains exactly why Terminal 1 couldn't connect.

### Step 2: Figure out which interface is the wired one

```bash
aria@ARIA2:~$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> ...
2: wlp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...        # wifi — not this one
3: enx00e04c680ede: <BROADCAST,MULTICAST,UP,LOWER_UP> ...   # wired — this is the lidar port
```

`wlp2s0` is wifi, ignore it. The one starting with `enx...` (yours will have a
different string after it, based on the adapter's MAC address) is the physical Ethernet
port the lidar is plugged into.

You can also double check with:

```bash
aria@ARIA2:~$ sudo lshw -class network
```

and look for the one labeled as a wired/Ethernet device rather than "Wireless interface."

### Step 3: Turn the interface on and give it an address

```bash
aria@ARIA2:~$ sudo ip link set enx00e04c680ede up
aria@ARIA2:~$ sudo ip addr add 192.168.1.50/24 dev enx00e04c680ede
```

### Step 4: Make sure it actually took

```bash
aria@ARIA2:~$ ip addr show enx00e04c680ede
3: enx00e04c680ede: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.1.50/24 scope global enx00e04c680ede
       valid_lft forever preferred_lft forever
```

```bash
aria@ARIA2:~$ hostname -I
10.1.162.81 192.168.1.50
```

Now you've got both your wifi address and your new wired address.

### Step 5: Ping it to be sure

```bash
aria@ARIA2:~$ ping 192.168.1.50
64 bytes from 192.168.1.50: icmp_seq=1 ttl=64 time=0.042 ms
64 bytes from 192.168.1.50: icmp_seq=2 ttl=64 time=0.070 ms
```

Getting replies means the interface is genuinely live. Hit `Ctrl+C` to stop the ping
once you've seen a couple of successful replies.

> **A gotcha I ran into:** the first time I added the IP, the ping came back with 100%
> packet loss — the interface just hadn't settled yet. Running `sudo ip addr add
> 192.168.1.50/24 dev enx00e04c680ede` a second time right before pinging again fixed
> it. So if your ping fails, just repeat steps 3 through 5 once.

### Step 6: Head back to Terminal 1

Once the ping works, switch back over to Terminal 1. If a driver process is still
sitting there from a previous attempt, kill it first:

```bash
aria@ARIA2:~$ pkill -f livox_ros_driver2_node
```

Then re-run the launch command:

```bash
ros2 launch livox_ros_driver2 rviz_MID360_launch.py
```

This time it should say `Init lds lidar success!` instead of `bind failed`.

---

## Terminal 3 — Start KISS-SLAM

Only start this once Terminal 1 has actually shown `livox/lidar publish use PointCloud2
format` — meaning the driver is confirmed to be streaming real data.

### Step 1: Get into the SLAM workspace and load ROS2

```bash
aria@ARIA2:~$ cd ~/slam_ws
aria@ARIA2:~/slam_ws$ source /opt/ros/jazzy/setup.bash
```

### Step 2: Launch

```bash
aria@ARIA2:~/slam_ws$ ros2 launch kiss_slam_ros slam.launch.py \
  topic:=/livox/lidar \
  visualize:=true \
  use_sim_time:=true
```

The `topic:=/livox/lidar` part needs to match whatever topic the driver is actually
publishing to — which we already confirmed in Terminal 1's log output.

### What a good launch looks like

```
[odometry_node-2] [INFO] [...] [odometry_node]: KISS-ICP ROS2 node configured successfully.
[odometry_node-2] [INFO] [...] [odometry_node]: Odometry Node initialized with odom_frame: odom, base_frame: base_link
[odometry_node-2] [INFO] [...] [odometry_node]: Odometry Node initialized. Waiting for data...
[slam_node-1] [INFO] [...] [slam_node]: KISS-SLAM configuration:
[slam_node-1]   Map Frame: map
[slam_node-1]   Odometry Frame: odom
...
[slam_node-1] [INFO] [...] [slam_node]: SLAM node initialized and waiting for keyframes.
```

RViz will pop open subscribed to `/global_voxel_map`, `/deskewed_points`, `/odometry`,
and `/global_pose`.

You'll almost certainly see a bunch of QoS warnings like this one:

```
[slam_node-1] [WARN] [...] [slam_node]: New subscription discovered on topic 'global_voxel_map', requesting incompatible QoS. No messages will be sent to it. Last incompatible policy: RELIABILITY
```

Don't worry about these — SLAM keeps running fine. It just means two nodes disagree on
how strictly messages need to be delivered, and it's a separate config fix, not
something blocking startup.

### Shutting it down

Hit `Ctrl+C` once and let it shut down on its own. You might see something like this in
the terminal — it's a known harmless quirk in how `rclpy` handles shutdown twice, not
something you broke:

```
[slam_node-1] rclpy._rclpy_pybind11.RCLError: failed to shutdown: rcl_shutdown already called on the given context, at ./src/rcl/init.c:333
```

If a node refuses to close, ROS will automatically escalate from SIGINT to SIGTERM to
SIGKILL over about 15 seconds — you don't have to do anything, just let it finish. But
if you're impatient (or it's really stuck), you can force it the same way we did in
Terminal 1:

```bash
pkill -f livox_ros_driver2_node
```

---

## Quick recap — the whole flow start to finish

1. **Terminal 1:** go to `~/ws_livox/src/livox_ros_driver2` → source ROS2 → `./build.sh
   jazzy` → `cd launch_ROS2` → step back out one folder and `source
   ../../install/setup.sh` → step back in → `ros2 launch livox_ros_driver2
   rviz_MID360_launch.py`. If you get `bind failed`, jump to step 2.
2. **Terminal 2:** find your wired interface with `ip link show` → bring it up with
   `sudo ip link set <interface> up` → give it an address with `sudo ip addr add
   192.168.1.50/24 dev <interface>` → confirm with `ping 192.168.1.50` → go back to
   Terminal 1, run `pkill -f livox_ros_driver2_node` if anything's lingering, then
   relaunch.
3. **Terminal 3:** once Terminal 1 shows `Init lds lidar success!` and is publishing,
   go to `~/slam_ws` → source ROS2 → `ros2 launch kiss_slam_ros slam.launch.py
   topic:=/livox/lidar visualize:=true use_sim_time:=true`.

And remember: **`pkill -f livox_ros_driver2_node` is your friend** — reach for it any
time a launch looks stuck or you're not sure a previous attempt actually shut down.
