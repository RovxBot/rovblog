# Homelab Configuration Night 1: Development Environment Setup

This document chronicles the first night of setting up a Kubernetes-based home lab, focusing on preparing the development environment on Windows using WSL and Nix.

!!! info "Session Overview"
    **Goal**: Set up a reproducible development environment for home lab deployment
    **Duration**: Evening session
    **Key Technologies**: WSL, Nix, Git, Terraform

## Windows Desktop Setup for Deployment

Setting up Windows Subsystem for Linux (WSL) provides a native Linux environment for running deployment scripts and tools that aren't available in PowerShell.

### WSL Installation and Configuration

**Install WSL (Windows Subsystem for Linux):**

```powershell
# Enable WSL feature and install default distribution
wsl --install
```

!!! tip "WSL Installation Notes"
    - Requires Windows 10 version 2004 or higher, or Windows 11
    - May require a system restart after installation
    - Enables both WSL and Virtual Machine Platform features

**Install Ubuntu 24.04 LTS:**
```powershell
# Install specific Ubuntu version for consistency
wsl --install Ubuntu-24.04
```

**Start Ubuntu 24.04 LTS:**
```powershell
# Launch the specific distribution
wsl -d Ubuntu-24.04
```

!!! warning "First Launch Setup"
    On first launch, Ubuntu will prompt you to:

    - Create a new user account
    - Set a password
    - Update the system packages

### Nix Package Manager Installation

**Install Determinate Nix for WSL:**
```bash
# Install Nix package manager with Determinate Systems installer
curl -fsSL https://install.determinate.systems/nix | sh -s -- install --determinate
```

!!! info "Why Nix?"
    Nix provides:

    - **Reproducible environments**: Exact same tools across different machines
    - **Declarative configuration**: Infrastructure as code approach
    - **Isolation**: No conflicts between different project dependencies
    - **Rollback capability**: Easy to revert to previous configurations

### Repository Setup and Project Initialisation

**Navigate to existing Git repository (if available):**
```bash
# Access Windows filesystem from WSL
cd /mnt/d/Git/homelab
```

**Set up Linux home directory structure:**
```bash
# Return to Linux home directory
cd ~

# Create organised directory structure
mkdir -p git
cd git
```

**Clone the home lab repository:**
```bash
# Clone the project repository
git clone https://github.com/RovxBot/homelab.git

# Navigate to project directory
cd homelab
```

!!! tip "Directory Structure Best Practices"
    **Windows Integration:**
    - `/mnt/c/` - Access Windows C: drive
    - `/mnt/d/` - Access Windows D: drive
    - Keep Linux projects in `~/git/` for better performance

    **Git Workflow:**
    - Use SSH keys for authentication
    - Configure Git user settings in WSL
    - Consider using Git credential manager for Windows integration
### Development Environment Activation

**Enter Nix development shell:**
```bash
# Activate the development environment with all required tools
nix develop
```

!!! info "Nix Development Shell"
    The `nix develop` command creates an isolated environment with:

    - Kubernetes tools (kubectl, helm, etc.)
    - Terraform for infrastructure provisioning
    - Ansible for configuration management
    - Development utilities and dependencies

**Run initial configuration:**
```bash
# Execute the configuration setup
make configure
```

## Configuration Parameters

The configuration process prompts for several key parameters that define the home lab environment:

### Basic Settings
- **Text editor**: `nano` (lightweight, beginner-friendly)
- **Domain**: `cooked.beer` (custom domain for services)
- **Seed repository**: `https://github.com/RovxBot/homelab`
- **Timezone**: `Australia/Adelaide` (ACST/ACDT)
- **Load balancer IP range**: `10.0.0.0/24` (internal service IPs)

!!! tip "Configuration Choices Explained"
    - **Domain selection**: Choose a domain you control for SSL certificates and DNS
    - **IP range**: Use RFC 1918 private ranges that don't conflict with your LAN
    - **Timezone**: Critical for log timestamps and scheduled tasks
    - **Editor**: Can be changed later, nano is good for beginners


### Server Infrastructure Configuration

**Ansible inventory configuration:**
```yaml
all:
  vars:
    control_plane_endpoint: 192.168.1.180  # Kubernetes API server endpoint
    load_balancer_ip_pool:
      - 10.0.0.0/24                        # MetalLB IP pool for services
metal:
  children:
    masters:                               # Kubernetes control plane nodes
      hosts:
        metal0:
          ansible_host: 192.168.1.190
          mac: 'D8:9E:F3:90:E8:31'
          disk: nvme0n1
          network_interface: enp0s31f6
        metal1:
          ansible_host: 192.168.1.191
          mac: 'D8:9E:F3:10:E8:A8'
          disk: nvme0n1
          network_interface: enp0s31f6
    workers:                               # Kubernetes worker nodes
      hosts:
        # Additional worker nodes to be added
```

