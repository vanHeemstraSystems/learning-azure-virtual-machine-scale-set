# 300 - Azure Virtual Machine Scale Set Reverse Engineering

I’ll create a comprehensive reverse engineering document for Azure Virtual Machine Scale Sets, following the same structure and depth as your NSG document.​​​​​​​​​​​​​​​​

I’ve created a comprehensive reverse engineering document for Azure Virtual Machine Scale Sets! It follows the same detailed structure as your NSG document.

[View your VMSS deep-dive document](computer:///mnt/user-data/outputs/Azure_VMSS_Reverse_Engineering.md)

## What’s covered (33 pages):

**Layer 1-3: User Experience & Logic**

- Portal interface and what you see
- Complete JSON structure breakdown (VM profile, instances, autoscale rules)
- How orchestration works (scale-out, scale-in, updates, auto-repair)

**Layer 4-5: Technology & Architecture**

- Azure Fabric Controller and placement algorithms
- Overprovisioning logic and why it exists
- Compute/virtualization stack mapping
- Instance state machine and extension deployment

**Layer 6-7: Deep Implementation**

- Minute-by-minute timeline of instance creation (~3 min process)
- Rolling update flow with health checks
- Automatic repair detection and execution
- Physical server architecture and resource allocation
- Fault domain distribution across datacenter racks

**Layer 8: Real-World Examples**

- Basic web server VMSS with nginx
- CPU-based autoscaling (70% scale-out, 30% scale-in)
- Scheduled scaling (business hours vs. nights/weekends)
- Multi-region with Traffic Manager failover
- Blue-green deployment strategy for zero-downtime updates

The document explains how clicking “scale to 5 instances” triggers a cascade from Azure Resource Manager → Fabric Controller → physical server allocation → Hyper-V VM creation → health monitoring → load balancer integration.

Perfect preparation material for discussing Azure compute scalability in your Team Rockstars assessment! Would you like me to create similar deep-dives for Azure Load Balancer or Application Gateway next?​​​​​​​​​​​​​​​​

# Reverse Engineering Azure Virtual Machine Scale Sets (VMSS)

## From Azure Portal to Physical Servers - A Complete Breakdown

A comprehensive deconstruction of how Azure VMSS works, from what you click in the portal down to the physical allocation of compute resources.

-----

## Table of Contents

- [Layer 1: What You See (Azure Portal)](#layer-1-what-you-see-azure-portal)
- [Layer 2: VMSS Components & Structure](#layer-2-vmss-components--structure)
- [Layer 3: How VMSS Works - Orchestration](#layer-3-how-vmss-works---orchestration)
- [Layer 4: Under the Hood - The Technology](#layer-4-under-the-hood---the-technology)
- [Layer 5: Mapping to Compute & Virtualization Stack](#layer-5-mapping-to-compute--virtualization-stack)
- [Layer 6: Scaling Flow - Instance Lifecycle](#layer-6-scaling-flow---instance-lifecycle)
- [Layer 7: Physical Implementation](#layer-7-physical-implementation)
- [Layer 8: Practical Examples](#layer-8-practical-examples)

-----

## Layer 1: What You See (Azure Portal)

### The User Interface View

When you navigate to a VMSS in the Azure Portal, here’s what you see:

```
Azure Portal → Resource Groups → Your VMSS
├── Overview
│   ├── Instances (current count)
│   ├── Instance view (grid of VMs)
│   └── Metrics (CPU, network, etc.)
├── Instances
│   ├── List of running VMs
│   ├── Instance health status
│   └── Actions (reimage, delete, etc.)
├── Scaling
│   ├── Manual scale
│   ├── Custom autoscale
│   └── Scaling history
├── Availability + scaling
│   ├── Upgrade policy
│   ├── Health monitoring
│   └── Automatic repairs
├── Networking
├── Disks
├── Extensions
├── Properties
└── Locks
```

### Creating a VMSS: The Simple View

**Portal Steps**:

1. Click “Create a resource”
1. Search “Virtual Machine Scale Set”
1. Fill in basics:
- Name: `MyWebVMSS`
- Region: `West Europe`
- Availability zones: `Zone 1, 2, 3`
- Orchestration mode: `Uniform`
- Instance count: `3`
1. Select VM size: `Standard_B2s`
1. Configure image: `Ubuntu 20.04 LTS`
1. Set scaling policy: `Manual` or `Autoscale`
1. Configure networking
1. Click “Create”

**What Just Happened?**

- Azure created an orchestration controller
- Deployed specified number of identical VMs
- Configured load balancer integration
- Set up health monitoring
- Enabled auto-scaling capability
- Created managed identities for instances

### The Abstraction

**What Azure Hides From You**:

```
Simple Portal View:
┌────────────────────────────┐
│ Virtual Machine Scale Set  │
│ - Name: MyWebVMSS          │
│ - Instances: 3             │
│ - Status: Running          │
└────────────────────────────┘

What's Actually Running:
┌────────────────────────────────────────────┐
│ Distributed Compute Orchestration System   │
│ - Fabric controller coordination           │
│ - Multiple physical servers                │
│ - Load balancer integration                │
│ - Health probe monitoring                  │
│ - Automatic instance management            │
│ - Scaling engine evaluation                │
│ - Instance placement optimization          │
│ - Network configuration per instance       │
│ - Storage allocation and management        │
│ - Extension deployment and management      │
└────────────────────────────────────────────┘
```

**The Magic**:

- You say “3 instances” → Azure finds 3 physical servers, allocates resources, deploys VMs
- You say “scale to 10” → Azure finds 7 more servers, deploys 7 more VMs
- Instance fails → Azure detects, deletes, recreates automatically
- Update image → Azure orchestrates rolling update across all instances

-----

## Layer 2: VMSS Components & Structure

### Anatomy of a VMSS

A VMSS consists of several interconnected components:

#### 1. VMSS Resource Itself

```json
{
  "name": "MyWebVMSS",
  "type": "Microsoft.Compute/virtualMachineScaleSets",
  "location": "westeurope",
  "sku": {
    "name": "Standard_B2s",
    "tier": "Standard",
    "capacity": 3
  },
  "properties": {
    "orchestrationMode": "Uniform",
    "singlePlacementGroup": true,
    "upgradePolicy": {
      "mode": "Automatic"
    },
    "virtualMachineProfile": {
      "storageProfile": {},
      "osProfile": {},
      "networkProfile": {},
      "extensionProfile": {}
    },
    "overprovision": true,
    "doNotRunExtensionsOnOverprovisionedVMs": false
  },
  "zones": ["1", "2", "3"]
}
```

**Key Properties Breakdown**:

|Property            |Purpose               |Values                    |Impact                                           |
|--------------------|----------------------|--------------------------|-------------------------------------------------|
|orchestrationMode   |How VMs are managed   |Uniform, Flexible         |Uniform = identical VMs, Flexible = heterogeneous|
|capacity            |Current instance count|0-1000                    |Number of running VMs                            |
|upgradePolicy       |How updates happen    |Automatic, Manual, Rolling|Update orchestration strategy                    |
|overprovision       |Deploy extra instances|true/false                |Creates +20% instances, keeps fastest            |
|singlePlacementGroup|Placement constraint  |true/false                |true = max 100 VMs, false = up to 1000           |
|zones               |Availability zones    |[“1”,“2”,“3”]             |Physical datacenter distribution                 |

#### 2. Virtual Machine Profile

This is the “template” for all instances:

```json
{
  "virtualMachineProfile": {
    "storageProfile": {
      "imageReference": {
        "publisher": "Canonical",
        "offer": "0001-com-ubuntu-server-focal",
        "sku": "20_04-lts-gen2",
        "version": "latest"
      },
      "osDisk": {
        "createOption": "FromImage",
        "caching": "ReadWrite",
        "managedDisk": {
          "storageAccountType": "Premium_LRS"
        },
        "diskSizeGB": 30
      },
      "dataDisks": []
    },
    "osProfile": {
      "computerNamePrefix": "myweb",
      "adminUsername": "azureuser",
      "linuxConfiguration": {
        "disablePasswordAuthentication": true,
        "ssh": {
          "publicKeys": [
            {
              "path": "/home/azureuser/.ssh/authorized_keys",
              "keyData": "ssh-rsa AAAA..."
            }
          ]
        }
      },
      "customData": "base64encodedscript"
    },
    "networkProfile": {
      "networkInterfaceConfigurations": [
        {
          "name": "mynic",
          "properties": {
            "primary": true,
            "ipConfigurations": [
              {
                "name": "myipconfig",
                "properties": {
                  "subnet": {
                    "id": "/subscriptions/.../subnets/default"
                  },
                  "loadBalancerBackendAddressPools": [
                    {
                      "id": "/subscriptions/.../backendAddressPools/myBackendPool"
                    }
                  ]
                }
              }
            ]
          }
        }
      ]
    },
    "extensionProfile": {
      "extensions": [
        {
          "name": "CustomScript",
          "properties": {
            "publisher": "Microsoft.Azure.Extensions",
            "type": "CustomScript",
            "typeHandlerVersion": "2.1",
            "autoUpgradeMinorVersion": true,
            "settings": {
              "fileUris": ["https://storage.blob.core.windows.net/scripts/install.sh"],
              "commandToExecute": "bash install.sh"
            }
          }
        }
      ]
    }
  }
}
```

**Profile Components**:

1. **Storage Profile**: What OS, disk configuration
1. **OS Profile**: Computer name, admin account, SSH keys, cloud-init
1. **Network Profile**: NIC configuration, subnet, load balancer
1. **Extension Profile**: Post-deployment scripts/agents

**Critical Concept**: This profile is the “cookie cutter” - every instance created is identical (in Uniform mode).

#### 3. Instance View

Each running VM is an “instance”:

```json
{
  "instanceId": "0",
  "name": "MyWebVMSS_0",
  "properties": {
    "latestModelApplied": true,
    "vmId": "abc-123-def-456",
    "hardwareProfile": {
      "vmSize": "Standard_B2s"
    },
    "storageProfile": {
      "imageReference": {},
      "osDisk": {
        "name": "MyWebVMSS_0_OsDisk_1_abc123",
        "managedDisk": {
          "id": "/subscriptions/.../disks/MyWebVMSS_0_OsDisk_1_abc123"
        }
      }
    },
    "osProfile": {
      "computerName": "myweb000000",
      "adminUsername": "azureuser"
    },
    "networkProfile": {
      "networkInterfaces": [
        {
          "id": "/subscriptions/.../networkInterfaces/mywebnic0"
        }
      ]
    },
    "provisioningState": "Succeeded",
    "instanceView": {
      "platformUpdateDomain": 0,
      "platformFaultDomain": 0,
      "statuses": [
        {
          "code": "ProvisioningState/succeeded",
          "level": "Info",
          "displayStatus": "Provisioning succeeded",
          "time": "2025-11-08T10:30:00.0000000+00:00"
        },
        {
          "code": "PowerState/running",
          "level": "Info",
          "displayStatus": "VM running"
        }
      ]
    }
  },
  "location": "westeurope",
  "zones": ["1"]
}
```

**Key Instance Properties**:

- **instanceId**: Unique identifier within scale set (0, 1, 2, …)
- **name**: Generated name (prefix + instanceId)
- **latestModelApplied**: Is this instance running the latest VMSS model?
- **vmId**: Azure’s internal unique VM identifier
- **platformUpdateDomain**: For planned maintenance (0-19)
- **platformFaultDomain**: Physical separation (0-2 typically)
- **zones**: Which availability zone (1, 2, or 3)
- **provisioningState**: Lifecycle state
- **instanceView**: Real-time status

#### 4. Scaling Configuration

**Manual Scaling**:

```json
{
  "sku": {
    "capacity": 5
  }
}
```

**Autoscale Settings** (separate resource):

```json
{
  "type": "Microsoft.Insights/autoscaleSettings",
  "name": "myAutoscaleSettings",
  "properties": {
    "enabled": true,
    "targetResourceUri": "/subscriptions/.../virtualMachineScaleSets/MyWebVMSS",
    "profiles": [
      {
        "name": "Default scaling profile",
        "capacity": {
          "minimum": "2",
          "maximum": "10",
          "default": "3"
        },
        "rules": [
          {
            "metricTrigger": {
              "metricName": "Percentage CPU",
              "metricResourceUri": "/subscriptions/.../virtualMachineScaleSets/MyWebVMSS",
              "timeGrain": "PT1M",
              "statistic": "Average",
              "timeWindow": "PT5M",
              "timeAggregation": "Average",
              "operator": "GreaterThan",
              "threshold": 75
            },
            "scaleAction": {
              "direction": "Increase",
              "type": "ChangeCount",
              "value": "1",
              "cooldown": "PT5M"
            }
          },
          {
            "metricTrigger": {
              "metricName": "Percentage CPU",
              "metricResourceUri": "/subscriptions/.../virtualMachineScaleSets/MyWebVMSS",
              "timeGrain": "PT1M",
              "statistic": "Average",
              "timeWindow": "PT5M",
              "timeAggregation": "Average",
              "operator": "LessThan",
              "threshold": 25
            },
            "scaleAction": {
              "direction": "Decrease",
              "type": "ChangeCount",
              "value": "1",
              "cooldown": "PT5M"
            }
          }
        ]
      }
    ],
    "notifications": [
      {
        "operation": "Scale",
        "email": {
          "sendToSubscriptionAdministrator": false,
          "sendToSubscriptionCoAdministrators": false,
          "customEmails": ["admin@example.com"]
        }
      }
    ]
  }
}
```

**Autoscale Rule Breakdown**:

- **metricName**: What to measure (CPU, Memory, Network, Custom metrics)
- **timeGrain**: Metric collection interval
- **timeWindow**: How much history to consider
- **timeAggregation**: How to aggregate (Average, Min, Max, Sum)
- **operator**: Comparison operator
- **threshold**: The trigger value
- **direction**: Increase or Decrease
- **type**: ChangeCount (add/remove N instances) or PercentChangeCount
- **value**: How many instances to add/remove
- **cooldown**: Wait time before next scaling action

#### 5. Upgrade Policy

Three modes for updating instances:

**Automatic**:

```json
{
  "upgradePolicy": {
    "mode": "Automatic",
    "automaticOSUpgradePolicy": {
      "enableAutomaticOSUpgrade": true,
      "disableAutomaticRollback": false
    }
  }
}
```

Behavior: Azure automatically updates all instances when VMSS model changes.

**Rolling**:

```json
{
  "upgradePolicy": {
    "mode": "Rolling",
    "rollingUpgradePolicy": {
      "maxBatchInstancePercent": 20,
      "maxUnhealthyInstancePercent": 20,
      "maxUnhealthyUpgradedInstancePercent": 20,
      "pauseTimeBetweenBatches": "PT2S",
      "enableCrossZoneUpgrade": true,
      "prioritizeUnhealthyInstances": true
    }
  }
}
```

Behavior: Updates instances in batches (e.g., 20% at a time), with health checks.

**Manual**:

```json
{
  "upgradePolicy": {
    "mode": "Manual"
  }
}
```

Behavior: You must manually upgrade each instance.

#### 6. Health Monitoring

**Application Health Extension**:

```json
{
  "extensionProfile": {
    "extensions": [
      {
        "name": "HealthExtension",
        "properties": {
          "publisher": "Microsoft.ManagedServices",
          "type": "ApplicationHealthLinux",
          "typeHandlerVersion": "1.0",
          "autoUpgradeMinorVersion": false,
          "settings": {
            "protocol": "http",
            "port": 80,
            "requestPath": "/health"
          }
        }
      }
    ]
  }
}
```

**Load Balancer Health Probe**:

```json
{
  "probe": {
    "protocol": "Http",
    "port": 80,
    "requestPath": "/health",
    "intervalInSeconds": 15,
    "numberOfProbes": 2
  }
}
```

**Automatic Repairs**:

```json
{
  "automaticRepairsPolicy": {
    "enabled": true,
    "gracePeriod": "PT30M",
    "repairAction": "Replace"
  }
}
```

Behavior: If instance fails health check → wait grace period → repair action (Replace = delete and recreate).

#### 7. Instance Protection

**Protection Policies**:

```json
{
  "protectionPolicy": {
    "protectFromScaleIn": true,
    "protectFromScaleSetActions": false
  }
}
```

- **protectFromScaleIn**: Instance won’t be deleted during scale-in
- **protectFromScaleSetActions**: Instance won’t be affected by VMSS operations (updates, etc.)

#### 8. Orchestration Modes

**Uniform Mode** (Traditional):

```json
{
  "orchestrationMode": "Uniform"
}
```

Characteristics:

- All instances identical
- Created from single VM profile
- Managed as a group
- Max 1000 instances
- Best for stateless workloads

**Flexible Mode**:

```json
{
  "orchestrationMode": "Flexible",
  "platformFaultDomainCount": 1
}
```

Characteristics:

- Instances can be different
- Can mix VM sizes
- Greater control over individual VMs
- Can attach existing VMs
- Better for stateful workloads

-----

## Layer 3: How VMSS Works - Orchestration

### The Orchestration Engine

VMSS has an orchestration controller that continuously evaluates and acts:

```
Orchestration Loop (continuous):

1. Evaluate Desired State
   - What capacity is configured?
   - What's the VM profile?
   - What's the upgrade policy?

2. Evaluate Current State
   - How many instances running?
   - What's the health status?
   - Are they running latest model?

3. Calculate Delta
   - Need more instances? → Scale out
   - Too many instances? → Scale in
   - Instances unhealthy? → Repair
   - Model changed? → Upgrade

4. Execute Actions
   - Create/delete instances
   - Update instances
   - Repair instances

5. Wait (typically 30-60 seconds)

6. Repeat
```

### Scaling Out (Adding Instances)

**Trigger**: Capacity increased (manual or autoscale)

```
Scenario: Scale from 3 to 5 instances

Step 1: Orchestrator detects capacity change
Current: 3 instances
Desired: 5 instances
Delta: +2 instances needed

Step 2: Determine placement
Query Azure Fabric Controller:
- Which availability zones? (Round-robin: zones 1, 2)
- Which fault domains? (Spread across FDs)
- Which physical servers have capacity?

Step 3: Allocate resources
Fabric Controller response:
- Instance 3 → Server in Zone 1, FD 0
- Instance 4 → Server in Zone 2, FD 1

Step 4: Create instances
For each new instance:
  a. Allocate VM ID
  b. Reserve compute (CPU/RAM)
  c. Create managed disk from image
  d. Create network interface
  e. Configure in load balancer backend pool
  f. Apply NSG rules
  g. Boot VM
  h. Run cloud-init/custom-data
  i. Deploy extensions
  j. Wait for health signal

Step 5: Overprovisioning (if enabled)
Actually creates 3 instances (20% extra):
- Instances 3, 4, 5 all start simultaneously
- First 2 to become healthy are kept
- Slowest one is deleted
Why? Ensures fast scale-out and consistent performance

Step 6: Instance ready
- Health check passes
- Added to load balancer rotation
- Capacity now: 5 instances
```

**Under the Hood**:

```python
def scale_out(vmss, new_capacity):
    current_instances = vmss.get_instances()
    delta = new_capacity - len(current_instances)
    
    # Calculate overprovision
    if vmss.overprovision:
        instances_to_create = delta * 1.2  # 20% extra
    else:
        instances_to_create = delta
    
    # Get placement constraints
    zones = vmss.zones
    fault_domains = vmss.platformFaultDomainCount
    
    # Create instances
    created_instances = []
    for i in range(instances_to_create):
        # Round-robin zone assignment
        zone = zones[i % len(zones)]
        
        # Request allocation from Fabric
        allocation = fabric_controller.allocate_vm(
            size=vmss.sku.name,
            zone=zone,
            fault_domain_spread=True
        )
        
        # Create instance
        instance = create_instance(
            vmss_id=vmss.id,
            instance_id=get_next_instance_id(),
            vm_profile=vmss.virtualMachineProfile,
            allocation=allocation
        )
        
        created_instances.append(instance)
    
    # Wait for health signals
    healthy_instances = wait_for_health(
        instances=created_instances,
        timeout=vmss.health_probe_grace_period
    )
    
    # If overprovisioned, keep only needed instances
    if vmss.overprovision:
        instances_to_keep = healthy_instances[:delta]
        instances_to_delete = healthy_instances[delta:]
        
        for instance in instances_to_delete:
            delete_instance(instance)
    
    # Add to load balancer
    for instance in instances_to_keep:
        load_balancer.add_to_backend_pool(instance)
    
    return instances_to_keep
```

### Scaling In (Removing Instances)

**Trigger**: Capacity decreased (manual or autoscale)

```
Scenario: Scale from 5 to 3 instances

Step 1: Orchestrator detects capacity change
Current: 5 instances (IDs: 0, 1, 2, 3, 4)
Desired: 3 instances
Delta: -2 instances

Step 2: Determine which instances to remove
Default policy: Highest instance IDs first
Protected instances: Skip these
Result: Select instances 3 and 4 for deletion

Step 3: Graceful removal process
For each selected instance:
  a. Remove from load balancer backend pool
     (New connections stop, existing complete)
  b. Wait for connection drain timeout (default: 15 min)
  c. Stop VM
  d. Delete VM resource
  e. Delete NIC
  f. Delete OS disk
  g. Delete data disks (if any)
  h. Release IP address
  i. Deallocate compute resources

Step 4: Capacity now: 3 instances
Remaining: Instances 0, 1, 2
```

**Scale-In Policy**:

```json
{
  "scaleInPolicy": {
    "rules": ["Default", "NewestVM", "OldestVM"],
    "forceDeletion": false
  }
}
```

Rules:

- **Default**: Highest instance ID, balanced across zones
- **NewestVM**: Delete newest instances first
- **OldestVM**: Delete oldest instances first

### Updating Instances

**Scenario**: You update the VM image in the VMSS model

```
Current state: 5 instances running Ubuntu 20.04
New model: Ubuntu 22.04
Upgrade policy: Rolling

Step 1: Model update detected
- VMSS model updated with new image reference
- Instances marked as "not running latest model"

Step 2: Orchestrator plans rolling update
Configuration:
- maxBatchInstancePercent: 20%
- pauseTimeBetweenBatches: 2 seconds
- maxUnhealthyInstancePercent: 20%

Batch plan:
- Batch 1: Instance 0 (1 instance = 20% of 5)
- Batch 2: Instance 1
- Batch 3: Instance 2
- Batch 4: Instance 3
- Batch 5: Instance 4

Step 3: Execute Batch 1
a. Check overall health
   - 5 instances, all healthy = 100% healthy
   - Threshold: 80% must be healthy
   - Status: OK to proceed

b. Update Instance 0
   - Reimage (delete OS disk, recreate from new image)
   - OR delete and recreate entire instance
   - Boot with new image
   - Run extensions
   - Wait for health probe

c. Verify health
   - Instance 0 healthy?
   - Overall health still > 80%?
   - If yes: proceed
   - If no: STOP (automatic rollback possible)

d. Wait pauseTimeBetweenBatches (2 seconds)

Step 4: Execute Batch 2
(Repeat for Instance 1)

Step 5-7: Execute remaining batches

Step 8: All instances updated
All instances now running Ubuntu 22.04
latestModelApplied: true for all
```

**Update Methods**:

1. **Reimage**:
- Deletes OS disk
- Creates new OS disk from updated image
- Keeps VM resource, NIC, IP address
- Faster (2-3 minutes)
1. **Delete and Recreate**:
- Deletes entire VM
- Creates new VM from scratch
- New NIC, new IP (unless using static)
- Slower (5-7 minutes)

### Health Monitoring & Automatic Repairs

```
Health Monitoring Loop:

1. Health Probe
   - Every 15 seconds (default)
   - HTTP GET /health endpoint
   - Expected: 200 OK response

2. Health Status Evaluation
   - Successful probe: Instance healthy
   - Failed probe: Increment failure count
   - 2 consecutive failures: Instance unhealthy

3. Automatic Repair Decision
   if instance.health == "unhealthy":
       if time_since_last_repair > grace_period:
           if repair_enabled:
               execute_repair_action()

4. Repair Actions
   - Restart: Soft reboot
   - Reimage: Reimage OS disk
   - Replace: Delete and recreate (default)

5. Repair Execution
   a. Mark instance as "under repair"
   b. Remove from load balancer
   c. Execute repair action
   d. Wait for health signal
   e. If healthy: add back to load balancer
   f. If still unhealthy: try again (max 5 attempts)
   g. If max attempts reached: admin intervention needed
```

**Repair Scenario**:

```
Time: 10:00:00 - Instance 2 healthy
Time: 10:01:00 - Health probe fails (timeout)
Time: 10:01:15 - Health probe fails again
Time: 10:01:15 - Instance marked unhealthy
Time: 10:01:15 - Start grace period (30 min default)
Time: 10:31:15 - Grace period expired, still unhealthy
Time: 10:31:15 - Trigger automatic repair
Time: 10:31:20 - Remove from load balancer
Time: 10:31:25 - Delete instance 2
Time: 10:31:30 - Create new instance 2
Time: 10:35:00 - New instance provisioned
Time: 10:36:00 - Health probe succeeds
Time: 10:36:00 - Add to load balancer
Time: 10:36:00 - Repair complete
```

### Load Balancer Integration

VMSS instances are automatically managed in load balancer backend pools:

```
Load Balancer Configuration:

Frontend IP (Public): 20.50.60.70
Backend Pool: MyWebVMSS-backendpool
├── Instance 0: 10.0.1.4:80
├── Instance 1: 10.0.1.5:80
├── Instance 2: 10.0.1.6:80
├── Instance 3: 10.0.1.7:80
└── Instance 4: 10.0.1.8:80

Health Probe:
- Protocol: HTTP
- Port: 80
- Path: /health
- Interval: 15 seconds
- Unhealthy threshold: 2 consecutive failures

Load Balancing Rule:
- Frontend: 20.50.60.70:80
- Backend: MyWebVMSS-backendpool:80
- Protocol: TCP
- Session Persistence: None
- Idle Timeout: 4 minutes
```

**Automatic Backend Pool Management**:

```python
def on_instance_created(instance):
    # Automatically add to backend pool
    backend_pool = vmss.load_balancer.backend_pool
    
    # Wait for health probe to pass
    wait_for_health(instance)
    
    # Add to pool
    backend_pool.add_member(
        ip=instance.private_ip,
        port=80
    )
    
    # Instance now receives traffic

def on_instance_deleted(instance):
    # Automatically remove from backend pool
    backend_pool = vmss.load_balancer.backend_pool
    
    # Drain connections (15 min default)
    backend_pool.drain_member(instance.private_ip)
    
    # Remove from pool
    backend_pool.remove_member(instance.private_ip)
```

-----

## Layer 4: Under the Hood - The Technology

### Azure Fabric Architecture

VMSS relies on Azure’s core infrastructure:

```
Azure Region (e.g., West Europe):

├── Azure Resource Manager (ARM)
│   └── VMSS resource definition stored here
│
├── Fabric Controllers (Control Plane)
│   ├── Fabric Controller Cluster
│   │   ├── FC Node 1 (active)
│   │   ├── FC Node 2 (standby)
│   │   └── FC Node 3 (standby)
│   │
│   └── Responsibilities:
│       ├── VM placement decisions
│       ├── Resource allocation
│       ├── Health monitoring
│       ├── Scaling orchestration
│       └── Upgrade coordination
│
├── Compute Nodes (Data Plane)
│   ├── Datacenter 1 (Availability Zone 1)
│   │   ├── Rack 1
│   │   │   ├── Server 1
│   │   │   │   ├── Hyper-V Hypervisor
│   │   │   │   ├── Host Agent
│   │   │   │   └── VMs:
│   │   │   │       ├── MyWebVMSS_0 (your VM)
│   │   │   │       └── Other customer VMs
│   │   │   └── Server 2
│   │   │
│   │   └── Rack 2
│   │
│   ├── Datacenter 2 (Availability Zone 2)
│   └── Datacenter 3 (Availability Zone 3)
│
└── Storage Systems
    ├── Managed Disk storage (Premium SSD)
    ├── OS images (stored once, used many times)
    └── Replication for durability
```

### The Orchestration Stack

```
Layer 1: User Interface
┌────────────────────────────────┐
│ Azure Portal / CLI / API       │
│ - You: "Scale to 5 instances"  │
└────────────────────────────────┘
         ↓ HTTPS REST API
         
Layer 2: Azure Resource Manager
┌────────────────────────────────┐
│ ARM (Resource Manager)         │
│ - Authenticates request        │
│ - Validates JSON               │
│ - Updates VMSS resource        │
│ - Emits event                  │
└────────────────────────────────┘
         ↓ Internal API
         
Layer 3: Compute Resource Provider
┌────────────────────────────────┐
│ Microsoft.Compute RP           │
│ - Processes VMSS operations    │
│ - Maintains VMSS state         │
│ - Coordinates with Fabric      │
└────────────────────────────────┘
         ↓ Fabric API
         
Layer 4: Fabric Controller
┌────────────────────────────────┐
│ Fabric Controller Cluster      │
│ - Receives scaling command     │
│ - Decides VM placement         │
│ - Allocates resources          │
│ - Issues creation commands     │
└────────────────────────────────┘
         ↓ Host Agent Protocol
         
Layer 5: Host Agent
┌────────────────────────────────┐
│ Host Agent (on physical server)│
│ - Receives VM creation request │
│ - Configures Hyper-V           │
│ - Attaches virtual disk        │
│ - Starts VM                    │
└────────────────────────────────┘
         ↓ Hypervisor API
         
Layer 6: Hypervisor
┌────────────────────────────────┐
│ Hyper-V Hypervisor             │
│ - Creates VM container         │
│ - Allocates virtual CPU/RAM    │
│ - Boots VM from VHD            │
└────────────────────────────────┘
         ↓
         
Layer 7: Your VM
┌────────────────────────────────┐
│ Guest OS (Linux/Windows)       │
│ - Boots up                     │
│ - Runs cloud-init              │
│ - Executes custom scripts      │
│ - Starts application           │
└────────────────────────────────┘
```

### Placement Algorithm

How does Azure decide where to put your VMs?

```python
def place_vm_instances(vmss, count):
    """
    Placement algorithm for VMSS instances
    """
    
    # Constraints
    zones = vmss.zones  # [1, 2, 3]
    fault_domains = vmss.platformFaultDomainCount  # typically 2-3
    vm_size = vmss.sku.name  # e.g., Standard_B2s
    region = vmss.location
    
    placements = []
    
    for i in range(count):
        # 1. Select Availability Zone (round-robin)
        zone = zones[i % len(zones)]
        
        # 2. Select Fault Domain (spread evenly)
        fault_domain = i % fault_domains
        
        # 3. Query Fabric for available capacity
        eligible_servers = fabric_controller.query_capacity(
            region=region,
            zone=zone,
            fault_domain=fault_domain,
            vm_size=vm_size,
            required_cpu=get_cpu_count(vm_size),
            required_ram=get_ram_gb(vm_size)
        )
        
        # 4. Select server (various criteria)
        selected_server = select_best_server(
            servers=eligible_servers,
            criteria=[
                'available_resources',  # Most capacity
                'current_load',         # Least loaded
                'proximity',            # Network proximity
                'affinity_rules'        # User-defined rules
            ]
        )
        
        # 5. Reserve resources
        reservation = fabric_controller.reserve_resources(
            server=selected_server,
            cpu=get_cpu_count(vm_size),
            ram=get_ram_gb(vm_size),
            duration='until_vm_created'
        )
        
        # 6. Record placement
        placements.append({
            'instance_id': i,
            'server': selected_server,
            'zone': zone,
            'fault_domain': fault_domain,
            'reservation': reservation
        })
    
    return placements
```

**Placement Goals**:

1. **High Availability**: Spread across zones and fault domains
1. **Performance**: Avoid overloaded servers
1. **Efficiency**: Pack VMs to minimize waste
1. **Isolation**: Separate customer workloads

### Overprovisioning Deep Dive

Why does Azure create extra instances during scale-out?

```
Problem: VM boot times are variable

Scenario: Scale from 3 to 5 instances (need +2 VMs)

Without Overprovisioning:
─────────────────────────────────────────
Create Instance 3:
  - Allocate server: 30 seconds
  - Copy OS disk: 90 seconds
  - Boot VM: 45 seconds
  - Run extensions: 60 seconds
  Total: 225 seconds (3min 45sec)

Create Instance 4:
  - Allocate server: 30 seconds
  - Copy OS disk: 120 seconds (slow disk!)
  - Boot VM: 50 seconds
  - Run extensions: 90 seconds (slow network!)
  Total: 290 seconds (4min 50sec)

Result: Must wait 4min 50sec for scale-out ❌

With Overprovisioning (create 3, keep 2):
─────────────────────────────────────────
Create Instances 3, 4, 5 simultaneously:
  
  Instance 3: 225 seconds → Healthy ✓
  Instance 4: 290 seconds → Healthy ✓
  Instance 5: 310 seconds → Healthy ✓
  
  Keep fastest 2: Instances 3 and 4
  Delete slowest: Instance 5

Result: Wait only 290 seconds (4min 50sec) ✓
But also: More consistent performance
```

**Overprovisioning Logic**:

```python
def scale_out_with_overprovision(vmss, target_count):
    current_count = len(vmss.instances)
    needed = target_count - current_count
    
    # Create 20% extra
    to_create = math.ceil(needed * 1.2)
    
    print(f"Need: {needed}, Creating: {to_create}")
    
    # Create all simultaneously
    created = []
    for i in range(to_create):
        instance = create_instance_async(vmss)
        created.append(instance)
    
    # Wait for health signals
    healthy = []
    timeout = 300  # 5 minutes
    
    while len(healthy) < needed and timeout > 0:
        for instance in created:
            if instance.is_healthy() and instance not in healthy:
                healthy.append(instance)
                print(f"Instance {instance.id} healthy at {300-timeout}s")
        
        if len(healthy) >= needed:
            break
        
        time.sleep(1)
        timeout -= 1
    
    # Keep fastest 'needed' instances
    instances_to_keep = healthy[:needed]
    instances_to_delete = [i for i in created if i not in instances_to_keep]
    
    # Delete extras
    for instance in instances_to_delete:
        delete_instance(instance)
    
    return instances_to_keep
```

### Instance State Machine

Each instance goes through a lifecycle:

```
State Machine:

┌──────────────┐
│   Pending    │ ← Initial state
└──────┬───────┘
       │ Fabric allocates resources
       ↓
┌──────────────┐
│  Allocating  │ ← Server selected, reservation made
└──────┬───────┘
       │ Start VM creation
       ↓
┌──────────────┐
│   Creating   │ ← Disk copied, NIC created
└──────┬───────┘
       │ Boot VM
       ↓
┌──────────────┐
│   Starting   │ ← VM booting
└──────┬───────┘
       │ OS boots
       ↓
┌──────────────┐
│   Running    │ ← VM running, extensions deploying
└──────┬───────┘
       │ Health probe
       ↓
┌──────────────┐
│   Healthy    │ ← Health probe passed → ADD TO LB
└──┬───────┬───┘
   │       │
   │       │ Health probe fails
   │       ↓
   │   ┌──────────────┐
   │   │  Unhealthy   │ ← Automatic repair triggered
   │   └──────┬───────┘
   │          │
   │          ↓
   │   ┌──────────────┐
   │   │  Repairing   │ ← Reimage or replace
   │   └──────┬───────┘
   │          │
   │          └─────→ (back to Creating/Starting)
   │
   │ Manual delete or scale-in
   ↓
┌──────────────┐
│   Deleting   │ ← VM being deleted
└──────┬───────┘
       │
       ↓
┌──────────────┐
│   Deleted    │ ← Resources deallocated
└──────────────┘
```

### Extension Deployment

Extensions run after VM boots:

```
Extension Execution Flow:

1. VM Boots
   - OS starts
   - Cloud-init runs (Linux) or FirstLogon (Windows)
   - Basic networking configured

2. Azure VM Agent Starts
   - Pre-installed in Azure images
   - Connects to Azure Fabric
   - Reports "VM ready"

3. Fabric Sends Extension List
   Extensions to deploy:
   - CustomScript
   - ApplicationHealth
   - AzureMonitor
   - Any user-defined extensions

4. VM Agent Downloads Extensions
   For each extension:
     - Download from Azure storage
     - Verify signature
     - Extract to /var/lib/waagent/ (Linux)

5. VM Agent Executes Extensions
   Execute in priority order:
   
   CustomScript (priority 1):
     - Download script from URL
     - Execute: bash install.sh
     - Wait for completion
     - Report status: Success/Failure
   
   ApplicationHealth (priority 2):
     - Configure health probe
     - Start health monitoring
     - Report status: Success
   
   AzureMonitor (priority 3):
     - Install monitoring agent
     - Configure metrics collection
     - Report status: Success

6. All Extensions Complete
   - VM Agent reports "Extensions ready"
   - Instance marked "Provisioned"
   - Health probe begins
```

**Extension Failure Handling**:

```
If extension fails:
  - Instance marked "Provisioning failed"
  - Instance NOT added to load balancer
  - If overprovision enabled: This instance discarded
  - If overprovision disabled: Manual intervention needed
  - Auto-repair MAY delete and recreate
```

### Networking Configuration

Each instance gets networking automatically configured:

```
Instance Network Setup:

1. Virtual NIC Created
   NIC Name: myweb_nic_abc123
   Attached to: VMSS instance
   
2. IP Address Allocation
   Method: Dynamic (from subnet DHCP)
   Subnet: 10.0.1.0/24
   Assigned IP: 10.0.1.4
   
3. NSG Applied
   NSG: MyWebVMSS-nsg
   Rules: Applied at NIC level
   
4. Load Balancer Association
   Backend Pool: MyWebVMSS-backendpool
   Health Probe: HTTP /health
   
5. DNS Configuration
   DNS Servers: Azure provided (168.63.129.16)
   DNS Suffix: internal.cloudapp.net
   Hostname: myweb000000.internal.cloudapp.net

6. Outbound Connectivity
   Method: SNAT via Load Balancer
   Public IP: Load balancer's frontend IP
   (All instances share same outbound IP)
```

-----

## Layer 5: Mapping to Compute & Virtualization Stack

### The Virtualization Hierarchy

```
┌─────────────────────────────────────────────┐
│ Layer 5: Application                        │
│ - nginx, Apache, Node.js                    │
│ - Your code running                         │
│ VMSS: No involvement                        │
└─────────────────────────────────────────────┘
              ↕
┌─────────────────────────────────────────────┐
│ Layer 4: Operating System                   │
│ - Linux kernel / Windows kernel             │
│ - Process management                        │
│ - File system                               │
│ VMSS: Deployed via image reference          │
└─────────────────────────────────────────────┘
              ↕
┌─────────────────────────────────────────────┐
│ Layer 3: Guest OS Extensions                │
│ - Azure VM Agent (waagent)                  │
│ - Custom scripts                            │
│ - Monitoring agents                         │
│ VMSS: Deployed via extensionProfile         │
└─────────────────────────────────────────────┘
              ↕
┌─────────────────────────────────────────────┐
│ Layer 2: Virtual Machine                    │
│ - Virtual CPU (vCPU)                        │
│ - Virtual RAM                               │
│ - Virtual disk (VHD/VHDX)                   │
│ - Virtual NIC                               │
│ VMSS: ✓ Creates and manages VMs            │
└─────────────────────────────────────────────┘
              ↕
┌─────────────────────────────────────────────┐
│ Layer 1: Hypervisor (Hyper-V)              │
│ - Virtual machine monitor                   │
│ - Hardware abstraction                      │
│ - Resource allocation                       │
│ - Memory management                         │
│ VMSS: Orchestrates via Fabric Controller    │
└─────────────────────────────────────────────┘
              ↕
┌─────────────────────────────────────────────┐
│ Layer 0: Physical Hardware                  │
│ - Physical CPU (Intel/AMD)                  │
│ - Physical RAM (DDR4/DDR5)                  │
│ - Physical disk (SSD/NVMe)                  │
│ - Physical NIC (25/100 Gbps)                │
│ VMSS: Allocates via Fabric Controller       │
└─────────────────────────────────────────────┘
```

### VM Size Mapping to Physical Resources

```
Example: Standard_B2s

Virtual Specification:
- vCPUs: 2
- RAM: 4 GB
- Temp storage: 8 GB
- Max data disks: 4
- Max NICs: 2
- Network bandwidth: Moderate

Physical Allocation:
┌────────────────────────────────┐
│ Physical Server                │
│ Intel Xeon: 64 cores           │
│ RAM: 512 GB                    │
│                                │
│ ┌──────────────────────────┐   │
│ │ Hyper-V Hypervisor       │   │
│ │                          │   │
│ │ Your VM (Standard_B2s)   │   │
│ │ ├─ 2 vCPUs (2 threads)   │   │
│ │ ├─ 4 GB RAM (reserved)   │   │
│ │ ├─ CPU credits pool      │   │
│ │ └─ Network throttle: ~1Gbps│ │
│ │                          │   │
│ │ Other VMs...             │   │
│ │ (40-60 VMs per server)   │   │
│ └──────────────────────────┘   │
└────────────────────────────────┘

Resource Allocation:
- CPU: Scheduled by hypervisor, not pinned
- RAM: Reserved exclusively for this VM
- Disk: Virtual disk on Azure managed storage
- Network: Virtual NIC with bandwidth limit
```

### Storage Architecture

```
VM Storage Hierarchy:

Your VM (Instance 0):
├── OS Disk (Managed Disk)
│   ├── Type: Premium SSD
│   ├── Size: 30 GB
│   ├── Performance: 120 IOPS, 25 MB/s
│   └── Physical location: Azure Storage
│       ├── 3 replicas (LRS)
│       ├── SSD storage cluster
│       └── Connected via RDMA network
│
├── Temporary Disk (Ephemeral)
│   ├── Size: 8 GB (Standard_B2s)
│   ├── Location: Local to physical server
│   ├── Lost on: Reboot, reimage, move
│   └── Use: Swap, temp files only
│
└── Data Disks (Optional)
    └── Disk 1: 128 GB Premium SSD

Image Storage:
├── Publisher: Canonical
├── Offer: UbuntuServer
├── SKU: 20_04-lts-gen2
└── Version: latest
    ├── Stored: Azure-managed image gallery
    ├── Replication: All regions
    └── Copy-on-write: Efficient disk creation
```

**How OS Disk is Created**:

```
Image Reference:
Canonical:UbuntuServer:20_04-lts-gen2:latest

↓ VMSS creates instance

1. Find base image VHD
   Location: Azure image gallery
   Size: 30 GB
   Format: VHD (Virtual Hard Disk)

2. Create snapshot reference
   (Copy-on-write, not full copy)
   
3. Create managed disk
   Name: MyWebVMSS_0_OsDisk_1_abc123
   Type: Premium_LRS
   Size: 30 GB
   
4. Initialize from image
   Method: Copy-on-write
   - Reads from base image
   - Writes to instance disk
   - Only changed blocks stored
   
5. Attach to VM
   Controller: SCSI
   LUN: 0 (boot disk)
   
6. Boot VM
   BIOS reads disk → boots OS

Total time: 60-90 seconds
```

### Networking Stack

```
Network Packet Flow (Outbound):

Application (nginx):
  ├─ Creates HTTP response
  └─ Sends to socket

Operating System (Linux kernel):
  ├─ TCP/IP stack processes
  ├─ Adds headers (TCP, IP)
  └─ Sends to virtual NIC

Virtual NIC:
  ├─ Receives from guest OS
  ├─ Queues packet
  └─ Signals hypervisor

Hyper-V vSwitch:
  ├─ Receives packet
  ├─ NSG evaluation
  ├─ NAT (if needed)
  └─ Routes to physical NIC

Physical NIC:
  ├─ Transmits packet
  └─ To Azure network fabric

Azure Network Fabric:
  ├─ Receives packet
  ├─ Routes to load balancer
  └─ Load balancer forwards to internet
```

-----

## Layer 6: Scaling Flow - Instance Lifecycle

### Complete Scale-Out Flow (Detailed)

Let’s trace creating a single instance from start to finish:

```
Scenario: Scale from 2 to 3 instances

Timeline:

T+0ms: You click "Scale to 3" in portal
├─ Browser sends HTTPS POST
└─ To: https://management.azure.com/subscriptions/.../virtualMachineScaleSets/MyWebVMSS?api-version=2023-03-01

T+50ms: Azure Resource Manager receives request
├─ Authenticate: Bearer token validated
├─ Authorize: Check RBAC permissions
├─ Parse: Extract capacity=3
└─ Validate: Is request valid?

T+100ms: ARM updates VMSS resource
├─ Database transaction: Update SKU.capacity
├─ Emit event: "VMSS capacity changed"
└─ Response to browser: 200 OK

T+200ms: Compute Resource Provider notified
├─ Receives event: "VMSS capacity changed"
├─ Reads current state: 2 instances
├─ Calculates delta: +1 instance needed
└─ Sends request to Fabric Controller

T+500ms: Fabric Controller processes request
├─ Determines placement:
│   ├─ Zone selection: Zone 1 (round-robin)
│   ├─ Fault domain: FD 0
│   └─ Query available capacity
│
├─ Finds eligible servers:
│   ├─ Server A in Zone 1, FD 0 - 40% CPU, 60% RAM
│   ├─ Server B in Zone 1, FD 0 - 30% CPU, 50% RAM ← Selected
│   └─ Server C in Zone 1, FD 0 - 70% CPU, 80% RAM
│
└─ Allocates resources on Server B:
    ├─ Reserve: 2 vCPUs
    ├─ Reserve: 4 GB RAM
    └─ Generate VM ID: abc-123-def-456

T+1s: Host Agent on Server B receives request
├─ Request: Create VM
├─ VM ID: abc-123-def-456
├─ Instance ID: 2
├─ Zone: 1
├─ FD: 0
└─ VM Profile: {...}

T+1.5s: Host Agent starts VM creation
├─ Create Virtual NIC:
│   ├─ Name: myweb_nic_2_xyz789
│   ├─ MAC: 00:0D:3A:12:34:56
│   ├─ Request IP from subnet
│   │   └─ Azure DHCP: Assigned 10.0.1.6
│   └─ Attach NSG: MyWebVMSS-nsg
│
└─ Create OS Disk:
    ├─ Name: MyWebVMSS_2_OsDisk_1_diskid
    ├─ Size: 30 GB
    ├─ Type: Premium_LRS
    ├─ Image: Canonical Ubuntu 20.04
    └─ Copy-on-write from base image

T+60s: Disk created and attached

T+62s: Configure Hyper-V
├─ Create VM configuration
├─ Assign vCPUs: 2
├─ Assign RAM: 4 GB
├─ Attach virtual NIC
├─ Attach OS disk
└─ Set boot order: Disk first

T+65s: Start VM (Power On)
├─ Hyper-V starts VM
├─ Virtual BIOS initializes
├─ Boot from virtual disk
└─ GRUB loads Linux kernel

T+70s: Linux boots
├─ Kernel initializes
├─ Systemd starts services
├─ Cloud-init executes
│   ├─ Reads instance metadata
│   ├─ Sets hostname: myweb000002
│   ├─ Configures SSH keys
│   └─ Runs custom-data script
└─ Network interfaces up

T+90s: Azure VM Agent starts
├─ Detects Azure environment
├─ Connects to Azure Fabric
│   └─ URL: http://168.63.129.16
├─ Reports: "VM ready"
└─ Requests extensions list

T+95s: Fabric sends extensions
Extensions to deploy:
1. CustomScript
2. ApplicationHealth

T+100s: Deploy CustomScript extension
├─ Download: https://storage.blob/scripts/install.sh
├─ Execute: bash install.sh
│   ├─ apt update
│   ├─ apt install nginx
│   ├─ Configure nginx
│   ├─ systemctl start nginx
│   └─ systemctl enable nginx
└─ Report: Success (T+160s)

T+165s: Deploy ApplicationHealth extension
├─ Configure health probe
│   ├─ Protocol: HTTP
│   ├─ Port: 80
│   └─ Path: /health
├─ Test endpoint: http://localhost/health
│   └─ Response: 200 OK
└─ Report: Success

T+170s: All extensions complete
├─ VM Agent reports: "Extensions ready"
├─ Instance marked: "Provisioning succeeded"
└─ Instance state: "Running"

T+175s: Health monitoring begins
├─ Application Health extension starts
├─ First probe: http://10.0.1.6/health
└─ Result: 200 OK → Instance HEALTHY

T+180s: Add to load balancer
├─ Backend pool: MyWebVMSS-backendpool
├─ Add member: 10.0.1.6:80
├─ Load balancer starts health probes
│   ├─ Probe 1: 200 OK
│   └─ Probe 2: 200 OK
└─ Instance added to rotation

T+185s: Scale-out complete!
├─ Capacity: 3 instances
├─ Instance 2: Healthy and serving traffic
└─ Portal updated: "Succeeded"

Total time: ~3 minutes
```

### Instance Update Flow

```
Scenario: Update VM image (Rolling upgrade)

Current: 3 instances running Ubuntu 20.04
Target: Ubuntu 22.04
Policy: Rolling, maxBatch 33% (1 instance at a time)

Timeline:

T+0s: You update VMSS model
├─ Change imageReference: Ubuntu 22.04
└─ Click "Save"

T+1s: ARM updates VMSS resource
├─ New model stored
└─ Instances marked: latestModelApplied=false

T+2s: Compute RP detects model change
├─ Trigger: Upgrade policy = Rolling
└─ Start rolling upgrade orchestration

T+5s: Orchestrator plans upgrade
├─ Total instances: 3
├─ maxBatchInstancePercent: 33%
├─ Batch size: 1 instance
└─ Batches: [Instance 0], [Instance 1], [Instance 2]

T+10s: Start Batch 1 (Instance 0)
├─ Check health: 3/3 healthy (100%)
├─ Threshold: 80% must be healthy
└─ Status: OK to proceed

T+11s: Upgrade Instance 0
├─ Method: Reimage (faster than recreate)
│
├─ Step 1: Prepare (T+11s)
│   ├─ Mark instance: "Updating"
│   ├─ Remove from load balancer
│   │   └─ Drain connections (wait 15 min? No, immediate for testing)
│   └─ Stop accepting new traffic
│
├─ Step 2: Stop VM (T+12s)
│   ├─ Send ACPI shutdown signal
│   ├─ Wait for graceful shutdown (30s timeout)
│   └─ Force power off if needed
│
├─ Step 3: Delete OS Disk (T+45s)
│   ├─ Unmount from VM
│   ├─ Delete managed disk
│   └─ Release storage
│
├─ Step 4: Create new OS Disk (T+50s)
│   ├─ Image: Ubuntu 22.04
│   ├─ Size: 30 GB
│   ├─ Copy from base image
│   └─ Time: 60 seconds
│
├─ Step 5: Attach new disk (T+110s)
│   └─ Mount to VM
│
├─ Step 6: Start VM (T+115s)
│   ├─ Power on
│   ├─ BIOS → GRUB → Kernel
│   └─ Boot time: 30 seconds
│
├─ Step 7: OS initialization (T+145s)
│   ├─ Cloud-init runs
│   ├─ Configure networking
│   ├─ Set hostname
│   └─ Install Azure VM Agent
│
├─ Step 8: Run extensions (T+160s)
│   ├─ CustomScript: install nginx
│   └─ ApplicationHealth: configure probe
│   └─ Time: 60 seconds
│
└─ Step 9: Health check (T+220s)
    ├─ Probe: http://10.0.1.4/health
    ├─ Result: 200 OK
    └─ Status: HEALTHY

T+225s: Instance 0 updated successfully
├─ Add back to load balancer
├─ Mark: latestModelApplied=true
└─ Update complete for Batch 1

T+227s: Wait pauseTimeBetweenBatches (2s)

T+229s: Start Batch 2 (Instance 1)
├─ Check health: 3/3 healthy (100%)
└─ Proceed with upgrade
    (Same process as Instance 0)

T+449s: Instance 1 updated successfully

T+451s: Wait pauseTimeBetweenBatches (2s)

T+453s: Start Batch 3 (Instance 2)
├─ Check health: 3/3 healthy (100%)
└─ Proceed with upgrade
    (Same process as Instance 0)

T+673s: Instance 2 updated successfully

T+675s: Rolling upgrade complete!
├─ All instances: latestModelApplied=true
├─ All instances: Running Ubuntu 22.04
└─ Total time: ~11 minutes

Status:
- 0 downtime (instances cycled gradually)
- Always 2/3 instances available
- Load balanced throughout
```

### Automatic Repair Flow

```
Scenario: Instance becomes unhealthy

Timeline:

T+0s: Instance 1 running normally
├─ Serving traffic
├─ Health probes: 200 OK
└─ Status: Healthy

T+60s: Application crashes (nginx stopped)

T+75s: Health probe fails
├─ HTTP GET http://10.0.1.5/health
├─ Result: Connection refused
└─ Failure count: 1

T+90s: Health probe fails again
├─ Result: Connection refused
├─ Failure count: 2
└─ Threshold: 2 consecutive failures
    → Instance marked UNHEALTHY

T+90s: Automatic repair triggered
├─ Policy: automaticRepairsPolicy.enabled = true
├─ Grace period: 30 minutes
└─ Start grace period timer

T+30m 90s: Grace period expired
├─ Instance still unhealthy
├─ Repair action: Replace
└─ Start repair process

T+30m 90s: Remove from load balancer
├─ Backend pool: Remove 10.0.1.5:80
└─ Stop routing traffic

T+30m 95s: Delete instance
├─ Stop VM
├─ Delete VM resource
├─ Delete NIC
├─ Delete OS disk
└─ Deallocate resources

T+30m 100s: Create new instance
├─ Instance ID: 1 (reuse same ID)
├─ Placement: Zone 2, FD 1 (different from before)
├─ Follow normal creation flow
└─ Time: ~3 minutes

T+33m 100s: New instance ready
├─ Provisioning: Succeeded
├─ Extensions: Deployed
├─ Health check: 200 OK
└─ Status: Healthy

T+33m 105s: Add to load balancer
├─ Backend pool: Add 10.0.1.5:80
└─ Resume traffic

T+33m 110s: Repair complete
├─ Instance 1: Replaced successfully
├─ Repair count: 1
└─ Monitoring continues

Repair Tracking:
- Max repair attempts: 5
- Current attempt: 1
- If 5 failures: Stop automatic repair, alert admin
```

-----

## Layer 7: Physical Implementation

### Physical Server Architecture

```
Typical Azure Compute Server:

Physical Chassis:
├── Motherboard
│   ├── Dual Intel Xeon Scalable (e.g., Platinum 8370C)
│   │   ├── 32 cores per CPU = 64 cores total
│   │   ├── 2 threads per core = 128 threads
│   │   └── Base clock: 2.8 GHz, Boost: 3.5 GHz
│   │
│   ├── RAM: 512 GB - 1 TB DDR4
│   │   ├── 16x 32GB DIMMs or 16x 64GB DIMMs
│   │   └── ECC enabled
│   │
│   ├── Network Interface Cards
│   │   ├── NIC 1: 25 Gbps (data plane)
│   │   ├── NIC 2: 25 Gbps (data plane)
│   │   └── NIC 3: 1 Gbps (management)
│   │
│   └── Storage Controllers
│       ├── Local NVMe: 2x 1TB (temp storage)
│       └── RDMA controller: For managed disks
│
├── Operating System: Windows Server (Hyper-V host)
│
└── Hyper-V Hypervisor
    ├── Host OS (minimal)
    ├── Virtual Switch (for VM networking)
    └── Virtual Machines
        ├── VMSS Instance (Standard_B2s): 2 vCPU, 4GB RAM
        ├── VMSS Instance (Standard_D4s): 4 vCPU, 16GB RAM
        ├── Other customer VM: 8 vCPU, 32GB RAM
        ├── Other customer VM: 2 vCPU, 8GB RAM
        └── ... (40-60 VMs typical)
```

### Physical Topology

```
Azure Datacenter (Availability Zone 1):

Building:
├── Power Systems
│   ├── Multiple utility feeds
│   ├── UPS systems
│   ├── Backup generators
│   └── N+1 redundancy
│
├── Cooling Systems
│   ├── CRAC units (Computer Room AC)
│   ├── Hot aisle / cold aisle
│   └── Free cooling when possible
│
└── Server Halls
    ├── Hall A
    │   ├── Row 1
    │   │   ├── Rack 01
    │   │   │   ├── TOR Switch (48 ports, 25Gbps)
    │   │   │   ├── Server 01 ← Your VMSS Instance 0 here
    │   │   │   ├── Server 02
    │   │   │   ├── ... (40 servers per rack)
    │   │   │   └── Server 40
    │   │   │
    │   │   ├── Rack 02 ← Your VMSS Instance 1 here (different rack = different FD)
    │   │   └── ... (40-60 racks per row)
    │   │
    │   └── Row 2..N
    │
    └── Hall B..N

Network Fabric:
Rack 01 TOR → Aggregation Switch → Core Router → Internet Gateway
Rack 02 TOR ↗

Storage Systems (separate from compute):
├── Storage Cluster 1 (Premium SSD)
│   └── Your VMSS OS disks stored here
└── Storage Cluster 2 (Standard HDD)

Networking:
- Within rack: 25 Gbps
- Between racks: 100 Gbps
- Between zones: 100 Gbps+ (dedicated dark fiber)
```

### Physical Resource Allocation

```
Physical Server: server-az1-row1-rack01-srv03

Total Resources:
- CPU: 64 cores (128 threads)
- RAM: 512 GB
- Network: 25 Gbps per NIC × 2 = 50 Gbps
- Temp Storage: 2 TB NVMe

Current Allocation:

┌────────────────────────────────────────────────┐
│ Hypervisor Overhead: 2 cores, 8 GB RAM        │
├────────────────────────────────────────────────┤
│ Your VMSS Instance 0 (Standard_B2s)           │
│ - vCPU: 2 cores                                │
│ - RAM: 4 GB (reserved)                         │
│ - Network: ~1 Gbps (throttled)                 │
│ - Temp: 8 GB                                   │
├────────────────────────────────────────────────┤
│ Customer A VM (Standard_D8s_v3)                │
│ - vCPU: 8 cores                                │
│ - RAM: 32 GB                                   │
│ - Network: ~4 Gbps                             │
├────────────────────────────────────────────────┤
│ Customer B VM (Standard_B4ms)                  │
│ - vCPU: 4 cores                                │
│ - RAM: 16 GB                                   │
├────────────────────────────────────────────────┤
│ ... 45 more VMs ...                            │
├────────────────────────────────────────────────┤
│ Total Allocated:                               │
│ - vCPU: 120 cores (93% of 128 threads)        │
│ - RAM: 480 GB (93% of 512 GB)                 │
│ - Network: 45 Gbps (90% of 50 Gbps)           │
└────────────────────────────────────────────────┘

Allocation Strategy:
- Oversubscription: Yes (vCPU > physical cores)
  Ratio: ~2:1 typical
  Reason: VMs rarely use 100% CPU simultaneously
  
- RAM: No oversubscription (strictly reserved)
  Your 4 GB is physically reserved
  Cannot be used by other VMs
  
- Network: Bandwidth throttling per VM
  Rate limiting enforced by vSwitch
  
- Storage: Managed disks are remote
  RDMA network to storage cluster
  Local NVMe for temp disk only
```

### Fault Domain and Update Domain Distribution

```
Physical Mapping:

Availability Zone 1:
├── Rack 01 (Fault Domain 0)
│   └── Server 03 → Your Instance 0
│       └── Update Domain: 0
│
├── Rack 02 (Fault Domain 1)
│   └── Server 15 → Your Instance 1
│       └── Update Domain: 1
│
└── Rack 03 (Fault Domain 2)
    └── Server 27 → Your Instance 2
        └── Update Domain: 2

Why Different Racks?
- Power: Each rack has separate power
- Network: Each rack has separate TOR switch
- Maintenance: Rack maintenance affects only 1 FD
- Failure: Single rack failure = only 1 instance down

Update Domains:
- Logical grouping for planned maintenance
- Azure updates one UD at a time
- Your 3 instances in 3 UDs = max availability
- During maintenance: 2/3 instances always available
```

### Network Path (Physical)

```
Outbound Packet Journey:

Your Application (Instance 0):
- Server: server-az1-row1-rack01-srv03
- VM: MyWebVMSS_0
- IP: 10.0.1.4

↓ (virtual)

Hyper-V vSwitch:
- Software switch in hypervisor
- NSG enforcement here
- NAT here
- VLAN tagging

↓ (physical)

Server NIC:
- Intel 25 Gbps NIC
- Physical MAC: AA:BB:CC:DD:EE:FF
- Port: Connected to TOR

↓ (copper/fiber)

Top-of-Rack Switch:
- Rack 01 TOR
- 48 ports, 25 Gbps each
- Uplinks: 2× 100 Gbps to aggregation

↓ (fiber)

Aggregation Switch:
- Aggregates 10-20 racks
- Routing decisions
- Load balancing

↓ (fiber)

Core Router:
- Datacenter backbone
- BGP routing
- 400 Gbps+ links

↓ (fiber)

Edge Router:
- Connection to internet
- Peering with ISPs
- DDoS protection

↓ (fiber)

Internet
```

### Storage Path (Physical)

```
How Your OS Disk is Accessed:

Your VM (Instance 0):
- Reads/writes to /dev/sda1
- Thinks it's a local disk
- Actually: Remote managed disk

↓ (virtual)

Hyper-V Storage Stack:
- Virtual disk controller
- I/O requests queued
- Sent to storage driver

↓ (RDMA network)

Storage Cluster:
- Separate storage servers
- Premium SSD tier
- 3 replicas for durability

Storage Server A (Primary):
├── Your disk: MyWebVMSS_0_OsDisk
├── Location: /dev/nvme0n1p456
└── Size: 30 GB

Storage Server B (Replica 1):
└── Copy of your disk

Storage Server C (Replica 2):
└── Copy of your disk

Write Operation:
1. VM writes 4KB block
2. Sent via RDMA to Storage Server A
3. Server A writes to local SSD
4. Server A replicates to B and C (synchronous)
5. All 3 writes complete
6. Acknowledge to VM
7. Latency: ~5ms

Read Operation:
1. VM reads 4KB block
2. Sent via RDMA to Storage Server A
3. Server A reads from local SSD
4. Returns data to VM
5. Latency: ~2ms

RDMA:
- Remote Direct Memory Access
- Bypasses CPU
- Ultra-low latency
- High throughput
```

-----

## Layer 8: Practical Examples

### Example 1: Basic Web Server VMSS

**Scenario**: Deploy scalable web servers

```json
{
  "name": "WebServerVMSS",
  "location": "westeurope",
  "sku": {
    "name": "Standard_B2s",
    "capacity": 3
  },
  "properties": {
    "orchestrationMode": "Uniform",
    "upgradePolicy": {
      "mode": "Rolling",
      "rollingUpgradePolicy": {
        "maxBatchInstancePercent": 20,
        "maxUnhealthyInstancePercent": 20,
        "pauseTimeBetweenBatches": "PT0S"
      }
    },
    "virtualMachineProfile": {
      "storageProfile": {
        "imageReference": {
          "publisher": "Canonical",
          "offer": "0001-com-ubuntu-server-focal",
          "sku": "20_04-lts-gen2",
          "version": "latest"
        },
        "osDisk": {
          "createOption": "FromImage",
          "caching": "ReadWrite",
          "managedDisk": {
            "storageAccountType": "Premium_LRS"
          }
        }
      },
      "osProfile": {
        "computerNamePrefix": "web",
        "adminUsername": "azureuser",
        "customData": "IyEvYmluL2Jhc2gKYXB0IHVwZGF0ZQphcHQgaW5zdGFsbCAteSBuZ2lueApzeXN0ZW1jdGwgZW5hYmxlIG5naW54CnN5c3RlbWN0bCBzdGFydCBuZ2lueAplY2hvICI8aDE+SGVsbG8gZnJvbSAkKGhvc3RuYW1lKTwvaDE+IiA+IC92YXIvd3d3L2h0bWwvaW5kZXguaHRtbA=="
      },
      "networkProfile": {
        "networkInterfaceConfigurations": [
          {
            "name": "webnic",
            "properties": {
              "primary": true,
              "ipConfigurations": [
                {
                  "name": "webipconfig",
                  "properties": {
                    "subnet": {
                      "id": "/subscriptions/.../subnets/default"
                    },
                    "loadBalancerBackendAddressPools": [
                      {
                        "id": "/subscriptions/.../backendAddressPools/webBackendPool"
                      }
                    ]
                  }
                }
              ],
              "networkSecurityGroup": {
                "id": "/subscriptions/.../networkSecurityGroups/web-nsg"
              }
            }
          }
        ]
      },
      "extensionProfile": {
        "extensions": [
          {
            "name": "HealthExtension",
            "properties": {
              "publisher": "Microsoft.ManagedServices",
              "type": "ApplicationHealthLinux",
              "typeHandlerVersion": "1.0",
              "autoUpgradeMinorVersion": true,
              "settings": {
                "protocol": "http",
                "port": 80,
                "requestPath": "/"
              }
            }
          }
        ]
      }
    },
    "overprovision": true,
    "automaticRepairsPolicy": {
      "enabled": true,
      "gracePeriod": "PT30M"
    }
  },
  "zones": ["1", "2", "3"]
}
```

**Custom-Data Script** (Base64 decoded):

```bash
#!/bin/bash
apt update
apt install -y nginx
systemctl enable nginx
systemctl start nginx
echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
```

**What This Does**:

1. Creates 3 instances across 3 zones
1. Each instance runs Ubuntu 20.04
1. Custom script installs and starts nginx
1. Health monitoring on HTTP port 80
1. Auto-repair if instance fails
1. Rolling updates (20% at a time)
1. Load balanced via backend pool

**Access**:

```bash
# Via load balancer public IP
curl http://20.50.60.70

# Response cycles through:
<h1>Hello from web000000</h1>
<h1>Hello from web000001</h1>
<h1>Hello from web000002</h1>
```

### Example 2: Autoscaling Based on CPU

**Autoscale Configuration**:

```json
{
  "type": "Microsoft.Insights/autoscaleSettings",
  "name": "WebServerAutoscale",
  "properties": {
    "enabled": true,
    "targetResourceUri": "/subscriptions/.../virtualMachineScaleSets/WebServerVMSS",
    "profiles": [
      {
        "name": "CPU-based scaling",
        "capacity": {
          "minimum": "2",
          "maximum": "10",
          "default": "3"
        },
        "rules": [
          {
            "metricTrigger": {
              "metricName": "Percentage CPU",
              "metricResourceUri": "/subscriptions/.../virtualMachineScaleSets/WebServerVMSS",
              "timeGrain": "PT1M",
              "statistic": "Average",
              "timeWindow": "PT5M",
              "timeAggregation": "Average",
              "operator": "GreaterThan",
              "threshold": 70
            },
            "scaleAction": {
              "direction": "Increase",
              "type": "ChangeCount",
              "value": "1",
              "cooldown": "PT5M"
            }
          },
          {
            "metricTrigger": {
              "metricName": "Percentage CPU",
              "metricResourceUri": "/subscriptions/.../virtualMachineScaleSets/WebServerVMSS",
              "timeGrain": "PT1M",
              "statistic": "Average",
              "timeWindow": "PT10M",
              "timeAggregation": "Average",
              "operator": "LessThan",
              "threshold": 30
            },
            "scaleAction": {
              "direction": "Decrease",
              "type": "ChangeCount",
              "value": "1",
              "cooldown": "PT10M"
            }
          }
        ]
      }
    ],
    "notifications": [
      {
        "operation": "Scale",
        "email": {
          "customEmails": ["ops@example.com"]
        },
        "webhooks": [
          {
            "serviceUri": "https://hooks.slack.com/services/XXX"
          }
        ]
      }
    ]
  }
}
```

**Behavior**:

```
Normal Load (40% CPU):
- Instances: 3
- Status: Stable

Traffic Spike (CPU rises):
T+0m:   CPU = 50% avg (no action)
T+1m:   CPU = 60% avg (no action)
T+2m:   CPU = 75% avg (no action, collecting data)
T+3m:   CPU = 80% avg (no action, collecting data)
T+4m:   CPU = 82% avg (no action, collecting data)
T+5m:   CPU = 78% avg over last 5 min → TRIGGER!
        ↓
        Scale out: +1 instance
        New capacity: 4 instances
        Cooldown: 5 minutes (no scaling during this time)
        
T+8m:   New instance ready and serving traffic
T+10m:  CPU = 65% avg (load distributed)
T+10m:  Cooldown expires
T+15m:  CPU still 75% avg → TRIGGER again!
        ↓
        Scale out: +1 instance
        New capacity: 5 instances

Traffic Decreases:
T+30m:  CPU = 25% avg over last 10 min → TRIGGER scale-in!
        ↓
        Scale in: -1 instance
        New capacity: 4 instances
        Cooldown: 10 minutes
        
T+40m:  Cooldown expires
T+50m:  CPU still 20% avg → TRIGGER scale-in!
        ↓
        Scale in: -1 instance
        New capacity: 3 instances
        
Steady State: 3 instances (default)
```

### Example 3: Scheduled Scaling

**Scenario**: Scale up during business hours, scale down at night

```json
{
  "profiles": [
    {
      "name": "Business Hours (Mon-Fri 8AM-6PM)",
      "capacity": {
        "minimum": "5",
        "maximum": "20",
        "default": "10"
      },
      "rules": [
        {
          "metricTrigger": {
            "metricName": "Percentage CPU",
            "threshold": 70
          },
          "scaleAction": {
            "direction": "Increase",
            "value": "2"
          }
        }
      ],
      "recurrence": {
        "frequency": "Week",
        "schedule": {
          "timeZone": "W. Europe Standard Time",
          "days": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
          "hours": [8],
          "minutes": [0]
        }
      }
    },
    {
      "name": "After Hours",
      "capacity": {
        "minimum": "2",
        "maximum": "5",
        "default": "2"
      },
      "rules": [],
      "recurrence": {
        "frequency": "Week",
        "schedule": {
          "timeZone": "W. Europe Standard Time",
          "days": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
          "hours": [18],
          "minutes": [0]
        }
      }
    },
    {
      "name": "Weekend",
      "capacity": {
        "minimum": "2",
        "maximum": "5",
        "default": "2"
      },
      "rules": [],
      "recurrence": {
        "frequency": "Week",
        "schedule": {
          "timeZone": "W. Europe Standard Time",
          "days": ["Saturday", "Sunday"],
          "hours": [0],
          "minutes": [0]
        }
      }
    }
  ]
}
```

**Schedule Behavior**:

```
Monday 8:00 AM:
- Switch to "Business Hours" profile
- Current: 2 instances
- Target: 10 instances (default)
- Action: Scale out +8 instances
- Time to scale: ~10 minutes

Monday 8:10 AM:
- Capacity: 10 instances
- Min: 5, Max: 20
- CPU-based autoscale active (threshold 70%)

Monday 6:00 PM:
- Switch to "After Hours" profile
- Current: 12 instances (scaled during day)
- Target: 2 instances (default)
- Action: Scale in -10 instances
- Time to scale: ~15 minutes (gradual)

Saturday 12:00 AM:
- Switch to "Weekend" profile
- Capacity: 2 instances
- Min: 2, Max: 5
- No CPU-based rules
```

### Example 4: Multi-Region VMSS with Traffic Manager

**Architecture**:

```
Azure Traffic Manager (Global)
├── Priority: West Europe (Primary)
│   └── VMSS: 5 instances
│       └── Load Balancer: westvmss-lb (20.50.60.70)
│
└── Failover: North Europe (Secondary)
    └── VMSS: 3 instances
        └── Load Balancer: northvmss-lb (20.60.70.80)

Traffic Routing:
- Normal: All traffic to West Europe
- West Europe down: Automatic failover to North Europe
- Health probes: Every 10 seconds
- Failover time: ~30 seconds
```

**VMSS in Each Region**:

```json
// West Europe VMSS
{
  "name": "WebVMSS-WestEU",
  "location": "westeurope",
  "sku": {"capacity": 5}
}

// North Europe VMSS  
{
  "name": "WebVMSS-NorthEU",
  "location": "northeurope",
  "sku": {"capacity": 3}
}
```

**Traffic Manager**:

```json
{
  "type": "Microsoft.Network/trafficmanagerprofiles",
  "name": "GlobalWebTM",
  "properties": {
    "trafficRoutingMethod": "Priority",
    "endpoints": [
      {
        "name": "WestEurope",
        "type": "azureEndpoints",
        "targetResourceId": "/subscriptions/.../publicIPAddresses/westvmss-lb-ip",
        "priority": 1,
        "endpointMonitorStatus": "Online"
      },
      {
        "name": "NorthEurope",
        "type": "azureEndpoints",
        "targetResourceId": "/subscriptions/.../publicIPAddresses/northvmss-lb-ip",
        "priority": 2,
        "endpointMonitorStatus": "Online"
      }
    ],
    "monitorConfig": {
      "protocol": "HTTP",
      "port": 80,
      "path": "/health",
      "intervalInSeconds": 10,
      "toleratedNumberOfFailures": 3,
      "timeoutInSeconds": 5
    }
  }
}
```

**Failover Scenario**:

```
Normal Operation:
Time: 10:00:00
- West Europe: 5 instances healthy
- North Europe: 3 instances healthy (standby)
- Traffic Manager: All traffic to West Europe
- User requests: routed to 20.50.60.70

West Europe Failure:
Time: 10:05:00
- West Europe datacenter issue
- Health probes start failing

Time: 10:05:10 - Probe 1 fails
Time: 10:05:20 - Probe 2 fails
Time: 10:05:30 - Probe 3 fails

Time: 10:05:30 - Failover triggered!
- Traffic Manager: Mark West Europe "Degraded"
- Traffic Manager: Route to North Europe
- DNS TTL: 60 seconds (clients gradually switch)

Time: 10:06:30 - Full failover complete
- All new connections: North Europe
- Existing connections: Gradually expire

West Europe Recovery:
Time: 10:30:00
- West Europe health restored
- Health probes succeeding

Time: 10:30:30 - Probe success count reaches threshold
- Traffic Manager: Mark West Europe "Online"
- Traffic Manager: Route back to West Europe (Priority 1)
- DNS updates propagate

Time: 10:31:30 - Back to normal
- Traffic: West Europe (primary)
- North Europe: Standby
```

### Example 5: Blue-Green Deployment with VMSS

**Strategy**: Zero-downtime updates using two VMSSs

```
Infrastructure:

Load Balancer (Shared)
├── Backend Pool: Blue (active)
│   └── VMSS-Blue: 5 instances (version 1.0)
│
└── Backend Pool: Green (inactive)
    └── VMSS-Green: 0 instances

Deployment Process:

Step 1: Deploy new version to Green
─────────────────────────────────────
- Scale VMSS-Green from 0 to 5
- Instances boot with version 2.0
- Health checks pass
- NOT in load balancer yet
Time: 5 minutes

Step 2: Test Green environment
─────────────────────────────────────
- Internal testing via direct IP
- Smoke tests pass
- Load testing
Time: 10 minutes

Step 3: Switch traffic (Blue → Green)
─────────────────────────────────────
- Load Balancer: Add Green to backend pool
- Both Blue and Green receiving traffic
- Monitor metrics
- No errors
Time: 2 minutes

Step 4: Remove Blue from rotation
─────────────────────────────────────
- Load Balancer: Remove Blue from backend pool
- All traffic to Green
- Monitor for 15 minutes
- Version 2.0 successful!

Step 5: Scale down Blue
─────────────────────────────────────
- VMSS-Blue: Scale from 5 to 1 (keep 1 for rollback)
- Green is now "Blue" (active)
- Old Blue is now "Green" (standby)

Rollback (if needed):
─────────────────────────────────────
If version 2.0 has issues:
- Load Balancer: Add Blue back (still has version 1.0)
- Load Balancer: Remove Green
- VMSS-Blue: Scale back to 5
- VMSS-Green: Scale to 0
Time to rollback: <2 minutes
```

**Implementation**:

```bash
#!/bin/bash

# Blue-Green deployment script

BLUE_VMSS="VMSS-Blue"
GREEN_VMSS="VMSS-Green"
LB_NAME="AppLoadBalancer"
BACKEND_POOL="AppBackendPool"

echo "Step 1: Deploy to Green"
az vmss scale --name $GREEN_VMSS --new-capacity 5
az vmss wait --name $GREEN_VMSS --custom "provisioningState=='Succeeded'"
echo "Green instances ready"

echo "Step 2: Wait for health checks"
sleep 120

echo "Step 3: Test Green environment"
# Run smoke tests here
./smoke-tests.sh green

if [ $? -eq 0 ]; then
  echo "Tests passed"
  
  echo "Step 4: Add Green to load balancer"
  az network lb address-pool add \
    --lb-name $LB_NAME \
    --name $BACKEND_POOL \
    --vmss $GREEN_VMSS
  
  echo "Step 5: Monitor traffic"
  sleep 300  # 5 min monitoring
  
  echo "Step 6: Remove Blue from load balancer"
  az network lb address-pool remove \
    --lb-name $LB_NAME \
    --name $BACKEND_POOL \
    --vmss $BLUE_VMSS
  
  echo "Step 7: Scale down Blue"
  az vmss scale --name $BLUE_VMSS --new-capacity 1
  
  echo "Deployment complete!"
else
  echo "Tests failed - rolling back"
  az vmss scale --name $GREEN_VMSS --new-capacity 0
  exit 1
fi
```

-----

## Summary: The Complete Picture

### What You Now Understand About VMSS

**Top Layer (Portal)**:

- Click, scale, configure
- Simple slider or autoscale rules
- Hides massive orchestration complexity

**Middle Layer (Components)**:

- VM profile (the template)
- Instances (individual VMs)
- Scaling policies (manual, metric-based, scheduled)
- Upgrade policies (automatic, rolling, manual)
- Health monitoring and auto-repair
- Load balancer integration

**Bottom Layer (Technology)**:

- Azure Fabric Controller (placement, orchestration)
- Hyper-V hypervisor (virtualization)
- Distributed across availability zones
- Physical servers across racks/datacenters
- RDMA storage networks
- Sophisticated monitoring and healing

**Key Orchestration Flows**:

- Scale-out: Find servers → allocate → create VMs → health check → add to LB
- Scale-in: Select instances → drain → delete → deallocate
- Update: Rolling batches → reimage/recreate → health check → next batch
- Repair: Detect unhealthy → grace period → delete/recreate → verify

### Key Insights

**VMSS is NOT**:

- ❌ Manual VM management
- ❌ Single point of failure
- ❌ Fixed capacity
- ❌ Limited to one region
- ❌ Only for stateless apps (Flexible mode supports stateful)

**VMSS IS**:

- ✅ Automated orchestration system
- ✅ Highly available by design
- ✅ Elastically scalable (2-1000 instances)
- ✅ Multi-region capable
- ✅ Self-healing with auto-repair
- ✅ Cost-optimized with scale-in
- ✅ Zero-downtime updates with rolling policy
- ✅ Integrated with Azure ecosystem (LB, Monitoring, etc.)

### How This Helps Your Technical Assessment

**When reviewing Azure architectures, you can now**:

1. Evaluate scalability design
- Is VMSS configured for expected load?
- Are min/max capacity appropriate?
- Is autoscaling based on right metrics?
1. Assess availability
- Instances spread across zones?
- Fault domain distribution correct?
- Auto-repair enabled?
1. Understand update strategy
- Rolling vs. automatic vs. manual?
- Health monitoring configured?
- Rollback capability?
1. Review cost optimization
- Right VM sizes chosen?
- Scale-in policies aggressive enough?
- Overprovisioning needed?
1. Verify operational practices
- Health probes configured?
- Notifications set up?
- Blue-green deployment possible?

**Questions you can answer**:

- “How does VMSS ensure high availability?”
  → Multi-zone deployment, auto-repair, FD distribution
- “What happens during a rolling update?”
  → Batch-by-batch reimaging with health checks
- “How quickly can you scale from 3 to 100 instances?”
  → ~5-10 minutes depending on VM size and overprovisioning
- “Can VMSS survive a datacenter failure?”
  → Yes, if deployed across multiple zones
- “How does autoscaling work?”
  → Metric evaluation → threshold triggers → orchestrator acts → cooldown

### Connection to Broader Cloud Concepts

**VMSS builds on**:

- Virtual Networks → Instances deployed into subnets
- Load Balancers → Backend pool management
- NSGs → Applied to each instance
- Managed Disks → OS/data disk storage
- Availability Zones → Physical distribution
- Azure Monitor → Metrics for autoscaling

**VMSS enables**:

- Microservices architecture → Scale services independently
- High availability → Multi-zone, auto-repair
- Cost efficiency → Scale down during low traffic
- DevOps practices → Blue-green, canary deployments
- Disaster recovery → Multi-region deployment

-----

## Next Steps

**Further Learning**:

1. **Hands-On Labs**:
- Deploy a basic VMSS
- Configure autoscaling
- Test auto-repair by killing instances
- Perform rolling update
- Implement blue-green deployment
1. **Advanced Topics**:
- VMSS with Azure Application Gateway
- VMSS with custom images (Packer)
- VMSS with managed identities
- VMSS with Azure DevOps CI/CD
- VMSS with spot instances (cost optimization)
1. **Troubleshooting**:
- Instance provisioning failures
- Health probe issues
- Autoscale not triggering
- Extension deployment failures
- Network connectivity problems
1. **Compare with Other Services**:
- VMSS vs. AKS (Kubernetes)
- VMSS vs. Azure Container Instances
- VMSS vs. Azure App Service
- VMSS vs. Individual VMs with Availability Sets

This document provides complete visibility into how Azure VMSS works from the user interface down to the physical hardware, empowering you to design, deploy, and troubleshoot VMSS-based solutions with confidence.

