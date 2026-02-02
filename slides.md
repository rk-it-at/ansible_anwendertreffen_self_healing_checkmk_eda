---
marp: true
title: Self-healing with Checkmk and Event-Driven Ansible
theme: rk-it
size: 16:9
paginate: true
footer: Ansible Anwendertreffen Austria 02/2026
---

# Self-healing with Checkmk and Event-Driven Ansible
## How to resolve issues automatically

---

<!-- _class: right-bg-author -->
# About me

- René Koch
- Self-employed consultant for:
  - Red Hat Ansible (Automation Platform)
  - Red Hat Enterprise Linux
  - Red Hat Satellite
  - Red Hat Identity Management (IPA)
- Experienced monitoring user (Nagios, Icinga,
  Checkmk)

---

<!-- _class: right-bg-author -->
# About me

- René Koch
  - rkoch@rk-it.at
  - +43 660 / 464 0 464
  - https://www.linkedin.com/in/rk-it-at
  - https://github.com/rk-it-at
  - https://github.com/scrat14

---

# Agenda

- Monitoring: the past and the present
- What is Checkmk?
- What is Event-Driven Ansible (EDA)?
- Event-Driven Workflow
- Use cases and best practices
- (Live Demonstration)

---

# Monitoring: the past and the present

- 🕰️ 2005: Received email alerts from Nagios 2 for issues with Solaris machines
- 🧩 Manual workflow:
  - 📩 Read email
  - 🔐 Log in to the system
  - 🔎 Check if issue still exists
  - 🛠️ Fix the issue
  - 🤬 **Repeat the same procedure over and over again**

---

# Monitoring: the past and the present

- 🤖 Today: Use EDA to decide if issues can be fixed with Ansible
- ⚡ Automated workflow:
  - 🧠 EDA analyzes the issue
  - 🚀 Triggers an AAP template if it can be fixed
  - 🧾 Ansible playbook solves the issue
  - 🙋 Handle only exceptional issues manually

---

<!-- _class: footnote-only -->
<!-- _backgroundImage: "url('assets/automation.png')" -->
<!-- _backgroundSize: contain -->
<!-- _backgroundPosition: center -->

<div class="footnote">
  Source: https://github.com/ansible/workshops/blob/devel/decks/ansible_rhel.pdf
</div>

---

<!-- _class: right-bg-checkmk -->
# What is Checkmk?

- **Monitoring platform** for infra-
  structure, applications and services
- Provides **agent- and agentless**
  checks, dashboards, and alerting
- Built for **scale** with distributed
  monitoring and automation support
- Helps **detect, analyze, and**
  **remediate** issues faster

---

<!-- _class: right-bg-eda -->
# What is Event-Driven Ansible?

- **EDA is automation that reacts to events** - not
  schedules
- Events can come from monitoring, webhooks,
  message queues, logs or cloud services
- Rules decide **when** to run Ansible actions
- Goal: **faster response** and **consistent**
  **remediation**

---

# What Is an "Event" (vs a Source Action)?

- ⚡ **Event**
  - a *state change* or *signal* that matters (e.g., alert fired, service down)
  - often noisy and hard to filter
- 🔁 **Source action**
  - a *routine trigger* (e.g., "on every commit")
  - predefined target/action
- 🧪 Examples:
  - 🚫 *Update an AAP project after each commit* (not EDA)
  - ✅ *Send all monitoring alerts to a webhook; EDA decides what to do* (EDA)

---

# Why EDA?

- ⚡ **Reduce MTTR** by acting immediately
- 📈 **Scale operations** without polling
- 🧩 **Standardize** responses to recurring incidents
- 🔁 **Close the loop** between detection and remediation

---

# Core Building Blocks

- 📡 **Event Sources**: where events originate (webhooks, Kafka, logs, etc.)
- 📘 **Rulebook**: conditions + actions
- 🛠️ **Actions**: run playbooks, set facts, send notifications, create tickets
- 🧭 **Controller** (optional): central execution and governance

---

<!-- _class: right-flow -->
# Event Driven Workflow

1. Event arrives from a source
2. Rulebook evaluates conditions
3. Matching rule triggers an action
4. Action runs playbook or other automation
5. Results can emit **new events** or update systems

---

<!-- _class: code-small -->
# Rulebook: restart named

