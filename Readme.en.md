# Git-Based Dynamic Ansible Role Management Without an Ansible Galaxy Server

This pattern shows a practical way to manage Ansible roles directly from your own Git server, without operating a dedicated Ansible Galaxy server.

The main idea is simple: roles are downloaded from Git at runtime, credentials are injected dynamically, and role versions are pinned to Git tags or commit SHAs for reproducible execution.

Instead of hiding dependencies inside roles, this approach keeps role management explicit at the project/playbook level.

## 📁 Recommended Directory Structure

To keep the project self-contained, use a flat project layout. The `roles/` directory is created and populated automatically during execution.

```text
my-ansible-project/
├── requirements.yml.j2    # Jinja2 template for Git authentication
├── playbook.yml           # Main playbook
└── roles/                 # Roles are downloaded here automatically
```

---

## 📄 Source Files

### 1. `requirements.yml.j2`

This template defines the required roles and injects Git credentials at runtime.

For production use, prefer protected Git tags or commit SHAs. A tag is easier to read, while a commit SHA gives stronger reproducibility.

```yaml
---
roles:
  - name: company.nginx
    src: "git+https://{{ git_user | urlencode }}:{{ git_token | urlencode }}@git.example.com/ansible-roles/nginx.git"
    version: "v3.1.0"

  - name: company.postgresql
    src: "git+https://{{ git_user | urlencode }}:{{ git_token | urlencode }}@git.example.com/ansible-roles/postgresql.git"
    version: "v3.4.0"
```

---

### 2. `playbook.yml`

This playbook controls the full flow:

1. Generate a temporary `requirements.yml` file with Git credentials.
2. Download the required roles into the local `roles/` directory.
3. Load the roles dynamically with `include_role`.
4. Remove the generated requirements file even if the playbook fails.

```yaml
---
- name: Dynamic Git-based role management without Galaxy server
  hosts: webservers
  become: true

  vars:
    # Recommended: store these values in Ansible Vault
    git_user: "{{ vault_git_user }}"
    git_token: "{{ vault_git_token }}"

    generated_requirements: "{{ playbook_dir }}/.requirements.generated.yml"
    local_roles_path: "{{ playbook_dir }}/roles"

  tasks:
    - name: Install and execute Git-based roles
      block:
        - name: Generate temporary requirements file with Git credentials
          ansible.builtin.template:
            src: requirements.yml.j2
            dest: "{{ generated_requirements }}"
            mode: "0600"
          delegate_to: localhost
          run_once: true
          no_log: true

        - name: Download roles into the local project directory
          ansible.builtin.command:
            cmd: >
              ansible-galaxy role install
              -r {{ generated_requirements }}
              -p {{ local_roles_path }}
              --force
          delegate_to: localhost
          run_once: true
          no_log: true

        - name: Load Nginx role dynamically
          ansible.builtin.include_role:
            name: company.nginx

        - name: Load PostgreSQL role dynamically
          ansible.builtin.include_role:
            name: company.postgresql

      always:
        - name: Remove temporary requirements file
          ansible.builtin.file:
            path: "{{ generated_requirements }}"
            state: absent
          delegate_to: localhost
          run_once: true
          no_log: true
```

---

## 🛠️ Why This Works

### Why use `include_role`?

`import_role` is static. Ansible processes it at parse time, before any task runs.

That means if the role does not exist on disk when the playbook starts, the playbook fails immediately with a `Role not found` error.

`include_role` is dynamic. It is evaluated at runtime, when the task is reached. This allows the playbook to download the required roles first, then load them afterward.

In this pattern, that distinction is essential.

---

### Why use `ansible-galaxy role install`?

Even without running an Ansible Galaxy server, the `ansible-galaxy` CLI can install roles directly from Git repositories.

This makes it useful as a lightweight role dependency installer.

The Galaxy server is not required. The CLI is only used as a local dependency management tool.

---

### Why use `--force`?

When roles already exist in the local `roles/` directory, `ansible-galaxy` may skip downloading them.

Using `--force` ensures that the requested version from `requirements.yml` is applied every time.

This is useful when a role version is changed from one Git tag or commit SHA to another.

The trade-off is that the install step may report changes more often, because the role directory is overwritten.

---

### Why pin versions?

Pinned versions make executions more reproducible.

Instead of always downloading the latest role state, the playbook uses a specific Git tag or commit SHA.

For stronger guarantees, use commit SHAs or protected/signed tags. A plain Git tag can still be moved unless your Git server prevents it.

---

### Security Considerations

Git credentials should not be stored directly in the repository.

Recommended practices:

```text
- Store Git credentials in Ansible Vault.
- Generate the credentials-based requirements file only at runtime.
- Set file permissions to 0600.
- Use no_log: true on tasks handling secrets.
- Delete the generated file in an always block.
- Add generated files to .gitignore.
```

Example `.gitignore`:

```gitignore
.roles/
roles/
.requirements.generated.yml
requirements.yml
```

---

## Summary

This approach keeps Ansible role dependencies explicit, versioned, and Git-based.

It avoids running a dedicated Galaxy server while still allowing roles to be pulled dynamically from private repositories.

The key principle is:

```text
Do not hide role dependencies inside roles.
Manage them explicitly at the project or playbook level.
```

This makes role usage easier to audit, easier to upgrade, and easier to reproduce across environments.
