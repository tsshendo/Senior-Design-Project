# Senior-Design-Project
This is the repository of the code that my team used for our capstone senior design project.

Senior Design Project Overview:

Situation:
The Internal Combustion Engine (ICE) car on the Pack Motorsports Formula SAE team was facing a persistent charge deficit. Since the alternator is integrated into the engine, it couldn’t be modified or replaced. Additionally, upgrading to a larger battery wasn’t possible due to space constraints. After some investigation, we identified that the off-the-shelf rectifier being used had an efficiency of only about 75–80%. This rectifier was the main bottleneck in the system's electrical performance.

Task:
Our goal was to address this inefficiency by replacing the passive rectifier with a more efficient solution. Commercial high-efficiency rectifiers were available, but they exceeded our limited budget. To work around this, we decided to design a custom three-phase active rectifier, targeting 97% efficiency and a power output of 400W.

Action:
The project was split into two main subsystems, with two team members working on each. My primary responsibilities were in the Power Management and Power Factor Correction (PFC) aspects of the system.

My Contributions:
1. Power Management:
I developed a power budget to analyze and plan the energy consumption of components on our custom PCB.

I selected and sized components (e.g., MOSFETs, capacitors, inductors) to minimize power losses, ensuring we could meet our efficiency target.

2. Power Factor Correction (PFC):
I wrote embedded C code to implement the PFC algorithm on an Infineon TLE9893 microcontroller.

The code was responsible for:

Interfacing with input signals from the alternator.

Measuring voltage and current to detect phase differences.

Generating PWM signals to control MOSFET gate triggering for active rectification.

Technical Breakdown of Power Factor Correction:
Power Factor Correction (PFC) improves system efficiency by ensuring that the current drawn by the rectifier is in phase with the voltage. This minimizes wasted power and reduces harmonic distortion.

To implement this, I used a transform-based control strategy common in motor control and power electronics:

Signal Processing and Control Steps:
Clarke Transform:
Converts the three-phase current signals into two components (α and β) in a stationary orthogonal frame. This reduces the complexity of the system.

Park Transform:
Converts the α-β signals into direct (d) and quadrature (q) components in a rotating reference frame aligned with the voltage vector. This transformation helps in identifying the phase shift between voltage and current.

PI Controllers:
The d and q components were fed into two separate Proportional-Integral (PI) controllers, which calculated the error between desired and actual current values. This error represents the reactive (wasted) and real (useful) power components.

Inverse Park and Clarke Transforms:
The corrected d-q values were converted back to α-β and then to three-phase signals. This step generated the control reference for switching the MOSFETs.

Space Vector PWM (SVPWM):
The final three-phase signals were fed into an SVPWM algorithm, which generated precise PWM signals to drive the gates of the MOSFETs, controlling when and how the AC signals are rectified to DC.

Outcome:
The project culminated in a working prototype of a high-efficiency active rectifier that addressed the vehicle’s charge deficit while remaining cost-effective. Our design provided a practical and scalable solution that significantly improved energy recovery from the alternator without needing physical modifications to the engine or battery system.


![WhatsApp Image 2025-06-24 at 7 47 24 PM](https://github.com/user-attachments/assets/e758d77b-d057-4255-a760-56c023ac2b79)

Here is an image of the final custom PCB the team developed and please refer to the Schematic PDF for the circuit design.
For hardware design explanation, please refer to HardwareOverview.md
![image](https://github.com/user-attachments/assets/14f9cbf0-9c22-4583-813f-bb2594925e17)
