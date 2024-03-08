# EFraS
This work provides a novel framework called EFraS for designing, developing, and evaluating various Virtual Network Embedding (VNE) strategies in a multi-domain SDN environment.


## Execution Environment:

Operation System: Microsoft Windows 10, 64bit.<br />
Physical Memory (RAM) 16.0 GB.<br />


### Prerequisites

Python 3.9.<br />
PyCharm Community Edition 2021.2. <br />
Mininet utility.<br />



## Getting Started
- Install [Mininet](http://mininet.org/download/)

- Install dependency packages </br>
     ```
     $ sudo pip install networkx
     $ sudo apt-get install python3-openpyxl
     ```

- **Instructions to run**:
    - Clone the repo </br>
        ```
        $ git clone https://github.com/keerthankumar22/EVNE.git
        ```
        
    - Run the Mininet code  </br>
        
        The two main executable files are `main.py` and `runner.py`. </br>
          - `main.py`: Used to run one experiment at a time. It reads configurations from the `configurations.json` file. </br>
          - `runner.py`: Used to run multiple experiments (over multiple iterations, for different numbers of VNRs, and for various VNE algorithms) at a time. It internally makes calls to the `main.py` depending on the configurations mentioned in `configurations.json`.
          </br>
        ``` 
        $ sudo python3 vne/runner.py 
        ```
        ``` 
        $ sudo python3 vne/main.py 
        ```
        ``` 
        $ sudo python3 vne/main.py -s 5 -a first-fit-algorithm -n 10
        ```

 - Optional installations: </br>
 If we wish to use the RYU controller (instead of Mininet's default ovs-controller), we must do additional installations.
     - Install [RYU controller](https://ryu.readthedocs.io/en/latest/getting_started.html)
     - Move/copy the ryu controller file `ryu_controller_vne.py` to the location where we installed RYU, into the directory `ryu/app/`
     - Start the ryu controller </br>
        ``` 
        $ ryu-manager ryu/app/ryu_controller_vne.py 
        ```
       we can modify this controller file to leverage RYU's features.

## Overview
The VNE emulator has the following main components:

- **Generate substrate network**: The substrate network is generated using *Mininet*, which provides a virtual test bed and development environment for software-defined networks (SDN). It creates the substrate network following a spine-leaf topology and does the IP address of all substrate hosts in the network. Flow table entries of all OpenFlow switches are also populated, thus establishing a deterministic path between every pair of hosts in the substrate network.

- **Generate VNRs**: Set/pool of VNRs is generated. The CPU and link bandwidth requirement limits can be specified in the configurations file, along with other parameters, such as the number of VNRs to generate. The list of VNRs can also be ranked/ordered before trying to serve/map them onto the substrate network.

- **VNE Algorithm**: Once the substrate network is ready and you have the list of VNRs to serve, the VNE algorithms module loops over the list of VNRs trying to serve/map them one at a time. It currently supports multiple VNE algorithms such as MWF, NORD, VNE-NRM, and ReMatch, and we have made it very easy to plug in and integrate any other algorithm. The VNE algorithm *selects* the substrate resources (i.e., substrate hosts and links) for serving/mapping the given VNR and passes the *selected substrate resources for mapping* to the next VNR mapping module.

- **VNR Mapping**: The actual mapping of VNR onto the substrate network happens here. Internally, this module handles IP addressing of the virtual hosts of VNR, flow table updations to support routing packets to virtual hosts, VLAN for isolation between VNRs, traffic control to restrict the bandwidth of a virtual link mapped onto a substrate link, etc.

- **Tests**: For testing the emulator setup and to test if each VNR is getting the allocated resource, we use network performance tools such as *iperf* for performing bandwidth tests. Reachability tests are performed using *ping*, where every virtual host shall be reachable to every other virtual host within the same VNR but not to any other host. CPU limit tests are also performed here to complete end-to-end testing of the VNE emulator.

</br>
</br>
<img src="https://github.com/geegatomar/Official-VNE-SDN-Major-Project/blob/master/images/emulator_architecture_diagram.png?raw=true" width="85%">
</br>


## Project Modules
The project work is broadly divided into the following parts:
### Substrate topology generation
1. Spine leaf topology creation
2. IP addressing of nodes
3. Flow table entry population of OFSwitch

### Generate VNRs

 **VNE Algorithm**: Once the substrate network is ready
### Mapping VNRs on the substrate network
1. VNE algorithm to *select* which substrate host to map the virtual host onto
2. The actual VNR *mapping* logic; IP addressing of virtual hosts, VLAN isolation, updating flow table entries, etc.

### To integrate your VNE algorithm into our emulator
This section explains how we integrated the [NORD algorithm](https://www.sciencedirect.com/science/article/abs/pii/S1389128623001068) in our emulator. The same set of steps can be followed to integrate any other VNE algorithm.
- The `vne_algorithm()` function in the module selects which vne algorithm function to call based on the algorithm specified in the configuration file. [vne_algorithms.py#L176]
- The `_nord_algorithm()` function is called by the previous function. [vne_algorithms.py#L41]
- A folder called `nord` is added in the code with all the NORD algorithm logic, and we add `nord_support.py` file to handle the conversion of data structures in our code convention to NORD's code convention. The main function in this file is `get_ranked_hosts()`, which returns the ordered list of ranked substrate hosts and ranked virtual hosts (in our code convention). [nord_support.py#L129]
- The `ranked_virtual_hosts` and `ranked_substrate_hosts` are then fed into the `_greedy_vne_embedding()` function, which returns the final set of selected substrate resources for mapping this VNR. [vne_algorithms.py#L63]
- The `vne_algorithm()` function is expected to return `cpu requirements for vnr mapping` and `bandwidth requirement for vnr mapping` in our code's convention, which will further be passed to the next module (VNR mapping) that will perform the actual mapping of VNR on substrate network.
     
### Testing
1. Pings for connectivity within VNR hosts
2. Iperf for bandwidth links
3. CPU limit tests

### Configurations.json provides the flexibility to set the various parameters related  to the substrate network and the VNRs generation. It also controls the algorithms to run and iterations. 
{
    "substrate": {

        "sl_factor": 6,
        "ll_factor": 3,
        "hl_factor": 3,

        "cpu_limit_min": 20,
        "cpu_limit_max": 100,

        "spine_to_leaf_links": {
            "bw_limit_min": 50,
            "bw_limit_max": 100
        },

        "leaf_to_host_links": {
            "bw_limit_min": 50,
            "bw_limit_max": 100
        }
    },

    "vnrs": {
        "num_vnrs": -, 
        "min_nodes": 2, 
        "max_nodes": 10, 
        "probability": 0.4, 
        "min_cpu": 1,
        "max_cpu": 10, 
        "min_bw": 1, 
        "max_bw": 5
    },

    "Xvne_algorithm": "worst-fit-algorithm",
    "Zvne_algorithm": "nord-algorithm",
    "Qvne_algorithm": "nrm-algorithm",
    "vne_algorithm": "ahp-algorithm",


    "iterations": 5,
    "vne_algorithms": ["worst-fit-algorithm", "nord-algorithm", "nrm-algorithm", "ahp-algorithm"],
    "num_vnrs_list": [20, 40, 60, 80]
}

# Sample Expected Results

Shows how many VNRs were successfully mapped onto the substrate network (by using the selected VNE algorithm)

![results](https://github.com/geegatomar/Official-VNE-SDN-Major-Project/blob/master/images/results.png?raw=true)
In this example, mapping was found for 3 out of the 4 VNRs.

### CPU tests passing
![results](https://github.com/geegatomar/Official-VNE-SDN-Major-Project/blob/master/images/results_cpu.png?raw=true)

### Iperf tests passing
<img src="https://github.com/geegatomar/Official-VNE-SDN-Major-Project/blob/master/images/results_iperf.png?raw=true" width="70%">



### Ping tests passing
<img src="https://github.com/geegatomar/Official-VNE-SDN-Major-Project/blob/master/images/results_ping.png?raw=true" width="90%">


### Additional ping tests
To demonstrate that VLANs are able to provide isolation between VNRs, we can run `pingall` between every pair of hosts in the network. 
- As expected, all the substrate hosts are able to reach every other substrate host (h1, h2, h3, ..., h18), and no other virtual hosts. 
- VNR1 has 4 virtual hosts (vnr1_vh1, vnr1_vh2, vnr1_vh3, vnr1_vh4); and as expected, each virtual host of VNR1 is able to reach every other virtual host of VNR1, but not able to reach any other host (i.e. any substrate host or any host of VNR2 or VNR3).
- Similar results can be observed for VNR2 and VNR3. Hence, isolation is achieved using VLANs in our emulator's network.


<img src="https://github.com/geegatomar/Official-VNE-SDN-Major-Project/blob/master/images/vlan_isolation_ping.png?raw=true" width="90%">
('X' denotes that there is no reachability and ping failed for that host pair)

---



### Results.xlsx
The results from multiple experiments run in [runner.py] are captured in this Excel file. The primary columns captured are as follows:
- **seed**: 
- **algorithm**: 
- **revenue**: 
- **total_cost**: 
- **revenuetocostratio**: 
- **accepted**:
- **total_request**: 
- **embedding ratio**:
- **pre_resource**: 
- **post_resource**: 
- **resources consumed**: 
- **No_of_Links_used**: 
- **No_of_Nodes_used**: 
- **total_nodes**: 
- **total_links**: 
- **total_execution_time**: 
- **avg_bandwidth_utilization**: 
- **avg_crb_utilization**: 
- **avg_link_utilization**: 
- **avg_node_utilization**:
 
### Performance Evaulation
The Proposed EFraS framework is evaluated by implementing the following state-of-the-art papers.<br />

Rematch.py -> The Main file related to the  ReMatch baseline approach [1].<br />
Topsis.py -> The Main file related to the  Proposed NORD approach [2].<br />
Greedy.py -> The Main file related to the  MFW a Greedy baseline approach [3].<br />
Nrm.py -> The Main file related to the  VNE-NRM baseline approach [4].<br />

### References
[1]. A. Satpathy, M. N. Sahoo, L. Behera, C. Swain, ReMatch: An Efficient Virtual Data Center Re-Matching Strategy Based on Matching Theory, IEEE Transactions on Services Computing (2022). doi: https://doi.org/10.1109/TSC.2022.3183259.
[2]. TG, K. K., Addya, S. K., Satpathy, A., & Koolagudi, S. G. (2023). NORD: NOde Ranking-based efficient virtual network embedding over single Domain substrate networks. Computer Networks, 225, 109661. doi:  https://doi.org/10.1016/j.comnet.2023.109661
[3]. TG, Keerthan Kumar, et al. "MatchVNE: A Stable Virtual Network Embedding Strategy Based on Matching Theory." 2023 15th International Conference on COMmunication Systems & NETworkS (COMSNETS). IEEE, 2023. doi: https://10.1109/COMSNETS56262.2023.10041377
[4]. P. Zhang, H. Yao, Y. Liu, Virtual network embedding based on computing, network, and storage resource constraints, IEEE Internet of Things Journal 5 (5) (2017) 3298–3304. doi: https://doi.org/10.1109/JIOT.2017.2726120.



