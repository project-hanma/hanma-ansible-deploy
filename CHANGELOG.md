# Changelog

All notable changes to the **Hanma Ansible Deploy** project will be documented in this file. This project orchestrates rootless Podman deployments for the Hanma Static Site Generator (SSG) utilizing systemd integrations (Quadlets).

---

## 🏗️ Milestone 6: Build Optimization & Idempotency Refactor
*May 27, 2026*

> [!TIP]
> This milestone optimizes the local build lifecycle by implementing Git-aware idempotency and surgical image cleanup. By persisting build contexts and using targeted labels, deployment times are reduced and shared host resources are protected from aggressive image pruning. This final iteration ensures correct task sequencing and fully reactive service restarts using strategically placed handler flushes.

### Commits
* **56948a7** — *refactor: implement reactive service restarts via handler flushes* (Chris Hammer)
  * **Intent:** Transition the service restart logic from an unconditional model to a purely reactive one.
  * **Rationale:** 
    * **Reactive Restarts:** Removed manual `restart_site` fact toggling in favor of global handler notifications across build, pull, and config tasks.
    * **Final Synchronization:** Added a terminal `flush_handlers` meta-task to ensure all state changes (including image and config updates) are resolved before the final systemd service state is enforced.
  * **Files Modified:** `hanma_deploy.yml`, `roles/podman_build/tasks/main.yml`

* **7920803** — *refactor: finalize podman build sequence and handler timing* (Chris Hammer)
  * **Intent:** Ensure deterministic build-before-deploy execution and clean handler sequencing.
  * **Rationale:** 
    * **Handler Synchronization:** Integrated `ansible.builtin.meta: flush_handlers` in the main deployment playbook to force container builds to complete before systemd service startup.
    * **Resilient Cleanup:** Transitioned the image pruning routine into a handler to guarantee it only executes after a successful image build.
    * **Label Consistency:** Synchronized label filters across build and cleanup handlers using the variablized `site_container_name` for precise cache management.
  * **Files Modified:** `hanma_deploy.yml`, `roles/common_handlers/handlers/main.yml`, `roles/podman_build/tasks/main.yml`

* **ddb42bb** — *refactor: optimize podman build idempotency and pruning* (Chris Hammer)
  * **Intent:** Transform the build role into a reactive, idempotent process.
  * **Rationale:** 
    * **Git Persistence:** Removed the build directory cleanup task, allowing the Git module to perform delta updates instead of full clones.
    * **Event-Driven Builds:** Relocated the build task to a handler triggered only on source changes.
    * **Path Hidden:** Relocated the build context to `.hanma_src` across all variable files.
    * **Targeted Pruning:** Implemented `podman image prune` with label filters to spare unrelated containers on the host.
  * **Files Modified:** `roles/common_handlers/handlers/main.yml`, `roles/podman_build/tasks/main.yml`, `vars/*.yml`

* **97b48f9** — *refactor: variablize podman build labels* (Chris Hammer)
  * **Intent:** Ensure consistent labeling for multi-container host environments.
  * **Rationale:** Variablized the build labels using `{{ site_container_name }}` to ensure that multiple Hanma instances on the same host can manage their own image lifecycles without interference.
  * **Files Modified:** `roles/common_handlers/handlers/main.yml`, `roles/podman_build/tasks/main.yml`

---

## 🛡️ Milestone 5: Security Hardening & Deployment Reliability
*May 26 – May 27, 2026*

> [!IMPORTANT]
> This milestone focuses on infrastructure security and deployment determinism. By implementing unprivileged container namespaces and resilient task orchestration, the project now adheres to higher security standards and guarantees cleanup of sensitive staging artifacts even during failed executions.

### Commits
* **b6975db** — *ansible.cfg/minor debug formatting tweak* (Chris Hammer)
  * **Intent:** Clean up configuration defaults and refine debug output.
  * **Rationale:** Pruned redundant settings from `ansible.cfg` (e.g., `forks`, `host_key_checking`) to improve readability. Applied YAML block scalar chomping (`|-`) to the container build debug message in the main playbook for cleaner log output.
  * **Files Modified:** `ansible.cfg`, `hanma_deploy.yml`

* **cce964f** — *chore: remove stale and unused hanma.container.j2 template* (Chris Hammer)
  * **Intent:** Decommission legacy container templates.
  * **Rationale:** Removed `templates/hanma.container.j2`, an obsolete artifact superseded by the modular `podman_quadlet` role. This cleanup ensures the codebase remains focused on the active Quadlet-based architecture.
  * **Files Modified:** `templates/hanma.container.j2` (Deleted)