```yaml
---

- name: Restart named on IPA server
  hosts: all
  gather_facts: false

  sources:
    - name: Listen on port 5000 for Checkmk events
      ansible.eda.webhook:
        port: 5000

  rules:
    - name: Restart named
      condition: >-
        event['payload']['servicename'] == "DNS example.com" and
        event['payload']['servicestate'] == "CRITICAL"
      action:
        run_job_template:
          name: "[LINUX] Restart named on IPA [@production] - Prompt"
          organization: "Default Organization"
          job_args:
            limit: "{{ event.payload.hostname }}"
```

---

<!-- _class: code-small -->
# Checkmk Notification Script

```bash
#!/usr/bin/env bash

HEADER="X-Checkmk-Token"
TOKEN="${NOTIFY_PARAMETER_1}"
URL="${NOTIFY_PARAMETER_2}"

JSON=`cat <<EOF
{
  "hostname": "${NOTIFY_HOSTNAME}",
  "hostoutput": "${NOTIFY_HOSTOUTPUT}",
  "hoststate": "${NOTIFY_HOSTSTATE}",
  "servicename": "${NOTIFY_SERVICEDESC}",
  "serviceoutput": "${NOTIFY_SERVICEOUTPUT}",
  "servicestate": "${NOTIFY_SERVICESTATE}",
  "date": "${NOTIFY_SHORTDATETIME}",
  "type": "${NOTIFY_NOTIFICATIONTYPE}",
  "what": "${NOTIFY_WHAT}"
}
EOF
`

curl -X POST -H "Content-Type: application/json" -H "${HEADER}: ${TOKEN}" -d "${JSON}" ${URL}
exit $?
```

---

# Playbook: restart named
<!-- _class: code-small -->

```yaml
---

- name: Restart named on IPA
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Restart named
      ansible.builtin.service:
        name: named
        state: restarted
```

---

<!-- _class: code-small -->
# Run a Rulebook (CLI)

```bash
$ ansible-rulebook -r rulebooks/restart_named.yml -i localhost

PLAY [Restart named on IPA] ****************************************************

TASK [Gathering Facts] *********************************************************
ok: [ipa01.example.com]

TASK [Restart named] ***********************************************************
changed: [ipa01.example.com]

PLAY RECAP *********************************************************************
ipa01.example.com : ok=2 changed=1 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0 
```

❗Use **run_playbook** action instead of **run_job_template**  for ansible-playbook
  command instead of Ansible Automation Platform.

---

# Ansible Automation Platform Integration

- 📦 **Projects**: Git repository configuration
- 🧪 **Decision Environments**: Container images to run rulebooks
- 🔐 **Credentials**: Secrets for Git, Controller, Hub, tokens,...
- 📡 **Event Streams**: Entry points for events (mapped to source definition in rulebook)
- 🚀 **Rulebook Activations**: Rulebook runs

---

<!-- _class: footnote-only -->
<!-- _backgroundImage: "url('assets/eda02.png')" -->
<!-- _backgroundSize: contain -->
<!-- _backgroundPosition: center -->

---

# Event-Driven Ansible Use Cases

- 🚨 **Monitoring alerts**: run remediation playbooks
- 🏗️ **Infrastructure events**: auto-scale or restart services
- 🔐 **Security findings**: isolate hosts or rotate credentials
- 🎫 **Ticketing**: enrich and open incidents automatically
- 📘 **Documentation**: update asset database or documentation system

---

<!-- _class: right-bg-best-practices -->
# Self Healing Best Practices

- Start with **low-risk** automations
- Use **idempotent** playbooks (if possible)
- Add **guardrails** (approvals, maintenance windows,
  downtimes)
- Emit **metrics and logs** for auditing

---

# Challenges with Self-healing

- 🔊 Triggering on noisy events (missing filtering)
- 🧪 Insufficient monitoring coverage
- 🧯 Healing the wrong host (issue caused by a backend dependency)
- 📚 Lack of knowledge or runbooks
- 🕒 Triggering during maintenance windows due to missing downtime

---

# Additional Information

- **Products**:
  - Ansible Automation Platform: https://www.redhat.com/en/technologies/management/ansible
  - Checkmk: https://checkmk.com/

- **Product Documentation**:
  - Ansible: https://docs.ansible.com/
  - Automation Decisions: https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/using_automation_decisions/index
  - Rulebooks: https://docs.ansible.com/projects/rulebook/en/latest/

---

# Summary

- 🛰️ Checkmk **monitors** your IT landscape and **notifies** on state change
- ⚡ EDA turns these events into **real-time automation**
- 🤝 It complements traditional Ansible by **reacting** instead of **scheduling**
- 🎯 Start small, measure impact, and iterate

---

<!-- _class: right-bg-author -->
# Thank you!

René Koch
Freelancer
Ansible Anwendertreffen Austria 18.02.2026
