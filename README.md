# Hanma Ansible Deploy

This Ansible project is used to build, configure, and deploy the **Hanma** Python Application as a rootless Podman container managed via systemd (quadlets). It supports deploying to different environments (e.g., dev, prod, pi) by utilizing host-specific variables.

## Project Structure

- **`hanma_deploy.yml`**: The main playbook that orchestrates the deployment process.
- **`ansible.cfg`**: Ansible configuration optimized with profiling and pipelining.
- **`hosts`**: The Ansible inventory file, categorizing hosts (e.g., `dashboard_prod`, `hanma_dev`, `pi_dev`).
- **`vars/`**: Directory containing environment-specific variables for deployment configurations (`dashboard_prod.yml`, `hanma_dev.yml`, `pi_dev.yml`).
- **`roles/`**: Directory containing various roles used by the playbook:
  - `podman_build`: Manages idempotent container image builds using a persistent, hidden context. It only triggers builds when the Hanma source code is updated or the image is missing.
  - `podman_quadlet`: Sets up the Podman quadlet `.container` file for systemd integration.
  - `enable_linger`: Enables systemd loginctl linger to allow rootless containers to start automatically and persist after the user logs out.
  - `git_site` / `local_site`: Deploys the site content either by pulling from a Git repository or synchronizing local files.
  - `stage_area`, `common_handlers`, `synchronize_site`: Utility roles. `common_handlers` orchestrates reactive restarts and targeted image pruning using strategically placed handler synchronization (flushing).

## Deployment Flow

When running the main playbook, it executes the following steps:
1. Validates the deployment environment and loads host-specific variables.
2. Attempts to pull the image from the registry. If unavailable, triggers an idempotent local build that only compiles if source code changes are detected or the image is missing locally.
3. Deploys the Hanma Podman quadlet unit file into the user's `~/.config/containers/systemd/` directory.
4. Ensures `loginctl enable-linger` is active for the target user to support rootless background services.
5. Deploys the site's content via the appropriate role (`git_site` or `local_site`), checking out or synchronizing configuration and static content.
6. Synchronizes handlers to ensure all build, configuration, and site changes are resolved.
7. Selectively starts or restarts the systemd user service (`<site_container_name>.service`) only when changes are detected in the image, configuration, or site content.

## Usage

You can target a specific environment by passing the `site_deploy` extra variable. This variable dictates which group of hosts from the inventory will be targeted and which variable file from the `vars/` directory will be loaded.

For example, to deploy the `hanma_dev` environment:

```bash
ansible-playbook -i hosts hanma_deploy.yml -e "site_deploy=hanma_dev"
```

To deploy the production dashboard:

```bash
ansible-playbook -i hosts hanma_deploy.yml -e "site_deploy=dashboard_prod"
```

If no `site_deploy` variable is provided, the playbook defaults to the `hanma_dev` environment.

## Variables

Variables for each environment are maintained in the `vars/` directory. Key variables include:

- `site_repo`: The Git repository containing the site's content.
- `site_deploy_path`: The directory on the remote host where the site should be placed.
- `site_port`: The port that the container will expose on the host.
- `podman_build_image_name`: The container image name/tag for the registry.
- `podman_quadlet_volumes`: Volume mappings to attach configurations and content into the container.
