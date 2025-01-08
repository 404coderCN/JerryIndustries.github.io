---
layout: post
title: FPGA Drum Synthesis
image: 
  path: /assets/img/projects/robothand_circuit.jpg
description: >
  We synthesized a 511 x 211 drum on an FPGA!
sitemap: false
hide_last_modified: true
---

# FPGA Drum Synthesis

## Introduction
Drums are one of the most widely used instrument around the world. Different drums produce different sounds: small bongos sound crisp and woody whilst a large bass drum create a low and sonorous sound. Despite the different varieties, the sound of drums can all be described by the propagation of waves in the drum membrane. To describe such wave propagation, we can apply the continuous 2D wave equation to a mesh of nodes. The amplitude of a particular node at the next time step is a function of its current amplitude, amplitude at the previous time step, and the amplitudes of its adjacent four nodes at the current time step.

$$
u_{i,j}^{n+1} = \left[1 + \frac{\eta \Delta t}{2}\right]^{-1} 
\left\{
\rho_{\text{eff}} \left[u_{i+1,j}^n + u_{i-1,j}^n + u_{i,j-1}^n + u_{i,j+1}^n - 4u_{i,j}^n\right] 
+ 2u_{i,j}^n 
- \left[1 - \frac{\eta \Delta t}{2}\right] u_{i,j}^{n-1}
\right\}
$$

$\rho_{eff}$ here is the normalized speed of sound in the medium, and it must be within 0.5 for a stable system. In a real drum, $\rho_{eff}$ is non-linearly related to the tension in the membrane and can be described by the following equation.

$$
\rho_{eff} = \min\Bigl(0.49, \rho_{0} + [u_{i,j} \cdot G_{tension}]^2\Bigr)
$$

Assuming $\frac{\eta\Delta t}{2}$ is small, we can further simplify the equation by approximating the first term using first-order Taylor expansion. The resultant simplified expression below is the one we will implement for this lab.

$$
{\rho_{eff}\Bigr[u_{i+1,j}^n+u_{i-1,j}^n+u_{i,j-1}^n+u_{i,j+1}^n-4u_{i,j}^n\Bigr]+2u_{i,j}^n-\Bigr[1-\frac{\eta\Delta t}{2}\Bigr]u_{i,j}^{n-1}\Bigl\}
$$

With these equations, we are challenged to synthesize the largest drum we can, meaning we have to perform calculation on as many nodes as possible before the 48kHz DAC requires a new sample. In this lab, we implemented a pure hardware-based solution to output the sound of drum from FPGA and used simple switches to demonstrate sound of different drum size. We were able to implement a drum with 211 columns and 511 rows.

## Design & Testing
### Overall design
Overall structure for this lab is simple which is shown in figure below. The aim is to design a drum module within FPGA to compute amplitude all nodes in drum(using 50MHz clock) and update center amplitude to audio system when there is space in FIFO(using 48kHz clock). The essence of this lab is that clock in FPGA is much faster than sampling frequency of audio system, so that we can explore how to design a drum module that can compute as many nodes as possible while not exceeding sampling period of audio system(1/48kHz). This is the timing limit for this lab, if this limit is exceeded, audio system will no longer functional. To efficiently compute amplitudes for as many nodes as possible, we designed standard unit node module and generate as many as possible. This leads to another limit in this lab which is resource limit in FPGA. When all available resources are consumed(DSP, Memory, register etc.), we are no longer able to increase number of unit nodes. Number of these nodes are denoted as column number of drum. In this case, these unit nodes can continuously compute amplitude in each cycle which expand the size of the drum. We denote the number of cycles these unit nodes can compute as the row number of drum. 

<img src="{{ "/assets/img/projects/overall_design.png" | prepend: site.baseurl | prepend: site.url}}" alt="Overall design" />

### Drum Module Design
Basic drum module in FPGA is shown below. We used start signal to indicate when to start computation and row_num to indicate maximum computational cycle(row number of drum). center_out signal is used to pass center amplitude to FIFO in audio system. Internal structure is divided in to data path and control path as usual. In data path, as introduced above, we instantiated multiple unit nodes to process computation in parallel. Since we need to transmit center amplitude to FIFO, center node needs extra logic to provide amplitude. Resting nodes were connected in serial on both sides of center node. Since all nodes can almost be regarded as identical, same control signals can be used in all of them. More detailed design is given in next section.

<img src="{{ "/assets/img/projects/drum_design.jpg" | prepend: site.baseurl | prepend: site.url}}" alt="Drum design" />

