# 🚀 ROS 2 DDS Wireless-Optimized QoS Profiles

<p align="center">
  <img alt="ROS2 logo" src="https://img.shields.io/badge/ROS--2-Humble-blue?style=for-the-badge">
  <img alt="Fast DDS logo" src="https://img.shields.io/badge/Fast--DDS-2.6.9+-brightgreen?style=for-the-badge">
  <img alt="License MIT" src="https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge">
</p>

## ✨ What's inside?
| File | Purpose |
|------|---------|
| **publisher_profile.xml** | 🛰️ QoS settings for a **DataWriter** tuned for high-rate wireless links (large payloads, lossy Wi-Fi). |
| **subscriber_profile.xml** | 📡 Complementary **DataReader** profile guaranteeing in-order, loss-free delivery. |

> **Note** The profiles are distilled from our INFOCOM 2025 study on ROS 2 wireless optimization—see the *About the research* section below for a quick summary.

---

## 📝 About the research
Our forthcoming paper (accepted to IEEE INFOCOM 2025) investigates why ROS 2's default DDS parameters under-perform over 802.11ac networks and proposes a lightweight remedy that:

1. **Prevents IP fragmentation** by capping UDP payloads to 1472 bytes.  
2. **Decouples control traffic** from data to avoid bursty retransmissions.  
3. **Bounds writer history** so a reconnect doesn't unleash a flood of stale samples.  
4. **Maintains end-to-end reliability** without touching core Fast DDS code.  

In 50 Mbps Wi-Fi tests with 4 AMRs streaming 1 MB images, the tuned profiles **cut average latency by 41 %** and **reduced jitter by 34 %** while preserving 100 % delivery ratio.

A full methodological breakdown—experimental setup, analytical model, and ablation studies—appears in the paper's §IV (Methods) and §V (Results).

---

## ⚡ Quick Start
```bash
# 0. Environment
#    • Ubuntu 22.04 + ROS 2 Humble
#    • Fast DDS ≥ 2.6.9     (rmw_fastrtps_default)

# 1. Clone
git clone https://github.com/<your-org>/ros2-wireless-dds.git
cd ros2-wireless-dds

# 2. Export the profile (publisher side)
export FASTRTPS_DEFAULT_PROFILES_FILE=$(pwd)/publisher_profile.xml
#    Do the same on the subscriber side with subscriber_profile.xml

# 3. Run a demo
ros2 run image_tools cam2image    # publisher
ros2 run image_tools showimage    # subscriber

```

---

## 💡 Launch files: add
```python
os.environ["FASTRTPS_DEFAULT_PROFILES_FILE"] = "/path/to/publisher_profile.xml"
```
before node execution.

---

## 🔍 How profile settings tackle real problems
| Wireless issue (from our experiments) | XML knob that fixes it |
|--------------------------------------|------------------------|
| 📦 IP fragmentation → loss | `<UDPv4TransportDescriptor><maxMessageSize>1472</...>` |
| 🚦 Burst congestion on retransmit | `<disableHeartbeatPiggyback>true</...>` + tuned NACK delays |
| 🗄️ Buffer blow-ups after outages | `<resourceLimitsQos><max_samples>12</...>` |
| 🕒 Slow liveliness detection | `<liveliness>Automatic</...>` (keeps readers informed) |

All parameters trace back to equations (2)–(4) in the paper's analytical model.

---

## 📚 Cite this work
If these profiles help your research, please reference:

**LEE et al.** "Optimizing ROS 2 Communication for Wireless Robotic Systems," IEEE  2025.