* **21c1ce9** — *chore: address security audit findings for unprivileged execution and deployment reliability* (Chris Hammer)
  * **Intent:** Close security gaps and enhance the robustness of the deployment lifecycle.
  * **Rationale:**
    * **Unprivileged Hardening:** Transitions container execution to a non-root service user (UID 1000). Implemented `UserNS=keep-id:uid=1000,gid=1000` mapping to bridge host-to-container permission gaps for high-numbered host UIDs.
    * **Resilient Cleanup:** Restructured the `local_site` role using a `block/always` construct to guarantee the removal of temporary staging directories in `/tmp` regardless of task success or failure.
    * **Idempotency Updates:** Added filesystem state checks to the `enable_linger` role to prevent redundant systemd commands.
    * **Build Determinism:** Forced image rebuilds using the `force: true` flag in the Podman build module and integrated post-build image pruning to maintain host disk health.
    * **Service Logic Optimization:** Refactored service start and restart operations into a single, declarative task using ternary state logic.
  * **Files Modified:** `hanma_deploy.yml`, `roles/common_handlers/handlers/main.yml`, `roles/enable_linger/tasks/main.yml`, `roles/local_site/tasks/main.yml`, `roles/podman_build/tasks/main.yml`, `roles/podman_quadlet/tasks/main.yml`, `roles/podman_quadlet/templates/podman_quadlet.j2`, `vars/dashboard_prod.yml`, `vars/hanma_dev.yml`, `vars/pi_dev.yml`, `vars/zengarden_prod.yml`

---

## 🏯 Milestone 4: Zen Garden Production & Execution Optimizations
*April 28, 2026*

> [!NOTE]
> This milestone completes the transition to a fully automated production deployment pipeline for the main Zen Garden website, adding immediate handler flushes and reactive container service restarts.

### Commits
* **a7ca0fd** — *add flush_handlers* (Chris Hammer)
  * **Intent:** Ensure that configuration and service updates are applied immediately mid-playbook rather than waiting until the end of the execution block.
  * **Rationale:** Flushes Ansible handlers immediately in the `podman_quadlet` role task sequence, preventing deployment lag and assuring immediate startup.
  * **Files Modified:** `roles/podman_quadlet/tasks/main.yml`

* **7ccc817** — *site_port change* (Chris Hammer)
  * **Intent:** Reconfigure production port allocation for the Zen Garden website container.
  * **Rationale:** Updated the target host port in the production configuration.
  * **Files Modified:** `vars/zengarden_prod.yml`

* **055fd82** — *add restart on image change; add the vars/zengarden_prod.yml* (Chris Hammer)
  * **Intent:** Auto-restart services upon image updates and establish production variables for Zen Garden.
  * **Rationale:** Integrated automated change-detection to restart systemd container services if the underlying image hash/tag is updated. Added full configuration details for Zen Garden's production release.
  * **Files Modified:** `hanma_deploy.yml`, `vars/zengarden_prod.yml`

---

## 🏗️ Milestone 3: Perms Variablization, Staging Improvements, and Registry Pull Strategy
*April 26, 2026*

> [!TIP]
> Introducing parameterized permissions makes the deploy role compatible across disparate server setups. Coupling this with a registry-first container pull strategy drastically reduces deployment times by avoiding local image building when a remote image exists.

### Commits
* **6b887f0** — *update: always force pull image from registry with local fallback* (Chris Hammer)
  * **Intent:** Optimize container deployment speed and image freshness.
  * **Rationale:** Instructs Podman to pull down updated images from the central registry first, only compiling/building the container image locally if remote retrieval fails.
  * **Files Modified:** `hanma_deploy.yml`

* **025b642** — *add vars* (Chris Hammer)
  * **Intent:** Re-establish baseline environment variables for Dev/Prod targets.
  * **Rationale:** Restores variable payloads and environment-specific configs for target nodes.
  * **Files Modified:** `.gitignore`, `vars/dashboard_prod.yml`, `vars/pi_dev.yml`

* **714c4f9** — *cleanup* (Chris Hammer)
  * **Intent:** Eliminate redundant local credentials and expand directory ignores.
  * **Rationale:** Housekeeping commit to prune unnecessary local overrides from active version control.
  * **Files Modified:** `.gitignore`, `vars/dashboard_prod.yml`, `vars/pi_dev.yml`

