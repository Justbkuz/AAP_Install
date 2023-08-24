# Production AAP Installation / Upgrade

## Requirements (Per Server)

### Resource requirements
- All
  - OS RHEL 8.6 or Later
  - Python 3.8
  - DNS Resolution of each host
- Automation Controller x3
  - VMsizing
    - CPUs 4
    - RAM 16
    - min 40GB /var/lib/awx for execution environment storage on hyrbid nodes  
      - Projects are stored on control and hybrid nodes, and for the duration of jobs, are also stored on execution nodes. If the cluster has many large projects, consider doubling the GB in /var/lib/awx/projects, to avoid disk space errors
    - min 40gb OS
- Automation Hub
  - Ensure that the following domain names are part of either the firewall or the proxy’s allowlist for successful connection and download of collections from automation hub or Galaxy server:
    - galaxy.ansible.com
    - registry.redhat.com
    - console.redhat.com
  - VM Sizing
    - CPUs 2
    - RAM 8gb
    - min 40gb OS
    - min 60gb /var
      - pulp will reside under /var and hold the collections and execution environments
- Database
  - PostgresSQL 13
  - VM Sizing
    - CPUs 4
    - RAM 16gb
    - min 40GB OS
    - min 220gb /var
      - 40gb per controller
      - 40gb per eda
      - 60gb per hub
- Event Driven Automation
  - VM Sizing
    - CPU 4
    - RAM 16gb
    - min 40gb OS
    - min 40gb /var

### Licensing requirements

- Redhat Subscription attached that include Ansible dependencies
- Generally, this is labled "Ansible Automation Platform XXX"
  - ```subscription-manager list --available --all | grep "Ansible Automation Platform" -B 3 -A 6```
    - Do not use MCT4022 as a pool_id for your subscription because it can cause Ansible Automation Platform subscription attachment to fail.
- see the following url for OS level licensing
  - https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.4/html/red_hat_ansible_automation_platform_planning_guide/proc-attaching-subscriptions_planning#doc-wrapper
- The controllers will get licensed by a manifest or by signing after installation and giving credentials to redhat 

## Authentication

- SSH Key Authentication Preferred.
- Need sudo permissions


## TLS/SSL Certificate
If the customer wants to use issued certificates, We recommend using a SAN certificate to cover all the services at once that are being installed. 

Create an OpenSSL configuration file holding the customers' information using this template as a guide:
```
[ req ] 
default_bits        = 4096 
distinguished_name  = req_distinguished_name 
req_extensions      = req_ext 
prompt              = no 
[ req_distinguished_name ] 
countryName         = <2 digit country code> 
stateOrProvinceName = <state> 
localityName        = <city> 
organizationName    = <organization> 
# NOTE: Not all certificate authorities like the CN to not be a FQDN. I find 
#   it fitting to not be a FQDN because which would you pick? The spec for 
#   x509 also makes it very plain this isn't a requirement. Regardless I've 
#   encountered customers that require it to be set to something else. 
commonName          = Ansible Automation Platform 
[ req_ext ] 
subjectAltName      = @alt_names 
[ alt_names ] 
# NOTE: Add entries for each service being deployed that will also be 
#   assigned a name that is meant to specify the service and not a specific 
#   node.  This is highly recommended as it forces the customer to pre- 
#   allocate that bit of configuration, reducing the need to perform re-work 
#   later.  This is important particularly for Hub and execution environments 
#   in controller as they are linked to the FQDN of hub. 
DNS.1               = ans-controller-lb.tld 
DNS.2               = ans-eda-lb.tld 
DNS.3               = ans-hub-lb.tld  
# NOTE: Add entries of the FQDN for each server being built 
DNS.4               = controller1.tld
DNS.5               = controller2.tld
DNS.6               = controller3.tld
DNS.7               = database1.tld 
DNS.8               = eda1.tld 
DNS.9               = hub1.tld 
# NOTE: Add entries of the hostname of each server being built. I've 
#   encountered CA's that do not allow this. If so, just remove it. The 
#   result will be a certificate error if someone browses directly to 
#   the hostname. 
DNS.11              = controller1
DNS.12              = controller2
DNS.13              = controller3
DNS.14              = database1 
DNS.15              = eda1 
DNS.16              = hub1  
# NOTE: Add entries containing the IP of each server being built. I've 
#   encountered CA's that do not allow this. If so, just remove it. The 
#   result will be a certificate error if someone browses directly to 
#   the hostname. 
IP.1                = 2001:db8::11 
IP.2                = 192.0.2.11 
IP.3                = 2001:db8::12 
IP.4                = 192.0.2.12 
IP.5                = 2001:db8::13 
IP.6                = 192.0.2.13 
IP.7                = 2001:db8::14 
IP.8                = 192.0.2.14 
IP.9                = 2001:db8::15 
IP.10               = 192.0.2.15 
```
Generate the CSR with the openssl command line tool. If a new private key is required that can be generated at the same time as well with a command like: 
```
openssl req -new -nodes -newkey rsa:4096 -reqexts req_ext -keyout san.key -config tls.conf -out san.csr  
```

# Diagram
```mermaid

flowchart TD;
    subgraph 1[internal_systems]
    a(Automation_Target_Systems)
    b(Monitoring_Systems)
    end
    2--"playbook execution"--> a
    b--> 6
    subgraph 2[Load_Balancer_URL]
    subgraph 3[Ansible_Automation_Controllers]
    c(ctrlhostname1)
    d(ctlrhostname2)
    e(ctrlhostname3)
    end
    end
    f<-.->|Exec_Env/Collections|2
    subgraph 4[AAP Components]
    subgraph 5[Automation_Hub]
    f(hubhostname)
    end
    subgraph 6[Event_Driven_Automation]
    g(edahostname)
    end
    end
    
    6-.->|Event Driven Templates|2
    subgraph 7[Database]
    h(dbhostname)
    end
    2 -->|table=awx|7
    5-->|table=automationhub|7
    6-->|table=eventdriven|7
    

```

# Upgrade Process
- Pre-Work
  - Create compatible Ansible 2.9.26 Execution Environment with Collections/pip requirements


1. Standup New Rhel 8.6+ Servers for Env
2. get ansible-automation-platform-setup-<latest_version>.tar.gz
3. install AAP 1.2 3.8.5 (new) https://releases.ansible.com/ansible-tower/setup/ansible-tower-setup-3.8.5-3.tar.gz
4. Note enabled job schedules in original
5. stop all scheduled jobs in original
6. Validate Enough Space to take a backup
7. take backup of environment
   1. ```./setup.sh -e 'backup_dest=/path/to/backup_dir/' -b``
8.  enable job schedules in original (if required)
9. restore backup to (new) aap 1.2
10. validate restore
11. get new ansible-automation-platform-setup-bundle-2.4-1.2-x86_64.tar.gz
12. configure inventory for (new) and add additional components
13. run setup.sh on (new) to upgrade
14. validate upgrade