# The Enterprise High-Availability Web Architecture Blueprint

**Environment:** Microsoft Azure

**Deployment Method:** Terraform (Infrastructure as Code)

**Objective:** Deploy a fault-tolerant, auto-scaling web application utilizing Layer 7 load balancing and automated configuration management.

**_Repo_:** [labs/Cloud/Azure/AZ104-HA-Web-Architecture at main · Arthur-K-99/labs](https://github.com/Arthur-K-99/labs/tree/main/Cloud/Azure/AZ104-HA-Web-Architecture)

## Introduction: The Well-Architected Framework

In modern enterprise environments, deploying a single virtual machine is no longer sufficient. Cloud architecture must align with the Microsoft Azure Well-Architected Framework, specifically focusing on **Reliability**, **Performance Efficiency**, and **Operational Excellence**. This blueprint transitions a traditional monolithic server setup into a stateless, dynamically scaling fleet capable of surviving localized data center hardware failures, mitigating traffic spikes, and minimizing human intervention through Infrastructure as Code (IaC).

## Phase 1: The Network Fabric (Layer 2/3 Foundation)

Before spinning up any compute resources, the network boundary must be established. Coming from a routing and switching background, building this in the cloud is essentially the software-defined equivalent of racking a core switch and carving out your VLANs.

- **The Virtual Network (VNet):** We deployed `vnet-az104ha-dev-eastus-001` with a `10.0.0.0/16` address space. This VNet acts as your isolated, private data center boundary within the `eastus` Azure region. By default, Azure handles the implicit routing (System Routes); any subnet created inside this /16 can inherently talk to any other subnet without needing a dedicated virtual router or static route table (User Defined Route - UDR). This `/16` gives us 65,536 theoretical IP addresses, leaving massive room for future enterprise expansion like peering with on-premises networks via ExpressRoute or VPN Gateways.
    
- **Subnet Segmentation Strategy:** We segmented the VNet into two distinct subnets, adhering to the principle of least privilege and micro-segmentation:
    
    - `snet-vmss-001 (10.0.2.0/24)`: The compute subnet where the actual web servers reside. A `/24` allows for 251 usable IPs (Azure reserves the first 3 and last 1 IPs in every subnet for protocol operations).
        
    - `snet-appgw-001 (10.0.1.0/24)`: The dedicated load balancer subnet.
        
- **The Application Gateway Subnet Requirement (AZ-104 Crucial):** Why did we have to create a completely separate subnet just for the App Gateway? Microsoft explicitly requires this. An Application Gateway isn’t just a simple software appliance; it is a managed cluster of underlying dynamic worker nodes operating under the hood. Azure needs an entire dedicated subnet so it can automatically consume and release private IP addresses for these invisible nodes as the load balancer scales its own internal capacity to handle heavy traffic spikes. _You cannot place any other resources (like VMs or databases) inside an App Gateway subnet._
    
    - _Network Security Group (NSG) Note:_ The App Gateway requires continuous communication with Azure's internal control plane to report health and scaling metrics. If you apply an NSG to the Application Gateway subnet in the future, you **must** allow inbound traffic on TCP ports `65200-65535` from the `GatewayManager` service tag, or the load balancer will enter a failed state.
        

## Phase 2: Traffic Ingress & Load Balancing (Layer 7)

Standard Azure Load Balancers operate at Layer 4 (Transport) of the OSI model, meaning they blindly forward TCP/UDP packets based purely on source and destination IP addresses and ports. While fast, they are unintelligent. We deployed an **Application Gateway**, which operates at Layer 7 (Application).

- **The Frontend Public IP:** `pip-appgw-001` acts as the single point of entry from the internet. Because we attached it to a Standard v2 Application Gateway, Azure requires this to be a strictly "Static" IP allocation (Dynamic is not supported on v2). This ensures DNS records pointed at this architecture will never break due to an IP change.
    
- **The Routing Engine & Layer 7 Intelligence:**
    
    - **The Listener:** We configured an HTTP listener to actively catch any web traffic hitting Port 80 on that public IP. In a production environment, this would be an HTTPS listener terminating port 443 traffic (SSL Offloading), thereby freeing up the backend VMs from the CPU-intensive task of decrypting traffic.
        
    - **The Routing Rule:** Once caught, the routing rule acts as the logic engine. Because it operates at Layer 7, it can actually "read" the HTTP request (the URI, headers, and payload). While we used a basic rule to forward everything to one pool, this Layer 7 capability is what allows enterprises to route traffic based on URLs—e.g., sending `contoso.com/images` to a storage pool, and `contoso.com/api` to a high-performance compute pool.
        
    - **Backend HTTP Settings:** We explicitly disabled "Cookie-based affinity". If left on, the App Gateway would inject a hash cookie into the user's browser, pinning their session to a specific backend server. Disabling it forces the gateway to use true round-robin distribution, evenly distributing every single refresh across the fleet. This is mandatory for truly stateless web applications.
        
    - **Health Probes:** Under the hood, the App Gateway continually pings the backend pool. If a server stops responding with an HTTP `200 OK` status, the gateway automatically removes it from the rotation, preventing users from seeing dead pages.
        
- **The TLS/SSL Security Policy (The Error):** During the Terraform deployment, the API threw a `400 Bad Request` regarding a deprecated TLS version. Microsoft recently enforced a ban on older, insecure encryption standards (TLS 1.0/1.1) for all new Application Gateways to comply with modern security frameworks like PCI-DSS and HIPAA. Because the Terraform provider still attempts to use a legacy 2015 default policy, we explicitly injected `AppGwSslPolicy20220101` to force strict, modern TLS 1.2+ compliance.
    

## Phase 3: Elastic Compute (The Virtual Machine Scale Set)

Instead of deploying individual, static virtual machines that require manual patching and monitoring, we deployed a **Virtual Machine Scale Set (VMSS)** using the `Standard_B2s` SKU.

- **The B-Series Hypervisor (CPU Credits):** B-Series VMs are "burstable." They operate on a token-bucket algorithm: they accumulate CPU credits while idling below their baseline performance threshold and consume those banked credits when under heavy load (like our `stress` test). If they run out of credits, Azure forcefully throttles their CPU to the baseline level (often 20% or 40%). This is perfect and cost-effective for cheap lab environments, but highly dangerous for heavy production web servers that require sustained performance. Production environments should use compute-optimized (F-series) or general-purpose (D-series) SKUs.
    
- **Uniform vs. Flexible Orchestration:** This orchestration mode difference was the exact cause of the missing “Run command” button during our troubleshooting phase.
    
    - When clicking through the portal, Azure defaults to **Flexible** mode. Flexible mode treats the scale set as a logical grouping of standard, highly manageable individual VMs. It allows you to mix and match VM sizes and spot instances within the same pool.
        
    - Terraform’s `azurerm_linux_virtual_machine_scale_set` defaults to **Uniform** mode. Uniform mode enforces strict, mathematically identical clones. It strips away individual VM management tools (like the Run command) because it dictates that you manage the infrastructure at the fleet level, not the node level. This is the gold standard for stateless, heavily scaled web architectures.
        
- **Upgrade Policy:** In our Terraform code, we set `upgrade_mode = "Manual"`. This protects the environment from accidental downtime. If we change the underlying OS image or script, Azure will _not_ automatically reboot running servers and disrupt active users. We must explicitly trigger an update (or rolling upgrade) to phase in the new changes.
    
- **Availability & Spreading (Fault Domains vs. Update Domains):** By leaving the advanced setting on “Max spreading”, we instructed Azure to physically distribute the underlying VMs across as many different physical hardware racks as possible within the data center.
    
    - _Fault Domains:_ Represent physical racks (shared power supply, shared top-of-rack switch). Spreading across Fault Domains ensures a single localized power failure doesn't drop your entire application.
        
    - _Update Domains:_ Logical groupings Azure uses when pushing host-level updates. Spreading across Update Domains ensures Azure won't reboot all your VMs simultaneously for maintenance.
        

## Phase 4: Automated Configuration Management (Bootstrapping)

The defining feature of cloud infrastructure is that servers should be ephemeral (easily replaced, destroyed, and rebuilt automatically). We achieved this using a bootstrap script.

- **The `cloud-init` Race Condition:** In our initial deployment, NGINX installed very quickly, but OS patches (`apt-get upgrade`) took several minutes. The moment the NGINX daemon started running, the Application Gateway’s health probe pinged port 80, received a successful response, and instantly started sending active internet browser traffic to that server. However, because the `cloud-init` script was still processing the OS upgrades in the background, your custom `index.html` `runcmd` hadn’t executed yet, resulting in the default Debian landing page being served to the internet.
    
- **Troubleshooting Uniform VMSS:** Because the portal "Run command" was disabled (due to Uniform mode), we diagnosed this directly on the instance by SSHing/executing bash commands against the underlying node to read the system logs: `tail -n 30 /var/log/cloud-init-output.log`. This revealed the YAML syntax failure.
    
- **The Pure Bash Solution:** To bypass the YAML parser limitations and the phased execution of `cloud-init` modules, we utilized a raw bash script starting with `#!/bin/bash` in the Terraform `custom_data` block.
    
    ```
    #!/bin/bash
    apt-get update
    apt-get install -y nginx stress
    rm -f /var/www/html/index.nginx-debian.html
    echo "<h1>Hello from Azure VMSS Instance: $(hostname)</h1>" > /var/www/html/index.html
    systemctl restart nginx
    ```
    
    This script bypasses the native `cloud-config` phases, running straight through the hypervisor as the `root` user to forcefully delete the default file, inject the dynamic `$HOSTNAME` variable, and restart the service in one linear sweep.
    
- **Real-World Evolution (Immutable Infrastructure):** In an enterprise production setting, running an `apt-get upgrade` on boot is discouraged because it introduces varying boot times and the risk of a broken package taking down a new instance. Instead, engineers use tools like **HashiCorp Packer** to pre-bake the OS, patches, and NGINX into a custom "Golden Image". Terraform would then deploy that Golden Image, resulting in a VM that boots in seconds with zero race conditions.
    

## Phase 5: Auto-Scaling Logic (Azure Monitor)

We configured Azure Monitor to watch the compute fleet and make scaling decisions dynamically based on real-time metrics, completely removing human intervention from the capacity planning process.

- **The Aggregation Trap:** When we used the `stress` tool on a single VM, the auto-scaler ignored it. Azure Monitor scales based on the **aggregate average** of the entire target pool. One VM at 100% and one at 1% yields a 50% fleet average. You must redline the majority of the fleet to cross the threshold and trigger an expansion.
    
- **The Rules & Thresholds:**
    
    - **Scale-Out:** When the average CPU exceeds 75% for 5 consecutive minutes, add 1 instance. (Ensuring the system can handle traffic spikes).
        
    - **Scale-In:** When the average CPU drops below 25% for 5 consecutive minutes, remove 1 instance. (Ensuring we stop paying for idle compute).
        
- **The Cooldown Window (Flap Prevention):** The 5-minute cool-down window is a critical architectural concept called "Flap Prevention". It prevents the infrastructure from aggressively spinning servers up and down (flapping) due to temporary, 10-second traffic spikes or immediate drops in CPU after a new instance is added. Flapping causes severe backend instability, broken sessions, and billing anomalies. By forcing Azure to wait 5 minutes before making another scaling decision, the system stabilizes.
    

## Phase 6: Infrastructure as Code (Terraform)

Transitioning from the Azure Portal's "ClickOps" to HashiCorp Terraform revealed how APIs truly manage infrastructure deployments, dependency mapping, and configuration state.

- **Handling Entra ID MFA (`AADSTS50076`):** The initial `az login` failed because your organization's Conditional Access policies required Multi-Factor Authentication, and the CLI failed to pass the correct tenant context back to the browser. Appending the specific `--tenant <TENANT_ID>` perfectly routed the authentication request to the correct identity provider, allowing the CLI to capture the OAuth token.
    
- **Targeted Rebuilding (`-replace`):** When we updated the bootstrap script in our code, the VMs didn't update automatically upon running `terraform apply` due to our "Manual" upgrade mode. Instead of destroying the entire VNet and App Gateway (which takes 15+ minutes), we used a targeted command: `terraform apply -replace="azurerm_linux_virtual_machine_scale_set.vmss"`. This instructed Terraform to surgically destroy and recreate _only_ the compute fleet, saving massive amounts of time.
    
- **Implicit Dependency Mapping:** In the `main.tf` file, we linked the VMSS to the load balancer using this line:
    
    ```
    application_gateway_backend_address_pool_ids = [for pool in azurerm_application_gateway.appgw.backend_address_pool : pool.id if pool.name == "bpool-vmss-001"]
    ```
    
    This declarative `for` loop dynamically searches the Application Gateway object for a specific backend pool name. Because the scale set _requires_ data that can only be generated after the Application Gateway exists, Terraform intelligently builds an implicit dependency graph. It guarantees the App Gateway is 100% provisioned before it even attempts to start building the web servers.
    
- **State Management & Version Control Security:** Terraform tracks the live status of Azure resources in a JSON-formatted `.tfstate` file.
    
    - **CRITICAL SECURITY WARNING:** This file must **never** be committed to Git. The state file stores your architecture secrets, admin passwords (like our VM `admin_password`), and API keys in plain text.
        
    - A proper `.gitignore` file must be used to exclude local `.terraform/` directories and `*.tfstate` files.
        
    - Conversely, you **must** commit the `.terraform.lock.hcl` file. This ensures that if another team member clones your repository six months from now, Terraform will download the exact same provider plugin versions you used, preventing breaking API changes from destroying your environment.
        
    - _Enterprise Upgrade:_ In a real-world scenario, you would configure a "Remote Backend," storing this `.tfstate` file inside a securely encrypted Azure Storage Account Blob. This allows multiple engineers to collaborate safely using state locks (preventing two people from applying changes at the exact same time).