# Ansible Detection as Code Lab

Manage custom Elastic Security detection rules through the Kibana REST API.
GitHub stores reviewed definitions; the Ansible controller performs deployment.

Target environment: Elastic Stack 9.4.3 (reported lab version). The starter uses
public v9 detection APIs. Live compatibility and permissions must be verified
against the lab before enabling rules. No Kibana endpoint or credentials are
included, and no live deployment has been performed.

## Scope

This first implementation supports custom KQL query rules only. It does not
manage Elastic prebuilt rules, dashboards, connectors, exceptions, or host agents.
It never deletes rules. Removing a JSON file does not remove its Kibana rule.

- `rules/*.json`: one definition per file, with a permanent `rule_id`.
- `playbooks/validate.yml`: offline validation of the supported schema.
- `playbooks/deploy.yml`: GET, compare, snapshot, POST or PATCH, then read back.
- `playbooks/verify.yml`: read-only comparison with Git definitions.
- `playbooks/test.yml`: local Ansible smoke test.

Requires Ansible Core 2.16+ and Python on the controller. Only built-in modules
are used. Run commands from the repository root. No SSH connection to Kibana is
required; HTTPS access from the controller is required.

## 1. Validate offline

```bash
ansible-playbook -i localhost, playbooks/test.yml
ansible-playbook -i localhost, playbooks/validate.yml
ansible-playbook -i localhost, playbooks/deploy.yml --syntax-check
ansible-playbook -i localhost, playbooks/verify.yml --syntax-check
```

Validation checks required fields, types, supported rule type, and duplicate IDs.
It does not compile KQL, check index mappings, validate credentials, or prove that
alerts will be generated. Keep the sample disabled until live testing.

## 2. Configure the controller for the hands-on session

Obtain the actual Kibana HTTPS base URL (including any reverse-proxy base path),
the existing Space ID, and an encoded Elastic API key. The key is separate from
the GitHub token. Its identity needs detection-rule management privileges in the
Space and access to the rule's source indices. Creating or updating rules can
change their execution authorization. Review the Elastic privileges documentation.

Enter values in the controller terminal, never commit them:

```bash
export KIBANA_URL='https://kibana.example.invalid:5601'
export KIBANA_SPACE='lab'
read -rsp 'Encoded Elastic API key: ' KIBANA_API_KEY
export KIBANA_API_KEY
# Optional PEM CA bundle on the controller for an internal certificate authority:
export KIBANA_CA_PATH='/path/to/internal-ca.pem'
```

Replace the example URL and Space. If the default Space is intended, explicitly
set `KIBANA_SPACE=default`. Omit KIBANA_CA_PATH when system trust is sufficient.
TLS verification stays enabled and redirects are not followed. Secrets and HTTP
responses are suppressed in Ansible output. Do not enable verbose debug logging
or capture the environment. Ansible Vault or an approved secret manager can be
introduced for persistent secret storage; this starter uses environment variables.

## 3. Deploy the disabled smoke-test rule

The example targets `logs-dac.smoke_test-*` and matches
`event.dataset : "dac.smoke_test"`. This is a synthetic test definition, not a
production detection. Decide on source indices and test events before enabling.

```bash
ansible-playbook -i localhost, playbooks/deploy.yml
ansible-playbook -i localhost, playbooks/verify.yml
```

Existing rules must carry `ManagedBy:ansible-dac` and be custom query rules.
Unchanged managed fields produce no API update. Changed rules are patched, so
unmanaged fields are preserved. Lists are compared in order; keep their order
stable. A second deployment should perform no rule mutation, although creating
a fresh local backup directory still counts as an Ansible filesystem change.
Run one deployment at a time and avoid simultaneous Kibana UI edits.

Before each update, the full previous response is saved in a private temporary
directory printed by the playbook. Move snapshots to approved durable storage
outside the repository if retention is needed. The operating system may clean
its temporary directory. Snapshots may contain internal rule information.

Live playbooks intentionally reject --check. Offline validation is not a live
dry-run. Read-back verification checks configuration, not execution health.

## 4. Enable only after review

Review mappings, look-back window, interval, source access and expected matches.
Set `enabled` to true in the JSON, review and commit the change, then run:

```bash
ansible-playbook -i localhost, playbooks/deploy.yml -e allow_enabled_rules=true
```

Check rule execution status and expected alerts in Kibana. Enabling is explicit;
the starter does not create test events, indices, or data integrations.

## Recovery and change control

Use a reviewed Git revert of a rule change, then deploy again to restore managed
fields. A raw GET snapshot contains server-generated fields: do not POST it as-is.
Restoring unmanaged fields from a snapshot requires a reviewed API request.
There is no automatic rollback across multiple rules; if deployment fails midway,
inspect the results, correct the cause, and rerun. Do not run concurrent deployments.

This repository may be public. Keep internal URLs, real infrastructure inventory,
credentials and confidential detection logic out of public commits. Use an
approved private destination before adding organization-specific content.

## Official references

- https://www.elastic.co/docs/api/doc/kibana/v9/operation/operation-createrule
- https://www.elastic.co/docs/api/doc/kibana/v9/operation/operation-readrule
- https://www.elastic.co/docs/api/doc/kibana/v9/operation/operation-patchrule
- https://www.elastic.co/docs/solutions/security/detect-and-alert/detections-privileges
- https://www.elastic.co/docs/solutions/security/detect-and-alert/detection-rule-concepts
