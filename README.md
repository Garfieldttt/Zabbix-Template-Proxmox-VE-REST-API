# Zabbix Template Proxmox VE REST API

This Zabbix template enables full monitoring of a Proxmox VE environment via the official REST API (Proxmox VE ≥ 7.0). It collects host and container metrics, backup jobs, storage status, tasks, and user information, and automatically generates discovery rules for nodes, LXC containers, QEMU VMs, storage pools, running tasks, and API users.

## Requirements

- **Zabbix Server** version 7.0 or higher  
- **HTTP Agent** module enabled on the Zabbix server  
- Proxmox VE API token with read permissions for Nodes, Tasks, Storage, LXC, QEMU, and Access  
- Host macros defined on the Zabbix host object (see “Macros” section)

## Installation

1. Download the template `Template Proxmox VE REST API.yaml`.

2. In the Zabbix web interface, go to **Configuration → Templates → Import** and import the template.

3. Create a new host:
   - Go to **Configuration → Hosts → Create host**
   - Enter a **Host name** (e.g. `proxmox01`)
   - Assign the template **Template Proxmox VE REST API**
   - Set the appropriate **Group** (e.g. `Linux servers`)
   - Leave the **Interfaces** section empty (the template uses the API, not an agent)

4. Configure the required host macros (see the “Macros” section of the documentation).

## Macros

### Required Macros

| Macro                   | Example Value      | Description                                          |
|-------------------------|--------------------|------------------------------------------------------|
| `{$PVE_IP}`             | `192.168.1.1`   | IP address or hostname of the Proxmox VE API server  |
| `{$PVE_PORT}`           | `8006`             | TCP port of the Proxmox API (default: 8006)          |
| `{$PVE_NODE}`           | `pve`              | Identifier of the Proxmox node                       |
| `{$PVE_API_USER}`       | `root@pam`         | API username including realm                         |
| `{$PVE_API_TOKEN_ID}`   | `Zabbix`           | Name/ID of the API token                             |
| `{$PVE_API_TOKEN}`      | **SECRET_TEXT**    | API token (store as a secret macro on the host)      |

### Optional Trigger Macros

| Macro                             | Example Value | Description                                                             |
|-----------------------------------|---------------|-------------------------------------------------------------------------|
| `{$ENABLE_BACKUP_ALERT}`          | `1`           | 1 = enable backup trigger, 0 = disable                                 |
| `{$ENABLE_NODE_STATUS_ALERT}`     | `1`           | 1 = enable node offline trigger, 0 = disable                           |
| `{$ENABLE_STORAGE_AVAILABLE_ALERT}`   | `1`       | 1 = enable low-space trigger, 0 = disable                              |
| `{$ENABLE_STORAGE_INACTIVE_ALERT}`    | `1`       | 1 = enable inactive storage trigger, 0 = disable                       |
| `{$ENABLE_TASK_ALERT}`            | `1`           | 1 = enable task failure trigger, 0 = disable                           |
| `{$ENABLE_TASK_STATUS_ALERT}`     | `1`           | 1 = enable general task status trigger, 0 = disable                   |
| `{$ENABLE_VM_STOP_ALERT}`         | `1`           | 1 = enable VM/LXC stop trigger, 0 = disable                            |
| `{$USER_EXPIRE_TIME}`             | `2d`          | Lead time (in days) for user-expiry trigger warning                    |

## Contents of the Template

### 2. Discovery Rules

| Discovery Rule       | Description                                                 |
|----------------------|-------------------------------------------------------------|
| **discover.nodes**   | Automatic detection of all Proxmox nodes                    |
| **discover.lxc**     | Detection of all LXC containers on the host                 |
| **discover.qemu**    | Detection of all QEMU/KVM VMs with LLD macros               |
| **discover.storage** | Listing of all storage pools and their status               |
| **discover.backup**  | Grouping and formatting of VZDUMP backup jobs               |
| **discover.tasks**   | Monitoring of all running tasks (excluding VZDUMP)          |
| **discover.users**   | Detection of all users and their expiration dates           |

### 3. Trigger Prototypes

- Backup failure  
- Node offline  
- Low storage & inactive storage  
- Task failure  
- VM/LXC stopped  
- User expiration (warning at configured lead time)

## Usage

1. Create an API token on the Proxmox host.
3. Add or select the host in Zabbix.  
4. Assign the “Template Proxmox VE REST API” to the host.  
5. Configure macros on the host’s Template tab (API credentials, node name, etc.).  
6. Enable monitoring and check initial metrics under **Monitoring → Latest data**.

### Screenshots
<img width="2306" height="780" alt="image" src="https://github.com/user-attachments/assets/c480f488-ba91-4c7e-871a-4a10b992bb52" />
<img width="2322" height="825" alt="image" src="https://github.com/user-attachments/assets/dd9523df-9c54-4096-a2d5-9bd3915e8a14" />
<img width="2310" height="623" alt="image" src="https://github.com/user-attachments/assets/2f6b0a36-60c5-4e63-84dd-d09d8f011bfe" />