!!! note "Infrastructure Design"
    **Control Plane Setup:**
    - **High Availability**: Multiple master nodes for redundancy
    - **Load Balancer**: MetalLB provides LoadBalancer services in bare metal
    - **Network**: Dell OptiPlex machines with consistent hardware

    **Hardware Specifications:**
    - **Network Interface**: `enp0s31f6` (Dell-specific naming)
    - **Storage**: NVMe SSDs (`nvme0n1`) for performance
    - **MAC Addresses**: Required for Wake-on-LAN functionality

### Terraform Cloud Integration

**Terraform Workspace Configuration:**
- **Workspace ID**: `ws-yM6CtwT9NWtgTug3`
- **Purpose**: Remote state management and collaborative infrastructure changes

!!! warning "Terraform Workspace Setup"
    The workspace ID configuration may need verification. Terraform Cloud workspaces provide:

    - **Remote state storage**: Prevents state file conflicts
    - **Collaborative workflows**: Team-based infrastructure management
    - **Variable management**: Secure storage of sensitive configuration
    - **Plan/apply automation**: CI/CD integration capabilities

### Version Control and Authentication

**Commit and push changes:**
```bash
# Stage all configuration changes
git add .

# Commit with descriptive message
git commit -m "Initial home lab configuration - Night 1"

# Push to remote repository
git push origin main
```

**GitHub Authentication:**
```bash
# Configure Git with your details
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Use Personal Access Token for authentication
# GitHub username/password authentication is deprecated
```

!!! tip "GitHub Token Setup"
    **Creating a Personal Access Token:**

    1. Go to GitHub Settings > Developer settings > Personal access tokens
    2. Generate new token with appropriate scopes:
       - `repo` - Full repository access
       - `workflow` - GitHub Actions access (if using CI/CD)
    3. Store token securely (password manager recommended)
    4. Use token as password when prompted by Git

## Session Summary and Next Steps

### What Was Accomplished Tonight

**Development Environment Setup:**
- WSL installation and Ubuntu 24.04 LTS configuration
- Nix package manager installation for reproducible environments
- Project repository cloning and initial setup

**Configuration Baseline:**
- Basic home lab parameters defined
- Ansible inventory structure created
- Terraform workspace integration configured

**Version Control:**
- Git repository initialised with configuration
- GitHub authentication configured with personal access tokens

### Lessons Learned

!!! success "Key Insights"
    - **WSL Integration**: Provides excellent Linux tooling on Windows
    - **Nix Benefits**: Reproducible environments eliminate "works on my machine" issues
    - **Configuration Management**: Declarative approach makes infrastructure predictable
    - **Documentation**: Real-time documentation captures decision rationale

### Tomorrow Night's Agenda

**Hardware Preparation Tasks:**
1. **Physical Setup**: Collect and prepare Dell OptiPlex 7050 mini PCs
2. **BIOS Configuration**:
   - Enable Wake-on-LAN functionality
   - Disable Secure Boot for Linux installation
   - Configure boot priorities
3. **Hardware Discovery**:
   - Collect actual MAC addresses from each unit
   - Identify network interface names
   - Verify storage device naming conventions
4. **Power Infrastructure**:
   - Install additional power supplies when they arrive
   - Plan rack/shelf organisation for mini PCs

**Technical Preparation:**
- Update Ansible inventory with real hardware details
- Prepare Ubuntu Server installation media
- Plan network addressing scheme
- Document BIOS settings for consistency

!!! tip "Hardware Preparation Tips"
    - **Label everything**: Physical labels help track which unit is which
    - **Document BIOS settings**: Take photos of important configuration screens
    - **Test Wake-on-LAN**: Verify functionality before final deployment
    - **Plan cable management**: Consider network and power cable routing

### Future Considerations

**Expansion Planning:**
- Additional mini PCs pending power supply delivery
- Scalability considerations for Kubernetes cluster growth
- Storage expansion options (NAS integration)
- Monitoring and observability stack planning

**Security Considerations:**
- Network segmentation planning
- Certificate management strategy
- Backup and disaster recovery procedures
- Access control and authentication methods