### Unit Node Design
Design structure of unit nodes is shown below. Common Unit Node indicates nodes on sides of center node. Two major parts are included: a three-stage pipeline flow of amplitude data and a combinational wave equation solver. Two M10K blocks, denoted as current M10K and previous M10K, are instantiated to store previous($u^{n-1}$) and current ($u^{n}$) amplitude data for corresponding column. Current M10K block is connected directly to the pipeline flow. Under control of control path, M10K block will produce data from first row to last continuously. In this case, three stages in this pipeline flow correspond to top($u_{i,j+1}^n$), current($u_{i,j}^n$) and bottom($u_{i,j-1}^n$) in the wave equation. To be more specific, every time a new data is produced by current M10K block, bottom will be switched to current, current be switched to top and top be updated by the new data. Two muxes were designed in top and bottom stage driven by is_top and is_bottom signals to introduce zero to bottom stage in first computation and top stage in last computation. Current amplitude data is connected to previous M10K block as write data because, once new data is produced, current amplitude becomes previous amplitude. Current amplitude is also configured as output to feed to adjacent unit nodes as left and right. As mentioned above, wave equation solver is a combinational implementation using current($u_{i,j}^n$), top($u_{i,j+1}^n$), bottom($u_{i,j-1}^n$), left($u_{i-1,j}^n$), right($u_{i+1,j}^n$), previous($u_{i,j}^{n-1}$). For M10K block initialization, we introduced two muxes to select the either initial data or computed data to write.

<img src="{{ "/assets/img/projects/unit_node_design.jpg" | prepend: site.baseurl | prepend: site.url}}" alt="Unit node design" />

As shown below, the internal structure of center node unit is the same as common unit except center output logic to provide center data to audio system which is specified in dash block in the drum design figure. In each sampling point of audio system, center_out should provide next center amplitude of the whole drum. This logic block consists of one register with enable signal and a mux. Register with enable signal is used to hold the next center data until being sampled. Mux is used to output predefined initial center amplitude of center when reset is HIGH. This predefined initial center is the summit of pyramid initialization values for each column and is denoted as init_summit.

### Control Path Design
To achieve maximum row number of drum size, we designed a compact FSM in control path as shown in the figure below. The most significant characteristic of this FSM is that each computation of one row of the drum costs only one single cycle. We used KEY0 as reset button. Once reset is pressed, memory initialization is implemented in MEM_INIT state. In this state, is_mem_init signal shown in drum design figure will be toggled so that predefined hard-coded initial data can be written into M10K block. The initial amplitude for the whole drum is pyramid using center node as summit. Starting from summit to edge, the amplitude of nodes will slowly ramp down towards 0. To prevent any unexpected behavior, we set initial amplitude for current M10K and previous M10K as the same. Internal counter will set control signal is_top HIGH when maximum row number is reached and extract the data from center node based on 48kHz clock in audio system. Then FSM will transfer to IDLE state and be waiting for start signal. Start signal is defined in audio system, which will be HIGH when there is space in FIFO. If so, consecutive two states will be WAIT_MEM and WAIT state. Then in RUN state, each drum row will be computed and updated until whole drum computation finished(top row reached). It is worth noting that there are two wait states between IDLE and RUN states while we only used one WAIT state in ModelSim simulation because we expected there is only one read latency of M10K block as shown in the M10K timing figure below. However, while instantiating M10K block using embedded IP in Quartus, we verified that although in configuration states there is only one latency, real data can only be read one more cycle later using SignalTap logic analyzer. Based on that, MEM_WAIT state was added so that system can wait until the first data reach current state in pipeline and start operating. When all nodes are computed, system goes back to IDLE state and repeat the process above.

<img src="{{ "/assets/img/projects/m10k_timing.png" | prepend: site.baseurl | prepend: site.url}}" alt="M10K timing" />

<img src="{{ "/assets/img/projects/control_path.png" | prepend: site.baseurl | prepend: site.url}}" alt="Control path" />

## Result & Analysis
### ModelSim Simulation
Simulation results from ModelSim is shown below for 33x33 drum size. This plot also shows functionality of communication between audio system that if state is LOW, output of center node will remain the same until start is HIGH. Result shows that center amplitude will oscillate with decreasing value until stable. It can be observed that with larger size of drum, the frequency of oscillation will decrease. Due to the large amount of nodes, a lot of time will be consumed to simulate, hence, we did not provide larger drum simulation here.

<img src="{{ "/assets/img/projects/ms_sim.PNG" | prepend: site.baseurl | prepend: site.url}}" alt="ModelSim result" />

