Homelab Ansible Playbook

Overview
- Bootstraps Debian-based nodes with base CLI tools
- Installs and enables Docker (Engine, Buildx, Compose plugin)
- Sets up SSH for key-based login and hardens SSH configuration (disables password and challenge-response auth; root login password disabled)
- Installs K3s: server on the control plane group and agents on worker nodes

Project layout
- inventory.ini — your hosts and common variables
- bootstrap-homelab.yaml — the playbook with 3 plays:
  - Base bootstrap for all_nodes (packages, SSH key install, SSH hardening, Docker)
  - K3s server for control_plane
  - K3s agent for workers

Prerequisites
- Targets: Debian/Ubuntu hosts reachable over SSH
- The user defined in the inventory (default: florian) can authenticate over SSH on first run
- Sudo is available on targets (become enabled); if sudo needs a password, you can provide it at runtime
- The playbook configures passwordless sudo for the configured user on each node (via /etc/sudoers.d)
- Control node: Ansible installed (you run the playbook from here)
- Have a public key available on the control node filesystem (path set by ssh_public_key_path)

Important variables
- ssh_public_key_path: path on the control node to the public key to deploy (default: ~/.ssh/id_rsa.pub). This is copied to each node on the first run.
- ansible_ssh_private_key_file (inventory): path to the private key used by Ansible to connect (default in inventory: ~/.ssh/id_rsa). Adjust to your key (e.g., ~/.ssh/id_ed25519) if needed.
- k3s_version: optional version pin for K3s (e.g., v1.30.4+k3s1). Leave empty for latest

First run (no SSH keys yet)
On the very first run, connect using passwords so the playbook can copy your public key to each node. If sudo requires a password, you’ll be prompted for it as well.

  ansible-playbook -i inventory.ini bootstrap-homelab.yaml --ask-pass --ask-become-pass

What happens during this run:
- Base packages are installed and upgraded
- Your public key is copied to /home/<ansible_user>/.ssh/authorized_keys
- SSH hardening disables password-based SSH logins (after your key is installed)
- Docker is installed, started, and your user is added to the docker group
- K3s server is installed on the control plane; workers join as agents

Subsequent runs (key-based)
After the first successful run, SSH key-based auth will work and you typically won’t need --ask-pass. Because the playbook configures passwordless sudo for the user, you also won’t need --ask-become-pass on subsequent runs.

  ansible-playbook -i inventory.ini bootstrap-homelab.yaml

Note on Docker repository setup (apt-key removal)
Modern Debian/Ubuntu releases removed apt-key. This playbook uses the recommended keyrings + signed-by method to add Docker’s APT repository:
- places the dearmored Docker key in /etc/apt/keyrings/docker.gpg
- references it via signed-by in the repo entry
No extra action is required on your side; this fixes common “Failed to find required executable apt-key” errors.

Pinning K3s version
Pass k3s_version at runtime or set it in group/host vars:

  ansible-playbook -i inventory.ini bootstrap-homelab.yaml -e k3s_version="v1.30.4+k3s1"

WSL2 users (Windows + Linux interop)
The lookup('file', ssh_public_key_path) runs on the control node. If you run Ansible from WSL, use a path that WSL can read.
- If your key is inside WSL (recommended): set ssh_public_key_path to a WSL path, e.g., ~/.ssh/id_ed25519.pub
- If your key is on Windows, reference it via WSL’s mount path, for example:

  ansible-playbook -i inventory.ini bootstrap-homelab.yaml \
    -e ssh_public_key_path=/mnt/c/Users/<YourWindowsUser>/.ssh/id_ed25519.pub \
    --ask-pass --ask-become-pass

Safety notes
- Because the playbook disables SSH password authentication, ensure your public key path is correct and readable from the control node so that the key is actually deployed before hardening takes effect.
- If the configured user doesn’t exist or can’t sudo, perform the initial bootstrap with a known admin (often root) and adjust become usage accordingly for that first run.
- Passwordless sudo is implemented by creating /etc/sudoers.d/<ansible_user> with "NOPASSWD:ALL" and validated with visudo. If you prefer passworded sudo, remove or override that file.

Troubleshooting
- SSH still asks for a password after first run:
  - Verify the public key exists at ssh_public_key_path on the control node
  - Confirm the key appears on targets in /home/<ansible_user>/.ssh/authorized_keys
  - Ensure file permissions on ~/.ssh (0700) and authorized_keys (0644) are correct; the playbook manages these
- WSL path errors when reading the key:
  - Use /mnt/c/… instead of a Windows-style C:\ path
- K3s agents fail to join:
  - Re-run the control plane play and ensure the node token is gathered; workers fetch it via hostvars
  - Check connectivity from workers to the control plane’s port 6443

License
This repository is provided as-is. Use at your own risk in homelab environments.
