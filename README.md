# dji_hass
Integration to launch DJI drone missions through Home Assistant

DISCLAIMER AND LEGAL IMPLICATIONS OF USING THIS INTEGRATION-
Drone flying is a regulated activity in many jurisdictions. This integration software makes no representation of preventing abuse or unauthorized flight. It is up to the pilot to ensure that the local regulation of the aviation authority (FAA in the US) are respected in terms including but not limited to ceiling, required transponders, weight, flight over people, requrements for night flight, etc. Recreational flight requires line of sight in most jurisdictions - do not let the drone fly unattended - ever. Non-recreational use may require licensing (in the US it requires obtaining the FAA Part 107 license as a pilot of an Unmanned Aerial Vehicle - UAV) 

THIS SOFTWARE IS PROVIDED AS-IS AND THE USE OF IT IMPLIES THE USER UNDERSTANDS THE IMPLICATIONS AND TAKES FULL RESPONSIBILITY TO ENSURE LAWFUL USAGE OF THE DRONE AND FOR ANY DAMAGES OR PENALTIES RESULTING FROM MISUSE OR BUGS.



Version 0.0.1 - 

Conceptual definition of the scenario goal
On a trigger defined by HA (e.g. alarm triggered or person detected while no one is home), send the drone on a predetermined mission, notify of the event and stream the video as an option, persist the video to local  (SAN) and/or cloud storage. Allow HA to control the mission (start/abort) and monitor the status of the drone (airborne/batter level)

Limitations: I can only test this with Mavic 2 Pro and Mavic 2 Zoom. Anyone that wants to chip in with a different drone is welcome.

Core features: 
 1) Create a service able to launch a drone and execute a predetermined mission. 
 2) Create a service able to stream and record video from the flying drone
 3) Create a service able to record snapshots from the flying drone

Fundamental technologies:
 1) Home Assistant in some guise or form, starting scenario: Home Assistant OS
 2) DJI SDK - unclear which one is the way to go to build a server-side component. The only SDKs supporting Mavic Pro 2 are Windows and iOS/Android. This may be a major limitation of the implementation.

Architecture in a nutshell
A server component running somewhere (ideally a HA add-on) talks MQTT and RTSP/RTMP with HA on one side and talks with the drone on the other. It exposes a set of services that HA can call to drive actions and gather status and it handles the communication with the aircraft. 

Sevice APIs:
- GetDroneFlightStatus - {airborne/landed}
- GetDroneBatteryLevel - int {0-100}
- ReturnToBase - void 
- ExecuteMission(MissionID) - void
- GetRTSPStream(port) - H264 stream
- StartRecording {stream}
- StopRecording {stream}
- TakeSnapShot {image}
