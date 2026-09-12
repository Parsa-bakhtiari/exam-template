Playbook & Roles
main.yml runs four roles against the monitoring group with become: true.

common: apt cache + base packages
node_exporter: binary from upstream tarball, systemd unit, port 9100
prometheus: binaries + prometheus.yml (validated with promtool), systemd unit, port 9090, scrapes itself and node_exporter
grafana: apt repo + package, datasource and dashboard via provisioning, port and admin credentials via a systemd drop-in

Handlers restart a service only when its binary, unit or config changes, so the playbook is idempotent: a second run reports changed=0.

Grafana is fully provisioned from disk, nothing is done in the UI: /etc/grafana/provisioning/datasources/prometheus.yml for the datasource, and /var/lib/grafana/dashboards/node-cpu-memory.json for the dashboard.


Inventory

Structure unchanged. Target in inventory/inventory/monitoring.yml:

yaml
monitoring:
  hosts:
    mon-1:
      ansible_host: 95.38.235.108
      ansible_user: root

Ports are in inventory/group_vars/monitoring.yml. Versions, paths and credentials are in each role's defaults/main.yml.

Credentials / Login
# Grafana  http://95.38.235.108:3000
user: admin
pass: admin@123

# Prometheus  http://95.38.235.108:9090
no auth

Challenges : 
---------------
Started in my own Ansible tree with a generic role layout, then had to move everything into the scenario-2 template: the entrypoint must be main.yml in the repo root and the host must live in inventory/inventory/monitoring.yml, since that is exactly what the grading command reads.
ansible_architecture triggers a deprecation warning. Using ansible_facts['architecture'] is the current form.
Instead of editing grafana.ini, the port and the admin credentials are set with GF_* environment variables in a systemd drop-in. The file shipped by the package stays untouched.