* **263d91e** — *Merge pull request 'perms variablized; move role, update local_site' (#8) from sync-perms into develop* (Chris H.)
  * **Intent:** Integrate permission variablization branch.
  * **Rationale:** Incorporates secure, configurable permission sets across directories and files.

* **ba5c3c7** — *perms variablized; move role, update local_site* (Chris Hammer)
  * **Intent:** Parameterize directory permissions and isolate cleaning routines.
  * **Rationale:** Moved `clean_stage_area` task files into a broader `stage_area` role structure. Variablized directories with custom ownership/group permissions to secure staging environments and systemd directories.
  * **Files Modified:** `README.md`, `hanma_deploy.yml`, `roles/local_site/tasks/main.yml`, `roles/stage_area/tasks/clean_stage_area.yml`, `roles/clean_stage_area/tasks/main.yml` (Deleted), `roles/synchronize_site/tasks/main.yml`, `vars/dashboard_prod.yml`, `vars/hanma_dev.yml`, `vars/pi_dev.yml`

* **152c2b4** — *typo; task name update* (Chris Hammer)
  * **Intent:** Improve logs readability in playbook executions.
  * **Rationale:** Clarified task names and descriptions inside the playbooks.
  * **Files Modified:** `hanma_deploy.yml`

---

## 🛠️ Milestone 2: Modular Architecture Refactor & Multi-Site Support
*April 24, 2026*

> [!IMPORTANT]
> This represents the single largest architectural shift in the project. The monolithic deploy script was dismantled and refactored into modular, reusable Ansible Roles. Additionally, official documentation and support for Ansible Automation Platform (AAP) were introduced.

| Legacy Monolithic Structure | Modular Role-Based Structure (New) |
| :--- | :--- |
| `deploy_site.yml` (79 lines of tasks) | `hanma_deploy.yml` (Orchestrator playbook) |
| inline commands | `roles/podman_build` (Modular builds) |
| static unit templates | `roles/podman_quadlet` (Systemd Quadlet spec) |
| manual directory setups | `roles/stage_area` & `roles/local_site` |
| no linting standards | `.ansible-lint` config integrated |

### Commits
* **bf7ac6a** — *docs: Add README.md detailing ansible deployment* (Chris Hammer)
  * **Intent:** Provide a user-facing technical reference for the deployment pipeline.
  * **Rationale:** Authored complete documentation of architecture, inventory setups, role capabilities, variables, and quick-start instructions.
  * **Files Modified:** `README.md`

* **9bff8be** — *Fixes to support AAP properly* (Chris Hammer)
  * **Intent:** Establish compatibility with Ansible Automation Platform runner engines.
  * **Rationale:** Patched runner environments to prevent root execution issues on AAP nodes.
  * **Files Modified:** `hanma_deploy.yml`, `roles/local_site/tasks/main.yml`

* **456fb09** — *Refactor to support all sites* (Chris Hammer)
  * **Intent:** Transition the deployment engine into a highly scalable, multi-site architecture.
  * **Rationale:** Deleted the monolithic `deploy_site.yml` in favor of a clean orchestrator (`hanma_deploy.yml`) coupled with dedicated Ansible roles:
    * `podman_build`: Manages image builds.
    * `podman_quadlet`: Generates systemd integration templates.
    * `local_site` & `git_site`: Facilitates either local files sync or remote git repository checkouts.
    * `enable_linger`: Configures persistent user lingering.
  * **Files Modified:** `.ansible-lint`, `deploy_site.yml` (Deleted), `hanma_deploy.yml`, `roles/*` (New Roles), `vars/dashboard_prod.yml`, `vars/hanma_dev.yml`, `vars/pi_dev.yml`

---

## ✅ Milestone 1: Playbook Initiation & Containerization Setup
*April 22 – April 23, 2026*

> [!NOTE]
> The starting phase focused on creating the initial Ansible scaffold, testing local/production host options, and configuring automated CI/CD checks via Gitea Actions.

### Commits
* **85785a1** — *remove notify* (Chris Hammer)
  * **Intent:** Revert experimental systemd notify handler.
  * **Files Modified:** `deploy_site.yml`

* **752317b** — *add notify* (Chris Hammer)
  * **Intent:** Hook change notifications into the deployment task flow.
  * **Files Modified:** `deploy_site.yml`

* **bf098da** — *branch change for dev* (Chris Hammer)
  * **Intent:** Point developer configs to target development branch.
  * **Files Modified:** `vars/hanma_dev.yml`

* **b68728d** — *further refinements to code and vars* (Chris Hammer)
  * **Intent:** Finetuning configuration settings and port variables.
  * **Files Modified:** `deploy_site.yml`, `vars/dashboard_prod.yml`, `vars/hanma_dev.yml`

* **d93ff4b** — *refinements to code and vars* (Chris Hammer)
  * **Intent:** Standardize site properties and remove development overrides.
  * **Files Modified:** `deploy_site.yml`, `templates/hanma.container.j2`, `vars/dashboard_dev.yml` (Deleted), `vars/dashboard_prod.yml`, `vars/hanma_dev.yml`

* **ef1ab12** — *much dynamic, many wow* (Chris Hammer)
  * **Intent:** Introduce dynamic path evaluation and automated pipeline linting.
  * **Rationale:** Configured Gitea linting workflows to ensure clean Ansible syntax on future commits, split target environments into dev vs. prod, and dynamically loaded specific variables.
  * **Files Modified:** `.ci.env`, `.gitea/workflows/ansible-lint.yml`, `deploy_site.yml`, `vars/dashboard_dev.yml`, `vars/dashboard_prod.yml`

* **b08acd1** — *initial* (Chris Hammer)
  * **Intent:** Scaffolding the basic deployment repository.
  * **Rationale:** Established repository defaults including `ansible.cfg`, custom systemd quadlet container template (`templates/hanma.container.j2`), a basic `deploy_site.yml` script, and initial host variables.
  * **Files Modified:** `.gitignore`, `ansible.cfg`, `deploy_site.yml`, `requirements.yml`, `templates/hanma.container.j2`, `vars/dashboard_vars.yml`
