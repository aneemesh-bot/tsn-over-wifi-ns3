# Wi-Fi QoS bridged to TSN topology over ns-3 in C++

Additional modifications: **ported to ns-3.36.1**
Password to ZIP is 1234

# Composition

- TSN-enabled switch and nodes connected to Wi-Fi 6 AP and stations with its own switch.
- The simulation uses PerfectClockModel for time sync
    - Different clocks but synchronized by design
    - An additional goal is to implement Fine Time Measurement (FTM) to obtain timestamps
    
    ![Untitled](https://file.notion.so/f/f/dac91141-a4f3-46d5-b1c8-065ad153880e/37af31d9-dde3-4d43-b805-753a8e045558/Untitled.png?id=e06e1e22-f781-45a8-a8e5-c745546065f1&table=block&spaceId=dac91141-a4f3-46d5-b1c8-065ad153880e&expirationTimestamp=1713254400000&signature=Og88jt3pA0KuRdOAkAp6A60fOinoJF5VXFaLiZ73dNY&downloadName=Untitled.png)
    

# Advantages

- QoS allows setting of priority of transmissions in a multimodal, bidirectional system, and is built into the WiFi model.
    - The specified topology has 3 STAs, 1 AP, 2 switches for the wireless and wired ends, and 3 wired nodes.
    - I implemented applications running on all nodes, transferring data back and forth to simulate traffic.
- QoS allows deterministic transmission for high-priority real-time applications, minimizing the impact of concurrent best-effort traffic.
- Additionally, I installed the time-aware shaper on to the switches, as well as a porting process to ns-3.36 to utilize the new build system and enable newer features of the ns-3 stack to be used.

# Configuration

- Setup process of ns-3.34 (I used **bake**) and ns-3.36.
    - Ubuntu 20.04 containerized instance
        - `sudo apt install g++ python3 python3-dev pkg-config sqlite3 cmake python3-setuptools git qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools gir1.2-goocanvas-2.0 python3-gi python3-gi-cairo python3-pygraphviz gir1.2-gtk-3.0 ipython3 openmpi-bin openmpi-common openmpi-doc libopenmpi-dev autoconf cvs bzr unrar gsl-bin libgsl-dev libgslcblas0 wireshark tcpdump sqlite libxml2 libxml2-dev libc6-dev libc6-dev-i386 libclang-dev llvm-dev automake python3-pip libxml2 libxml2-dev libboost-all-dev`
        - `git clone [https://gitlab.com/nsnam/ns-3-dev.git](https://gitlab.com/nsnam/ns-3-dev.git) && cd ns-3-dev`
        - `git checkout -b ns-3.36.1-branch ns-3.36.1`
        - VS Code running on the same instance to utilize container’s libraries
        - The porting process resolves the non-termination issue in ns-3.34 since 3.36 supports concurrent execution of applications and Flowmon displays statistics after the execution is over. Additionally, CMake allows easier debugging through IDEs.
            - launch.json for debugging ns-3.36
            
            ```
            "version": "0.2.0",
                "configurations": [
                    {
                        "name": "(gdb) Launch from scratch",
                        "type": "cppdbg",
                        "request": "launch",
                        "program": "${workspaceFolder}/build/scratch/TSNWifi/ns3.36.1-${fileBasenameNoExtension}-default",
                        "args": [],
                        "stopAtEntry": false,
                        "cwd": "${workspaceFolder}",
                        "preLaunchTask": "Build",
                        "environment": [
                            {
                                "name": "LD_LIBRARY_PATH",
                                "value": "${workspaceFolder}/build/lib/"
                            }
                        ],
            ```
            
            - Pipe Launching for ns-3.34
            
            ```json
            {
                        "name": "(gdb) Pipe Launch",
                        "type": "cppdbg",
                        "request": "launch",
                        //"program": "${workspaceFolder}/build/utils/ns3-dev-test-runner-debug",
                        "program": "${workspaceFolder}/build/scratch/TSNWifi/${fileBasenameNoExtension}",
                        //"args": ["--nWifi=12"],
                        "stopAtEntry": false,
                        "cwd": "${workspaceFolder}",
                        "environment": [],
                        "externalConsole": false,
                        "pipeTransport": {
                            "debuggerPath": "",  // leave blank
                            "pipeProgram": "${workspaceFolder}/waf",
                            // pipeArgs is essentially the entire waf command line arguments                
                            "pipeArgs": [                    
                                "--command-template", "\"",                 // opening double quote for command template 
                                "${debuggerCommand}",                       // gdb path and --interpreter arg already in debuggerCommand 
                                "--args", "%s",                             // Need to add --args %s to the gdb call
                                "\"",                                       // closing quote for command template
                                "--run", "${fileBasenameNoExtension}",      // Run call with the filename
                                ],
                                "quoteArgs":false,
                                "pipeCwd": ""
                        },
            ```
            

# Implementation

The first step after configuration is enabling QosSupported in *src/tsnwifi/wifi-setup.cc*

![Untitled](Wi-Fi%20QoS%20bridged%20to%20TSN%20topology%20fbee240c0fdf46f3a43604b78b685308/Untitled%201.png)

```cpp
mac.SetType ("ns3::ApWifiMac",
				"EnableBeaconJitter", BooleanValue (false), "QosSupported", BooleanValue (true),
				"Ssid", SsidValue (ssid));
```

Create InetSocketAddress objects for QoS tagging, based on AC Matrix followed in ns-3.34+

```cpp
Ipv4Address apIp = APWifiInterfaces.GetAddress (0);
Ipv4Address staIP1 = staInterfaces.GetAddress (0);
Ipv4Address staIP2 = staInterfaces.GetAddress (1);
Ipv4Address staIP3 = staInterfaces.GetAddress (2);

InetSocketAddress apAddress (apIp, 10);
apAddress.SetTos (0x0);
	
InetSocketAddress sta1Address (staIP1, 10);
sta1Address.SetTos (0x2e);
	
InetSocketAddress sta2Address (staIP2, 10);
sta2Address.SetTos (0x18);
	
InetSocketAddress sta3Address (staIP3, 11);
sta3Address.SetTos (0x14);
```

![Untitled](Wi-Fi%20QoS%20bridged%20to%20TSN%20topology%20fbee240c0fdf46f3a43604b78b685308/Untitled%202.png)

*TOS Matrix, ns-3.34 docs*

- A topology is implemented to enable cross-communication between nodes on the network using UDP to generate traffic.

![Untitled](Wi-Fi%20QoS%20bridged%20to%20TSN%20topology%20fbee240c0fdf46f3a43604b78b685308/Untitled%203.png)

```cpp
	ApplicationContainer serverApp1
	UdpServerHelper server1 (9);
	serverApp1 = server1.Install (wireStaNodeContainer.Get (0));
	serverApp1.Start (Seconds (0.0));
	serverApp1.Stop (Seconds (simulationTime + 1));

	UdpClientHelper client1 (wireInterfaces.GetAddress (0), 9);
	client1.SetAttribute ("MaxPackets", UintegerValue (4294967295u));
	client1.SetAttribute ("Interval", TimeValue (Time ("0.002"))); // packet interval
	client1.SetAttribute ("PacketSize", UintegerValue (200)); // packet size in bytes
	ApplicationContainer clientApp1 = client1.Install (wifiStaNodeContainer.Get (nWifi - 3)); // 0 -> STA1
	clientApp1.Start (Seconds (0.2));
	clientApp1.Stop (Seconds (simulationTime + 1.2));

	ApplicationContainer serverAppSTA2;
	UdpServerHelper server2 (10);
	serverAppSTA2 = server2.Install (wifiStaNodeContainer.Get (nWifi - 2)); // 1 -> STA2
	serverAppSTA2.Start (Seconds (0));
	serverAppSTA2.Stop (Seconds (simulationTime + 1));

	UdpClientHelper clientAPforSTA2 (sta2Address, 10);
	clientAPforSTA2.SetAttribute ("MaxPackets", UintegerValue (4294967295u));
	clientAPforSTA2.SetAttribute ("Interval", TimeValue (Time ("0.002"))); // packet interval
	clientAPforSTA2.SetAttribute ("PacketSize", UintegerValue (100)); // packet size in bytes
	ApplicationContainer clientApp2 = clientAPforSTA2.Install (APNodeContainer.Get(0)); // AP
	clientApp2.Start (Seconds (0.2));
	clientApp2.Stop (Seconds (simulationTime + 1.2));
	
	ApplicationContainer serverAppAP;
	UdpServerHelper server3 (10);
	serverAppAP = server3.Install (APNodeContainer.Get(0));
	serverAppAP.Start (Seconds (0.0));
	serverAppAP.Stop (Seconds (simulationTime + 1));

	UdpClientHelper clientSTA3forAP (apAddress, 10);
	clientSTA3forAP.SetAttribute ("MaxPackets", UintegerValue (4294967295u));
	clientSTA3forAP.SetAttribute ("Interval", TimeValue (Time ("0.002"))); // packet interval
	clientSTA3forAP.SetAttribute ("PacketSize", UintegerValue (100)); // packet size in bytes
	ApplicationContainer clientApp3 = clientSTA3forAP.Install (wifiStaNodeContainer.Get (nWifi - 1));
	clientApp3.Start (Seconds (0.2));
	clientApp3.Stop (Seconds (simulationTime + 1.2)); 

	ApplicationContainer serverAppSTA3;
	UdpServerHelper server4(11);
	serverAppSTA3 = server4.Install (wifiStaNodeContainer.Get (nWifi - 1)); // 2 -> STA3
	serverAppSTA3.Start (Seconds (0.0));
	serverAppSTA3.Stop (Seconds (simulationTime + 1));

	UdpClientHelper clientAPforSTA3 (sta3Address, 11);
	clientAPforSTA3.SetAttribute ("MaxPackets", UintegerValue (4294967295u));
	clientAPforSTA3.SetAttribute ("Interval", TimeValue (Time ("0.002"))); // packet interval
	clientAPforSTA3.SetAttribute ("PacketSize", UintegerValue (100)); // packet size in bytes
	ApplicationContainer clientApp4 = clientAPforSTA3.Install (APNodeContainer.Get(0)); // AP
	clientApp4.Start (Seconds (0.2));
	clientApp4.Stop (Seconds (simulationTime + 1.2));

	UdpClientHelper clientSTA2forAP (apAddress, 10);
	clientSTA2forAP.SetAttribute ("MaxPackets", UintegerValue (4294967295u));
	clientSTA2forAP.SetAttribute ("Interval", TimeValue (Time ("0.002"))); // packet interval
	clientSTA2forAP.SetAttribute ("PacketSize", UintegerValue (100)); // packet size in bytes
	ApplicationContainer clientApp5= clientSTA2forAP.Install (wifiStaNodeContainer.Get (nWifi - 2));
	clientApp5.Start (Seconds (0.2));
	clientApp5.Stop (Seconds (simulationTime + 1.2));

	UdpClientHelper clientwireSTA2forSTA2 (sta2Address, 10);
	clientwireSTA2forSTA2.SetAttribute ("MaxPackets", UintegerValue (4294967295u));
	clientwireSTA2forSTA2.SetAttribute ("Interval", TimeValue (Time ("0.002"))); // packet interval
	clientwireSTA2forSTA2.SetAttribute ("PacketSize", UintegerValue (100)); // packet size in bytes
	ApplicationContainer clientApp6= clientwireSTA2forSTA2.Install (wireStaNodeContainer.Get(1));
	clientApp6.Start (Seconds (0.2));
	clientApp6.Stop (Seconds (simulationTime + 1.2));
	
	ApplicationContainer serverAppSTA1;
	UdpServerHelper server5 (10);
	serverAppSTA1 = server5.Install (wifiStaNodeContainer.Get (nWifi - 3));
	serverAppSTA1.Start (Seconds (0.0));
	serverAppSTA1.Stop (Seconds (simulationTime + 1));

	UdpClientHelper clientwireSTA3forSTA1 (sta1Address, 10);
	clientwireSTA3forSTA1.SetAttribute ("MaxPackets", UintegerValue (4294967295u));
	clientwireSTA3forSTA1.SetAttribute ("Interval", TimeValue (Time ("0.002"))); // packet interval
	clientwireSTA3forSTA1.SetAttribute ("PacketSize", UintegerValue (100)); // packet size in bytes
	ApplicationContainer clientApp7= clientwireSTA3forSTA1.Install (wireStaNodeContainer.Get(2));
	clientApp7.Start (Seconds (0.2));
	clientApp7.Stop (Seconds (simulationTime + 1.2));
```

## Testing

- Flowmon
    - ns3::FlowMonitorHelper
    - ns3::Ipv4HelperClassifier - statistics segregation per flow
    
    ```cpp
    for (std::map<FlowId, FlowMonitor::FlowStats>::const_iterator iter = stats.begin (); iter != stats.end (); ++iter)
    	{
    		Ipv4FlowClassifier::FiveTuple t = classifier->FindFlow (iter->first);
    
    		NS_LOG_UNCOND("\n----Flow ID:" <<iter->first);
    		NS_LOG_UNCOND("Src Addr " <<t.sourceAddress << " Dst Addr "<< t.destinationAddress);
    		NS_LOG_UNCOND("Sent Packets=" <<iter->second.txPackets);
    		NS_LOG_UNCOND("Received Packets =" <<iter->second.rxPackets);
    		//NS_LOG_UNCOND("Lost Packets =" <<iter->second.txPackets-iter->second.rxPackets);
    		NS_LOG_UNCOND("Packet delivery ratio =" <<iter->second.rxPackets*100/iter->second.txPackets << "%");
    		//NS_LOG_UNCOND("Packet loss ratio =" << (iter->second.txPackets-iter->second.rxPackets)*100/iter->second.txPackets << "%");
    		NS_LOG_UNCOND("Packet loss percentage =" << iter->second.txPackets-iter->second.rxPackets << "/" << iter->second.txPackets << "(" << (iter->second.txPackets-iter->second.rxPackets)*100/iter->second.txPackets << "%)");
    		NS_LOG_UNCOND("Avg. Delay =" << NanoSeconds(((iter->second.delaySum).GetInteger())/iter->second.rxPackets));
    		NS_LOG_UNCOND("Sum. Delay =" << iter->second.delaySum);
    		NS_LOG_UNCOND("Avg. Jitter =" << NanoSeconds(((iter->second.jitterSum).GetInteger())/iter->second.rxPackets));
    		NS_LOG_UNCOND("Sum. Jitter =" <<iter->second.jitterSum);
    	}
    ```
    
    [Dataset QoS ns-3 measurements.xlsx](Wi-Fi%20QoS%20bridged%20to%20TSN%20topology%20fbee240c0fdf46f3a43604b78b685308/Dataset_QoS_ns-3_measurements.xlsx)
    
    - For a more accurate picture, a manual measurement using PCAP files is required.
    
    ```cpp
    // in TSNWifi.cc
    csma.EnablePcap ("TSNWifi-wire.pcap", APDevices.Get (0), true);
    csma.EnablePcap ("TSNWifi-wire", wireDevices.Get (0), true);
    csma.EnablePcap ("TSNWifi-Switch1-AP", switchFullLink.Get (1), true);
    csma.EnablePcap ("TSNWifi-Switch1-Node0", switch1Link.Get (0), true);
    csma.EnablePcap ("TSNWifi-Switch1-Switch0", switch1Link.Get (3), true);
    csma.EnablePcap ("TSNWifi-Switch0-Switch1", switchFullLink.Get (0), false);
    
    // in wifi-setup.cc
    phy.EnablePcap ("AccessPoint", m_netDeviceContainerAP.Get (0));
    phy.EnablePcap ("StationBG", m_netDeviceContainerSTA.Get (0)); // STA1 (BG)
    phy.EnablePcap ("StationFG", m_netDeviceContainerSTA.Get (1)); // STA2 (FG)
    ```
    

I am using Wireshark to analyze traffic.
