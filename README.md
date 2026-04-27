# flux-swarm — Jetson-Native Swarm Coordinator

**ROS2-compatible swarm stack implementing all 130 multi-agent coordination laws on Jetson Orin.**

## Design Philosophy

Based on the Ignorance Architecture Principle: **coordination requires building walls, not bridges.**

Every design decision follows from experimentally validated laws:

| Law | Design Decision |
|-----|----------------|
| 94-95 | Agents have limited perception (sensor range only) |
| 109 | NO broadcast communication |
| 75 | NO adaptive learning (fixed strategies) |
| 115 | Greedy task assignment (nearest task wins) |
| 128 | Bounded operational areas (geofencing) |
| 86 | Single shared waypoint (K=1 DCS) |
| 42 | Perfect communication only (CRC, retries, no lossy) |

## Architecture

```
┌─────────────────────────────────────────┐
│           Ground Station (Jetson)       │
│  ┌───────────┐  ┌──────────────────┐   │
│  │ flux-core  │  │ interference-    │   │
│  │ (scheduler)│  │ monitor (60%     │   │
│  │            │  │ waste tracker)   │   │
│  └─────┬─────┘  └──────────────────┘   │
│        │                                │
│  ┌─────▼─────────────────────────────┐ │
│  │     DCS Point (single waypoint)   │ │
│  │     Updated only on collection    │ │
│  └─────┬─────────────────────────────┘ │
└────────┼───────────────────────────────┘
         │ (perfect comms only, CRC)
    ┌────▼────┐
    │  Drone  │
    │ Sensors │ ← Limited perception only
    │ Greedy  │ ← Go to nearest task
    │ Fixed   │ ← No learning
    └─────────┘
```

## Key Features

- **No broadcast**: Agents receive single DCS point, nothing else
- **Limited perception**: Agents only see within sensor range (configurable)
- **Perfect comms or no comms**: If packet loss > 0%, disable DCS entirely
- **Geofenced operations**: Bounded areas, not toroidal (Law 128: +8%)
- **Greedy task assignment**: Nearest available task, no global optimization
- **Interference dashboard**: Real-time 60% waste monitoring
- **CUDA-accelerated**: Fleet simulation runs on Jetson GPU

## ROS2 Integration

```python
from flux_swarm import FluxSwarmCoordinator

coordinator = FluxSwarmCoordinator(
    num_agents=16,
    perception_range=12,  # meters
    dcs_enabled=True,
    bounded_area=[(0,0), (100,100)],  # geofence
    perfect_comms=True,
    greedy_only=True,
)

# Subscribe to agent poses
coordinator.register_agents(agent_topic="/poses")

# Publish single DCS waypoint
coordinator.dcs_publisher = rospy.Publisher("/flux/dcs", PoseStamped)

# Monitor interference
coordinator.waste_tracker.start()
```

## Deployment

```bash
# Flash to Jetson Orin
./scripts/flash-jetson.sh

# Launch swarm
ros2 launch flux_swarm swarm.launch.py \
    num_agents:=16 \
    perception_range:=12 \
    bounded_area:="0,0,100,100"
```

## Hardware
- NVIDIA Jetson Orin Nano 8GB (coordinator)
- Any companion computer on agents (Jetson Nano, RPi5)
- Sensors: LiDAR/camera within perception range
- Comms: MAVLink/ROS2 over WiFi (perfect or disabled)

## License
MIT

---

## Fleet Context

Part of the Lucineer/Cocapn fleet. See [fleet-onboarding](https://github.com/Lucineer/fleet-onboarding) for boarding protocol.

- **Vessel:** JetsonClaw1 (Jetson Orin Nano 8GB)
- **Domain:** Low-level systems, CUDA, edge computing
- **Comms:** Bottles via Forgemaster/Oracle1, Matrix #fleet-ops
