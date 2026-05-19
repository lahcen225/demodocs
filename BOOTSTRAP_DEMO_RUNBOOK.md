# Bootstrap-first demo runbook

Use this when the SSH/crypto bootstrap step takes too long during a
demo. The idea is to run bootstrap first, then run the normal project
playbook without enabling bootstrap again.

Run all commands from the repository root:

```bash
cd ~/2526-pe2-lahcen225
```

## 1. Create the temporary bootstrap-only playbook

This file is temporary and lives outside the project directory:

```bash
cat >/tmp/bootstrap_crypto_only.yml <<'YAML'
---
- name: Bootstrap SSH crypto stack before provisioning
  hosts: all
  become: true
  gather_facts: false
  serial: 1
  roles:
    - role: bootstrap_crypto
YAML
```

The content of `/tmp/bootstrap_crypto_only.yml` is:

```yaml
---
- name: Bootstrap SSH crypto stack before provisioning
  hosts: all
  become: true
  gather_facts: false
  serial: 1
  roles:
    - role: bootstrap_crypto
```

## 2. Run bootstrap only

```bash
ANSIBLE_ROLES_PATH=ansible/roles ansible-playbook -i local-deploy/inventory.ini /tmp/bootstrap_crypto_only.yml --vault-password-file ansible/.vault_pass -e bootstrap_crypto_enabled=true
```

This runs only the `bootstrap_crypto` role. It may update OpenSSL,
OpenSSH, crypto policies, and reboot VMs if packages changed.

## 3. Run the normal playbook

```bash
ansible-playbook -i local-deploy/inventory.ini ansible/playbook.yml --vault-password-file ansible/.vault_pass
```

Do not add `-e bootstrap_crypto_enabled=true` to this second command.
Without that variable, the bootstrap tasks in `ansible/playbook.yml`
are skipped and the rest of the deployment runs normally.

## Command order

```bash
cd ~/2526-pe2-lahcen225
cat >/tmp/bootstrap_crypto_only.yml <<'YAML'
---
- name: Bootstrap SSH crypto stack before provisioning
  hosts: all
  become: true
  gather_facts: false
  serial: 1
  roles:
    - role: bootstrap_crypto
YAML
ANSIBLE_ROLES_PATH=ansible/roles ansible-playbook -i local-deploy/inventory.ini /tmp/bootstrap_crypto_only.yml --vault-password-file ansible/.vault_pass -e bootstrap_crypto_enabled=true
ansible-playbook -i local-deploy/inventory.ini ansible/playbook.yml --vault-password-file ansible/.vault_pass
```



```
  ansible -i local-deploy/inventory.ini storage -b -m shell -a "sudo -u postgres psql ticketing -c '\dt'"
```
  Dat toont de tabellen.

  Daarna bijvoorbeeld users/events/reservaties bekijken:
```
  ansible -i local-deploy/inventory.ini storage -b -m shell -a "sudo -u postgres psql ticketing -c 'SELECT * FROM events LIMIT 5;'"

  ansible -i local-deploy/inventory.ini storage -b -m shell -a "sudo -u postgres psql ticketing -c 'SELECT * FROM users LIMIT 5;'"

  ansible -i local-deploy/inventory.ini storage -b -m shell -a "sudo -u postgres psql ticketing -c 'SELECT * FROM reservations LIMIT 5;'"
```
  Als je eerst niet weet hoe de tabellen exact heten:
```
  ansible -i local-deploy/inventory.ini storage -b -m shell -a "sudo -u postgres psql ticketing -c '\dt'"
```
