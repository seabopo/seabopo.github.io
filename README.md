**Building a Low-Cost AKS Test Cluster**
========================================

#### PROJECT GOAL
To build an Azure Kubernetes Service (AKS) test cluster to start studying for some certifications. My Visual 
Studio Subscription comes with a $50 monthly Azure credit, so this is the budget target for my test subscription.

#### CLUSTER REQUIREMENTS
I need to be able to test the following:
  * Both Linux and Windows nodes.
  * Multiple nodes for affinity and anti-affinity testing.
  * Availability zones to test Topology Spread Constraints.
  * NGINX-INGRESS using a load balancer.
  * A public IP.

#### CLUSTER COMPONENTS AND COSTS
The minimum components I'll need for testing:
  * A [Kubernetes Cluster](https://learn.microsoft.com/en-us/azure/aks/free-standard-pricing-tiers) 
    control plane for cluster management. There are three available SKUs: Free, Standard and Premium. 
    Since this is a test cluster I'll use the free SKU. The Free SKU includes all AKS features and supports 
    clusters up to 1,000 nodes, though Microsoft recommends 10 or fewer nodes for a free cluster.
  * A [load balancer](https://learn.microsoft.com/en-us/azure/load-balancer/skus) for ingress/egress. 
    AKS clusters support two different load balancer SKUs: Basic and Standard. The Basic SKU is free, but only 
    supports a single VM Scale set and doesn't support Availability Zones. The Standard SKU supports Availability 
    Zones and multiple node pools (required for Windows support), so this will be the one I use. It will 
    cost approximately $15/month if used 24x7 with low amounts of traffic.
  * A [public IP address](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-addresses) 
    (~$3/month) will be required to test public access.
  * A low-cost VM SKU for nodes, which is covered in the next section.
  
#### SELECTING A LOW-COST VM SKU
Most resources on the internet recommend the 
[B-Series](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes-b-series-burstable) VM SKUs because
they are very budget-friendly, starting around $30/month for both Linux and Windows. However, Microsoft no longer 
allows the use of B-Series VMs as System Nodes for an AKS cluster because of performance issues with the scheduler. 
Since Kubernetes requires that the first node pool be a Linux system node pool, another SKU has to be used. 

The following list contains lower-cost, General Purpose SKUs that support Local (Ephemeral) Storage and Premium 
Storage as of 12/15/2024. Unfortunately the cheapest usable VM Size for the system node pool is now almost 
double the cost of the equivalent B-Series SKUs at around $50 (ARM) or $60 (x64) per month.

| VM Size       |vCPUs| RAM |Disks |IOPS |Local Disk  |Linux / 4hrs | Win / 4hrs | Notes |
|---------------|-----|-----|------|-----|------------|-------------|------------|-------|
|   B2s         |  2  |  4  |  4   |1280 |   8 (SCSI) | $28  / $5   | $33  / $6  | (s)   |
|   D2plds_v5   |  2  |  4  |  4   |3750 |  75 (SCSI) | $49  / $9   | $101 / $17 | (a)   |
|   B2ms        |  2  |  8  |  4   |1920 |  16 (SCSI) | $56  / $10  | $61  / $10 | (s)   |
|***D2alds_v6***|  2  |  4  |  4   |4000 | 110 (NVME) | $61  / $10  | $131 / $22 |       |
|   D2lds_v5    |  2  |  4  |  4   |3750 |  75 (SCSI) | $61  / $10  | $113 / $19 |       |
|***D2as_v4***  |  2  |  8  |  4   |3200 |  16 (SCSI) | $61  / $10  | $113 / $19 |       |
|   D2s_v3      |  2  |  8  |  4   |3200 |  16 (SCSI) | $62  / $10  | $117 / $20 |       |
|***D2ads_v5*** |  2  |  8  |  4   |3750 |  75 (SCSI) | $65  / $11  | $117 / $20 |       |
|   D2ds_v4     |  2  |  8  |  4   |3200 |  75 (SCSI) | $72  / $12  | $124 / $21 |       |
|   D2ds_v5     |  2  |  8  |  4   |3700 |  75 (SCSI) | $72  / $12  | $124 / $21 |       |
|   D4plds_v5   |  4  |  8  |  8   |6400 | 150 (SCSI) | $98  / $16  | $201 / $34 | (a)   |
|   B4ms        |  4  | 16  |  8   |2880 |  32 (SCSI) | $112 / $19  | $122 / $20 | (s)   |
|   D4alds_v6   |  4  |  8  |  8   |7600 | 220 (NVME) | $121 / $20  | $224 / $37 |       |
|   D4lds_v5    |  4  |  8  |  8   |6400 | 150 (NVME) | $122 / $20  | $226 / $38 |       |
|   D4ads_v5    |  4  | 16  |  8   |6400 | 150 (SCSI) | $131 / $22  | $235 / $39 |       |

**Notes**
  * VM Size costs are monthly for 24 hours and 4 hours per day for 30 days respectively.
  * VM Size costs do not include the storage costs, which are $14 and $3 for 24 hours and 4 hours respectively.
  * Local Disk sizes are in GiB.
  * For any VM Size other than the Burstable (B-Series) the Windows license cost doubles the monthly cost.
  * (a) = ARM SKUs, which may be less compatible with system node pools apps (I need to check on this).
  * (s) = SKUs that can't be used for System-Mode Node Pools.
  * [Azure Virtual Machine Naming Conventions](
     https://learn.microsoft.com/en-us/azure/virtual-machines/vm-naming-conventions)  
      a = AMD-based processor  
      b = Block Storage performance  
      d = diskful (that is, a local temp disk is present)  
      i = isolated size  
      l = low memory; a lower amount of memory than the memory intensive size  
      m = memory intensive; the most amount of memory in a particular size  
      p = ARM Cpu  
      t = tiny memory; the smallest amount of memory in a particular size  
      s = Premium Storage capable, including possible use of Ultra SSD   


#### **ESTIMATED CLUSTER COSTS**
I selected the **D2plds_v5** VM Size for the Linux system nodes because it's the cheapest system node with with 
4GB of RAM, annd I *think* I can get away with 4GB of RAM. The D2plds_v5 node was using 38% of it's memory for 
the base node install, so I should be able to fit NGINX and the other system services I need on that node.
I selected the B-Series VM Sizes for the app nodes because the B-Series still works well for non-system services 
and they only cost an extra $5 or $10 per month for the Windows licenses where every other VM Size doubles 
the monthly cost for a Windows OS vs a Linux OS.

| Cluster Type                  | VM Size         |vCPUs| RAM  | Disk  | $/hr | $/day  | 24x7 | 12x7 | 8x7  | 4x7 |
|-------------------------------|-----------------|-----|------|-------|------|--------|------|------|------|-----|
| 1 Sys                         |D2plds_v5        |  2  |  4   |   75  | $.09 |  $2.20 |  $66 |  $33 |  $22 | $11 |
| 1 Sys + 1 Lin App             |D2plds_v5 + B2s  | 2/2 | 4/4  | 75/8  | $.13 |  $3.13 |  $94 |  $47 |  $31 | $16 |
| 1 Sys + 1 Win App             |D2plds_v5 + B2s  | 2/2 | 4/4  | 75/8  | $.14 |  $3.30 |  $99 |  $50 |  $33 | $17 |
| 1 Sys + 1 Lin App + 1 Win App |D2plds_v5 + B2s  | 2/2 | 4/4  | 75/8  | $.18 |  $4.23 | $127 |  $64 |  $42 | $21 |
| 1 Sys + 1 Win App             |D2plds_v5 + B2ms | 2/2 | 4/8  | 75/16 | $.18 |  $4.23 | $127 |  $64 |  $42 | $21 |
| 1 Sys + 1 Win App             |D2ads_v5  + B2ms | 2/2 | 8/8  | 75/16 | $.20 |  $4.77 | $143 |  $72 |  $48 | $24 |
| 1 Sys + 3 Lin App             |D2plds_v5 + B2s  | 2/2 | 4/4  | 75/8  | $.21 |  $5.00 | $150 |  $75 |  $50 | $25 |
| 1 Sys + 2 Lin App + 2 Win App |D2plds_v5 + B2s  | 2/2 | 4/4  | 75/8  | $.26 |  $6.27 | $188 |  $95 |  $63 | $31 |
| 1 Sys + 3 Lin App + 3 Win App |D2plds_v5 + B2s  | 2/2 | 4/4  | 75/8  | $.35 |  $8.30 | $249 | $125 |  $83 | $42 |
| 3 Sys + 3 Lin App + 3 Win App |D2plds_v5 + B2s  | 2/2 | 4/4  | 75/8  | $.48 | $11.57 | $347 | $174 | $116 | $58 |


<style>
    p + ul { margin-block-start: -15px }
    p      { margin-block-start:  20px }
    h4     { margin-block-end:   -16px; font-weight: bold; }
</style>

<!--
P + ul {
    display: block;
    list-style-type: disc;
    margin-block-start: -15px;
    margin-block-end: 1em;
    margin-inline-start: 0px;
    margin-inline-end: 0px;
    padding-inline-start: 40px;
}
-->

