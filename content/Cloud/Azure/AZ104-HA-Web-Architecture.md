# The Enterprise High-Availability Web Architecture Blueprint

**Environment:** Microsoft Azure 
**Deployment Method:** Terraform (Infrastructure as Code) 
**Objective:** Deploy a fault-tolerant, auto-scaling web application utilizing Layer 7 load balancing and automated configuration management.

*Repo*: [labs/Cloud/Azure/AZ104-HA-Web-Architecture at main · Arthur-K-99/labs](https://github.com/Arthur-K-99/labs/tree/main/Cloud/Azure/AZ104-HA-Web-Architecture)

---

## Phase 1: The Network Fabric (Layer 2/3 Foundation)

Before spinning up any compute resources, the network boundary must be established. Coming from a routing and switching background, building this in the cloud is essentially the software-defined equivalent of racking a core switch and carving out your VLANs.

- **The Virtual Network (VNet):** We deployed `vnet-az104ha-dev-eastus-001` with a `10.0.0.0/16` address space. This VNet acts as your isolated, private data center boundary within the `eastus` Azure region. By default, Azure handles the implicit routing; any subnet created inside this /16 can inherently talk to any other subnet without needing a dedicated router.
    
- **Subnet Segmentation Strategy:** We segmented the VNet into two distinct subnets:
    
    - `snet-vmss-001 (10.0.2.0/24)`: The compute subnet where the actual web servers reside.
        
    - `snet-appgw-001 (10.0.1.0/24)`: The dedicated load balancer subnet.
        
- **The Application Gateway Subnet Requirement:** Why did we have to create a completely separate subnet just for the App Gateway? Microsoft explicitly requires this. An Application Gateway isn't just a simple software firewall; it is a cluster of underlying dynamic worker nodes. Azure needs an entire dedicated subnet so it can automatically consume and release private IP addresses for these invisible nodes as the load balancer scales its own internal capacity to handle traffic spikes.


## Phase 2: Traffic Ingress & Load Balancing (Layer 7)

Standard Azure Load Balancers operate at Layer 4 (Transport) of the OSI model, meaning they blindly forward TCP/UDP packets based purely on source and destination IP addresses. We deployed an **Application Gateway**, which operates at Layer 7 (Application).

- **The Frontend Public IP:** `pip-appgw-001` acts as the single point of entry from the internet. Because we attached it to a Standard v2 Application Gateway, Azure requires this to be a strictly "Static" IP allocation.
    
- **The Routing Engine:**
    
    - **The Listener:** We configured an HTTP listener to actively catch any web traffic hitting Port 80 on that public IP.
        
    - **The Routing Rule:** Once caught, the routing rule acts as the logic engine. Because it operates at Layer 7, it can actually "read" the HTTP request. While we just used a basic rule to forward everything to one pool, this Layer 7 capability is what allows enterprises to route traffic based on URLs (e.g., sending `contoso.com/images` to a completely different set of servers than `contoso.com/api`).
        
    - **Backend HTTP Settings:** We explicitly disabled "Cookie-based affinity". If left on, the App Gateway would pin a user's session to a specific server. Disabling it forces the gateway to use true round-robin distribution, distributing every single refresh click evenly across the fleet.
        
- **The TLS/SSL Security Policy (The Error):** During the Terraform deployment, the API threw a `400 Bad Request` regarding a deprecated TLS version. Microsoft recently enforced a ban on older, insecure encryption standards (TLS 1.0/1.1) for all new Application Gateways. Because the Terraform provider still attempts to use a legacy 2015 default policy, we explicitly injected `AppGwSslPolicy20220101` to force modern TLS 1.2+ compliance.
    

## Phase 3: Elastic Compute (The Virtual Machine Scale Set)

Instead of deploying individual, static virtual machines, we deployed a **Virtual Machine Scale Set (VMSS)**.

- **Uniform vs. Flexible Orchestration:** This was the cause of the missing "Run command" button.
    
    - When clicking through the portal, Azure defaults to **Flexible** mode, treating the scale set as a collection of standard, highly manageable individual VMs.
        
    - Terraform's `azurerm_linux_virtual_machine_scale_set` defaults to **Uniform** mode. Uniform mode enforces strict, identical clones and strips away individual VM management tools (like the Run command) because it expects you to manage the infrastructure at the fleet level, not the node level.
        
- **Availability & Spreading:** By leaving the advanced setting on "Max spreading", we instructed Azure to physically distribute the underlying VMs across as many different physical hardware racks (Fault Domains) as possible within the data center. If a top-of-rack switch fails, only one instance goes down, and the App Gateway instantly reroutes traffic to the surviving instances.
    

## Phase 4: Automated Configuration Management (Bootstrapping)

The defining feature of cloud infrastructure is that servers should be ephemeral (easily replaced). We achieved this using a bootstrap script.

- **The `cloud-init` Race Condition:** NGINX installs very quickly, but OS patches (`apt-get upgrade`) take time. The moment NGINX started running, the Application Gateway's health probe pinged port 80, received a response, and instantly started sending your browser to that server. However, because the `cloud-init` script was still processing the OS upgrades in the background, your custom `index.html` command hadn't executed yet, resulting in the default Debian landing page being served to the internet.
    
- **The Pure Bash Solution:** To bypass the YAML parser limitations in Terraform's Uniform mode, we utilized a raw bash script starting with `#!/bin/bash`.
    
    Bash
    
    ```
    apt-get update
    apt-get install -y nginx stress
    rm -f /var/www/html/index.nginx-debian.html
    echo "<h1>Hello from Azure VMSS Instance: $(hostname)</h1>" > /var/www/html/index.html
    systemctl restart nginx
    ```
    
    This bypasses the phased `cloud-config` modules, running straight through the hypervisor as the root user to forcefully delete the default file, inject the dynamic `$HOSTNAME` variable, and restart the service.
    

## Phase 5: Auto-Scaling Logic (Azure Monitor)

We configured Azure Monitor to watch the compute fleet and make scaling decisions dynamically without human intervention.

- **The Aggregation Trap:** When we stress-tested a single VM, the auto-scaler ignored it. Azure Monitor scales based on the _aggregate average_ of the entire target pool. One VM at 100% and one at 1% yields a 50% average. You must redline the majority of the fleet to trigger an expansion.
    
- **The Rules:**
    
    - **Scale-Out:** When average CPU exceeds 75% for 5 consecutive minutes, add 1 instance.
        
    - **Scale-In:** When average CPU drops below 25% for 5 consecutive minutes, remove 1 instance.
        
- The 5-minute cool-down window is critical; it prevents the infrastructure from aggressively spinning servers up and down due to temporary, 10-second traffic spikes.
    

## Phase 6: Infrastructure as Code (Terraform)

Transitioning from the portal to Terraform revealed how APIs manage infrastructure deployments.

- **Handling Entra ID MFA (`AADSTS50076`):** The initial `az login` failed because Conditional Access policies required Multi-Factor Authentication, and the CLI failed to pass the correct tenant context to the browser. Appending the specific `--tenant` ID perfectly routed the authentication request to the correct identity provider.
    
- **Implicit Dependency:** In the `main.tf` file, we linked the VMSS to the load balancer using this line: `application_gateway_backend_address_pool_ids = [for pool in azurerm_application_gateway.appgw.backend_address_pool : pool.id if pool.name == "bpool-vmss-001"]` This `for` loop dynamically searches the Application Gateway for a specific backend pool name. Because it requires data from the Application Gateway, Terraform intelligently builds an implicit dependency graph, guaranteeing the App Gateway is fully provisioned before it attempts to build the web servers.
    
- **State Management:** When the deployment initially failed on the App Gateway TLS error, running `terraform apply` a second time did not rebuild the VNet or Subnets. Terraform reads the `.tfstate` file, compares it to the live Azure environment, and strictly executes the delta (the missing App Gateway).