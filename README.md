# DDS Optimizer for Wirelss Large Payload Transfer
<p align="center">
  <img alt="ROS2 logo" src="https://img.shields.io/badge/ROS--2-Humble-blue?style=for-the-badge">
  <img alt="Fast DDS logo" src="https://img.shields.io/badge/Fast--DDS-2.6.9-brightgreen?style=for-the-badge">
</p>


## 📝 Abstract
Wireless transmission of large payloads, such as high-resolution images and LiDAR point clouds, is a major bot tleneck in ROS 2, the leading open-source robotics middleware. The default Data Distribution Service (DDS) communication stack in ROS 2 exhibits significant performance degradation over lossy wireless links. Despite the widespread use of ROS 2, the underlying causes of these wireless communication challenges remain unexplored. In this paper, we present the first in-depth network-layer analysis of ROS 2’s DDS stack under wireless conditions with large payloads. We identify the following three key issues: excessive IP fragmentation, inefficient retransmission timing, and congestive buffer bursts. To address these issues, we propose a lightweight and fully compatible DDS optimization framework that tunes communication parameters based on link and payload characteristics. Our solution can be seamlessly applied through the standard ROS 2 application interface via simple XML-based QoS configuration, requiring no protocol modifications, no additional components, and virtually no in tegration efforts. Extensive experiments across various wireless scenarios demonstrate that our framework successfully delivers large payloads in conditions where existing DDS modes fail, while maintaining low end-to-end latency.


## 📝 Paper Summary
ROS 2 uses a DDS-based communication stack, but suffers from severe performance degradation when transmitting large payloads over wireless networks. The key root causes are identified as IP fragmentation, inefficient retransmission timing, and buffer bursts. To address these issues, the paper proposes a lightweight optimization framework that leverages only XML-based QoS configuration without modifying the DDS protocol. The optimization consists of: (i) setting the RTPS message size to 1472 B, (ii) configuring the retransmission rate as *n* = *2r*, and (iii) adjusting the HistoryCache size based on payload size *u* and link bandwidth. All improvements are fully compatible with existing ROS 2 applications and require no changes to application logic or middleware internals. Experiments were conducted on ROS 2 Humble using Fast DDS 2.6.9 version over IEEE 802.11ac wireless links. Test scenarios included different packet error rates (1%, 20%), temporary link outages, and varying payload sizes (32–512 KB). Default DDS and LARGE_DATA mode failed to maintain performance under high loss, while the proposed optimization remained stable up to 512 KB. The effectiveness of HistoryCache tuning was particularly evident in link outage recovery scenarios. Overall, this work demonstrates that practical, protocol-compliant DDS tuning enables robust and real-time wireless communication in ROS 2.


## About the optimizer
Our forthcoming paper introduces an optimizer for configuring parameters to efficiently transmit large payloads.

> **Input**: Publish rate *r*, payload size *u*, link-layer throughput *T*<sub>OS→Link</sub>, link utilization 𝜔  
> **Step 1**: **Prevents IP fragmentation** Set RTPS maxMessageSize = 1472 B  
> **Step 2**: **Decouples control traffic** Set retransmission rate *n* = *2r*   
> **Step 3**: **Bounds writer history** Set HistoryCache size as <div align="center"> <img width="169" height="56" alt="image" src="https://github.com/user-attachments/assets/e95dfebe-7b2a-4076-bee3-71e8b73491cb" /></div>
> **Output**: Optimized ROS 2 XML QoS profile


## Essential QoS Settings for Topic Communication
| QoS Policy| QoS Value | Function | 
|------|---------|---------|
| **history** | **KEEP_ALL** | To store all data without loss |
| **reliability** | **RELIABLE** | To ensure that every message sample is delivered without loss |
| **reliability.max_blocking_time** | **A sufficiently large value** | To prevent publisher blocking or data loss |


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


## Performance comparison across DDS modes and payload sizes
<div align="center">
  <img src="https://github.com/user-attachments/assets/c5b10170-74ac-4b8a-a0e7-9dc8ac82e742" alt="table2" width="600">
</div>


## Quick Start
```bash
git clone https://github.com/anonymous-ld/large-data-optimization.git
cd large-data-optimization
./scripts/install.sh
```


## How to run it from the terminal
```bash
python3 DDS_Optimizer.py r={publish rate} u={payload size} T={link-layer throughput} w={link utilization}
```

### usage
&nbsp;&nbsp;&nbsp;To generate the XML configuration file, use the following parameters:  
&nbsp;&nbsp;&nbsp;[publish rate = 30 Hz], [payload size = 330 KB], [link-layer throughput = 90 Mb/s], and [link utilization = 0.6 (recommended range: 0.6–0.7)].
```Input
python3 DDS_Optimizer.py r=30 u=330000 T=90000000 w=0.6
```
&nbsp;&nbsp;&nbsp;Then, the optimized XML file will be generated with the following output.
```
[INFO] Publisher XML:DDS_Opimizer_r30_u330000_T90_pub.xml  
[INFO] Subscriber XML:DDS_Opimizer_r30_u330000_T90_sub.xml  
[INFO] HistoryCache_Size = 309 | Retransmission_ns = 16666666
```


## 📢 Notice
This project is currently compatible with ROS 2 Humble using Fast DDS 2.6.9.
Support for other DDS vendors such as Cyclone DDS and OpenDDS is planned in future updates.

