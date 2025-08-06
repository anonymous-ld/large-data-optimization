# 🚀 DDS Optimizer for Wirelss Transfer

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

## ⚡ DDS_Optimizer.py
```python3
#!/usr/bin/env python3
import sys
import math

def parse_args(args):
    """Parse command-line arguments"""
    params = {}
    for arg in args:
        if '=' in arg:
            key, val = arg.split('=')
            params[key.strip()] = float(val.strip())
    return params

def compute_parameters(params):
    """Compute QoS parameters based on input"""
    r = params.get("r", 10)           # Publish rate [Hz]
    u = params.get("u", 100000)       # Payload size [bytes]
    T = params.get("T", 1e8)          # OS-to-Link throughput [bytes/sec]
    w = params.get("w", 0.5)          # Link utilization (parsed but not used here)

    retrans_ns = int(1e9 / (2 * r))   # Heartbeat period in nanoseconds
    history_cache = math.floor(T / u) # History cache size = floor(T / u)

    return {
        "r": int(r),
        "u": int(u),
        "T": int(T),
        "w": w,
        "retrans_ns": retrans_ns,
        "history_cache": history_cache
    }

def generate_publisher_xml(params, output_file):
    """Generate XML configuration for the publisher"""
    xml_content = f"""<?xml version="1.0" encoding="UTF-8" ?>
<dds xmlns="http://www.eprosima.com">
    <profiles>
        <transport_descriptors>
            <transport_descriptor>
                <transport_id>udp_transport</transport_id>
                <type>UDPv4</type>
                <maxMessageSize>1472</maxMessageSize>
            </transport_descriptor>
        </transport_descriptors>
        <participant profile_name="optimizer_participant_pub_profile" is_default_profile="true">
            <rtps>
                <userTransports>
                    <transport_id>udp_transport</transport_id>
                </userTransports>
                <useBuiltinTransports>false</useBuiltinTransports>
            </rtps>
        </participant>
        <publisher profile_name="optimizer_publisher_profile" is_default_profile="true">
            <topic>
                <historyQos>
                    <kind>KEEP_ALL</kind>
                </historyQos>
                <resourceLimitsQos>
                    <max_samples>{params['history_cache']}</max_samples>
                    <max_instances>10</max_instances>
                    <max_samples_per_instance>{params['history_cache']}</max_samples_per_instance>
                </resourceLimitsQos>
            </topic>
            <qos>
                <disable_heartbeat_piggyback>true</disable_heartbeat_piggyback>
                <reliability>
                    <kind>RELIABLE</kind>
                    <max_blocking_time>
                        <sec>1000</sec>
                    </max_blocking_time>
                </reliability>
            </qos>
            <times>
                <initialHeartbeatDelay>
                    <nanosec>0</nanosec>
                </initialHeartbeatDelay>
                <heartbeatPeriod>
                    <sec>0</sec>
                    <nanosec>{params['retrans_ns']}</nanosec>
                </heartbeatPeriod>
                <nackResponseDelay>
                    <nanosec>0</nanosec>
                </nackResponseDelay>
                <nackSupressionDuration>
                    <sec>0</sec>
                </nackSupressionDuration>
            </times>
        </publisher>
    </profiles>
</dds>
"""
    with open(output_file, "w") as f:
        f.write(xml_content)
    print(f"[INFO] Publisher XML generated: {output_file}")

def generate_subscriber_xml(params, output_file):
    """Generate XML configuration for the subscriber"""
    xml_content = f"""<?xml version="1.0" encoding="UTF-8" ?>
<dds xmlns="http://www.eprosima.com">
   <profiles>
        <transport_descriptors>
            <transport_descriptor>
                <transport_id>udp_transport</transport_id>
                <type>UDPv4</type>
                <maxMessageSize>1472</maxMessageSize>
            </transport_descriptor>
        </transport_descriptors>
        <participant profile_name="optimizer_participant_sub_profile" is_default_profile="true">
            <rtps>
                <userTransports>
                    <transport_id>udp_transport</transport_id>
                </userTransports>
                <useBuiltinTransports>false</useBuiltinTransports>
            </rtps>
        </participant>
        <subscriber profile_name="optimizer_subscriber_profile" is_default_profile="true">
            <topic>
                <resourceLimitsQos>
                    <max_samples>{params['history_cache']}</max_samples>
                    <max_instances>10</max_instances>
                    <max_samples_per_instance>{params['history_cache']}</max_samples_per_instance>
                </resourceLimitsQos>
            </topic>
            <qos>
                <liveliness>
                    <kind>AUTOMATIC</kind>
                    <lease_duration>
                        <sec>DURATION_INFINITY</sec>
                    </lease_duration>
                    <announcement_period>
                        <sec>DURATION_INFINITY</sec>
                    </announcement_period>
                </liveliness>
            </qos>
            <times>
                <initialAcknackDelay>
                    <sec>0</sec>
                    <nanosec>0</nanosec>
                </initialAcknackDelay>
                <heartbeatResponseDelay>
                    <sec>0</sec>
                    <nanosec>0</nanosec>
                </heartbeatResponseDelay>
            </times>
        </subscriber>
    </profiles>
</dds>
"""
    with open(output_file, "w") as f:
        f.write(xml_content)
    print(f"[INFO] Subscriber XML generated: {output_file}")

def main():
    required_keys = {"r", "u", "T", "w"}

    if len(sys.argv) < 2:
        print("Usage: python3 DDS_Optimizer.py r=30 u=330000 T=90000000 w=0.6")
        sys.exit(1)

    args = parse_args(sys.argv[1:])
    if not required_keys.issubset(args.keys()):
        print("Usage: python3 DDS_Optimizer.py r=30 u=330000 T=90000000 w=0.6")
        sys.exit(1)

    qos = compute_parameters(args)
    base_name = f"DDS_Optimizer_r{qos['r']}_u{qos['u']}_T{qos['T']//1_000_000}"
    pub_filename = f"{base_name}_pub.xml"
    sub_filename = f"{base_name}_sub.xml"

    generate_publisher_xml(qos, pub_filename)
    generate_subscriber_xml(qos, sub_filename)

    print(f"[INFO] HistoryCache_size = {qos['history_cache']} | Retransmission_ns = {qos['retrans_ns']}")

if __name__ == "__main__":
    main()

```

---

## 💡 How to run it from the terminal
```bash
python3 DDS_Optimizer.py r={publish rate} u={payload size} T={link-layer throughput} w={link utilization}
```

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

<!--**LEE et al.**--> "Optimizing ROS 2 Communication for Wireless Robotic Systems," IEEE INFOCOM 2026.
