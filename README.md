# Ansible Docker Installation Automation

This repository contains Ansible playbooks to automate Docker installation on Ubuntu VMs. The playbook creates a dedicated ansible user, installs Docker CE with Docker Compose, and copies a Dockerfile to target machines.

## Enhanced Features

This playbook includes advanced Ansible features:
- **When Conditions** - Conditional task execution
- **Handlers** - Event-driven service management  
- **Roles Support** - Modular playbook organization

## Project Structure

### Roles-Based Structure
```
ansible-docker-project/
├── README.md
├── host.ini
├── Dockerfile
└── roles/
    ├── user_management/
    │   ├── tasks/main.yml         # User creation tasks
    │   └── handlers/main.yml      # User-related handlers
    └── docker_installation/
        ├── tasks/main.yml         # Docker installation tasks
        └── handlers/main.yml      # Docker service handlers
```

## When Conditions

### Usage in Playbook

**Variable-Based Conditions:**
```yaml
when: create_ansible_user | bool    # Only run if variable is true
when: install_docker | bool         # Only run if variable is true
```

**OS Detection:**
```yaml
when: ansible_os_family == "Debian"  # Only run on Ubuntu/Debian systems
```

**Multiple Conditions (AND):**
```yaml
when: 
  - create_ansible_user | bool
  - install_docker | bool           # Both must be true
```

**Conditional Logic:**
```yaml
when: ansible_user != "ansible"     # Skip if already connecting as ansible user
```

### Where Used

- **User creation tasks** - Only run when `create_ansible_user: true`
- **Docker installation tasks** - Only run when `install_docker: true`
- **OS-specific tasks** - Only run on Debian-family systems
- **SSH key copying** - Only run when not already ansible user

### Benefits

- **Flexible execution** - Skip parts of playbook as needed
- **Idempotent** - Safe to run multiple times
- **Cross-platform** - Conditional OS detection
- **Error prevention** - Avoid conflicts with existing setups

## Handlers

### Defined Handlers

```yaml
handlers:
  - name: restart docker
    systemd:
      name: docker
      state: restarted
      
  - name: reload systemd
    systemd:
      daemon_reload: yes
```

### Where Triggered

**restart docker handler:**
```yaml
- name: Install Docker CE on Ubuntu
  apt: [docker packages]
  notify: restart docker    # Triggers handler
```

**reload systemd handler:**
```yaml
- name: Add Docker's official GPG key
  get_url: [download key]
  notify: reload systemd    # Triggers handler
```

### Handler Benefits

- **Efficiency** - Only restart services when actually changed
- **Order** - Handlers run after all tasks complete
- **Deduplication** - Handler runs once even if notified multiple times
- **Service Management** - Proper service restart sequence

## Roles (Alternative Organization)

### Role Structure

**User Management Role:**
- Location: `roles/user_management/`
- Purpose: Handle ansible user creation and SSH setup
- Tasks: User creation, sudo setup, SSH key copying
- Handlers: User creation notifications

**Docker Installation Role:**
- Location: `roles/docker_installation/`  
- Purpose: Install and configure Docker
- Tasks: Package installation, service management, verification
- Handlers: Docker service restart, systemd reload

### Role Usage

```yaml
roles:
  - role: user_management
    when: create_ansible_user | bool
  - role: docker_installation
    when: install_docker | bool
```

### Role Benefits

- **Modularity** - Each role has single responsibility
- **Reusability** - Roles can be used in other playbooks
- **Organization** - Clean separation of concerns
- **Maintainability** - Easier to update individual components

## Playbook Execution Options

### Standard Execution
```bash
ansible-playbook -i inventory docker-install-playbook.yml --ask-become-pass
```

### Skip User Creation
```bash
ansible-playbook -i inventory docker-install-playbook.yml --ask-become-pass \
  --extra-vars "create_ansible_user=false"
```

### Only Install Docker
```bash
ansible-playbook -i inventory docker-install-playbook.yml --ask-become-pass \
  --extra-vars "create_ansible_user=false install_docker=true"
```

### Target Specific Host
```bash
ansible-playbook -i inventory docker-install-playbook.yml --ask-become-pass \
  --limit user1
```

## What the Playbook Does

The playbook performs the following tasks on each target VM:

### User Management
- Creates a dedicated `ansible` user
- Adds ansible user to sudo group with NOPASSWD privileges
- Copies SSH keys from the current user to ansible user
- Sets up proper SSH directory permissions

### Docker Installation
- Updates apt package cache
- Installs required dependencies
- Adds Docker's official GPG key
- Adds Docker repository
- Installs Docker CE, CLI, containerd, and plugins
- Installs Docker Compose (both plugin and standalone)
- Starts and enables Docker service
- Adds ansible user to docker group

### File Management
- Creates `/home/ansible/docker/` directory
- Copies your Dockerfile to each VM
- Sets proper file ownership and permissions

## Post-Installation

After successful execution:

1. **The ansible user is created** with:
   - Sudo access without password requirement
   - Docker group membership
   - SSH key access

2. **Docker is installed and running** with:
   - Docker CE latest version
   - Docker Compose plugin and standalone
   - Service enabled for auto-start

3. **Your Dockerfile is copied** to:
   - Location: `/home/ansible/docker/Dockerfile`
   - Owner: ansible user

## Future Playbook Runs

For subsequent playbooks, you can update your inventory to use the ansible user:

```ini
[prod]
user1 ansible_host=192.168.112.128 ansible_user=ansible
user2 ansible_host=192.168.112.130 ansible_user=ansible

[prod:vars]
ansible_connection=ssh
ansible_python_interpreter=/usr/bin/python3
```

Then run without password prompts:
```bash
ansible-playbook -i host.ini your-next-playbook.yml
```

## Troubleshooting

### Common Issues

**SSH Connection Failed:**
```bash
# Test SSH connectivity manually
ssh user1@192.168.112.128

# Check SSH key was copied correctly
ssh-copy-id user1@192.168.112.128
```

**Sudo Permission Denied:**
- Ensure the user has sudo privileges on the target VM
- Use `--ask-become-pass` flag to provide sudo password

**Inventory Parse Error:**
- Check host.ini syntax (no spaces around = signs)
- Ensure proper indentation in YAML files

**Docker Service Not Starting:**
```bash
# Check Docker status on target VM
sudo systemctl status docker
sudo journalctl -u docker
```

### Verification Commands

After playbook execution, verify on target VMs:

```bash
# Check ansible user was created
id ansible

# Check Docker installation
docker --version
docker-compose --version

# Check Docker service
sudo systemctl status docker

# Check Dockerfile was copied
ls -la /home/ansible/docker/
```

## Security Notes

- The ansible user is created with NOPASSWD sudo access for automation purposes
- SSH keys are used for authentication
- Docker group membership allows running Docker without sudo
- Consider using Ansible Vault for sensitive data in production environments

## Requirements

- **Control Machine:** Ansible 2.9+
- **Target VMs:** Ubuntu 18.04+ with SSH access
- **Network:** SSH connectivity between control machine and target VMs

## License

This project is provided as-is for educational and automation purposes.