### Time Evaluation
Because our implementation is capable of parallel calculation and the FPGA implementation uses dedicated hardware for numerical computation, the FPGA solution is much faster than the HPS solution. For HPS implementation, computation time increases linearly with the number of nodes, because all calculations are done serially. When running the HPS solution, we are able to synthesize bigger drums by using a bigger cache. This difference can be seen by comparing the "break" point when using -O3 and -O0 for running the HPS script. When we use the -O3 command, the HPS solution is able to finish computation for a 50x50 drum within the time limit, and we are able to hear clear audio outputs. Whereas the -O0 command only enable realtime computation of a 20x20 drum. The increase in cache size enables bigger drum synthesis because the time needed to retrieve data from cache is much faster than from memory, so previous time step amplitude, top, and bottom node amplitudes become available faster as compared to smaller cache size implementations. Thus, more nodes can be computed within the time limit. The computation time for our FPGA solution, on the other hand, increase linearly with the number of rows, because all columns are computed in parallel and we step through rows each cycle. The computational time for the whole drum was evaluated as row_number+3/(50MHz), and the computation time of our biggest drum is about half the time constrain of 20.8us. The three extra cycles is a result of extra states to read from the M10K memory. Our timing evaluation indicates time is not the limit for this design. Although we still have half the time available, we are unable to increase row number further because we already consumed almost all M10K blocks on the FPGA, as shown in the resource utilization table. The drum computation time table and compute time graph give a more intuitive comparison of the computation time of our FPGA solution for different drum sizes with maximum fixed 211 columns. We can see from the figure that computation time is linear with respect to number of rows.

| Size      | Computation Time (μs) |
|-----------|------------------------|
| 211x5     | 0.16                  |
| 211x9     | 0.24                  |
| 211x17    | 0.40                  |
| 211x33    | 0.72                  |
| 211x65    | 1.36                  |
| 211x129   | 2.64                  |
| 211x257   | 5.20                  |
| 211x511   | 10.22                 |

**Table: Computation time for different sized drums**

<img src="{{ "/assets/img/projects/comp_time.png" | prepend: site.baseurl | prepend: site.url}}" alt="Computation time graph" />

### Area & Resource Evaluation
Chip Planner in Quartus provides a good way to evaluate the area and resource utilization of the FPGA as shown in figure below. It can be observed that most of resource is used by our 511x211 drum design. A more detailed resource report is given by compilation report shown in the resource utilization table. Report shows that we used almost all logic resource, pins and memory bits and all DSP blocks. It is interesting to point out that theoretically 174 columns is regarded as limit for instantiating unit nodes, however, unit nodes are able to be generated up to 211 in our design. We deduced that compiler will automatically use logic resource to build more multipliers when there is no available DSP.

<img src="{{ "/assets/img/projects/chip_plannar.PNG" | prepend: site.baseurl | prepend: site.url}}" alt="Chip plannar" />

### Power Spectrum
To evaluate the frequency change of sound due to size change of drum and report in a more quantitative way, we connected the audio system output to oscilloscope to provide power spectrum using FFT with maximum column number with different rows as shown below. It can be observed that, with increasing row number, the largest peak frequency decreases which indicates the change of sounds.

<img src="{{ "/assets/img/projects/9r_211c.png" | prepend: site.baseurl | prepend: site.url}}" alt="Power spectrum" />

<img src="{{ "/assets/img/projects/17r_211c.png" | prepend: site.baseurl | prepend: site.url}}" alt="Power spectrum" />

<img src="{{ "/assets/img/projects/33r_211c.png" | prepend: site.baseurl | prepend: site.url}}" alt="Power spectrum" />

<img src="{{ "/assets/img/projects/65r_211c.png" | prepend: site.baseurl | prepend: site.url}}" alt="Power spectrum" />

<img src="{{ "/assets/img/projects/129r_211c.png" | prepend: site.baseurl | prepend: site.url}}" alt="Power spectrum" />

<img src="{{ "/assets/img/projects/257r_211c.png" | prepend: site.baseurl | prepend: site.url}}" alt="Power spectrum" />

<img src="{{ "/assets/img/projects/511r_211c.png" | prepend: site.baseurl | prepend: site.url}}" alt="Power spectrum" />

The theoretical frequency of the drum mode is labeled as red lines figure below. The theoretical frequencies are multiples of the frequency of the first power peak. The three visible peaks on our power spectrum is hit by the theoretical values, however, some of the theoretical frequencies of the drum mode does not seem to be highlighted in the actual power spectrum.

<img src="{{ "/assets/img/projects/powerspec.JPG" | prepend: site.baseurl | prepend: site.url}}" alt="Power spectrum final" />