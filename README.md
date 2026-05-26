# Lab: Setup Port Forwarding (DNAT) — Rich `forward-port` Rules

- **Series:** linux-ops-mastery — RHCSA Firewall
- **Subjects covered:** Destination NAT via **`firewall-cmd` rich rules**, `forward-port port=`, `protocol=`, `to-port=`, optional `to-addr=`, **`--add-forward-port`** equivalence, **`--permanent`**, **`--reload`**, verifying with `--list-forward-ports` and `--list-rich-rules`
- **Career arcs covered:** RHCSA (EX200 — publish internal app on alternate port), RHCE (Ansible `firewalld` forward-port parameters), SRE (temporary traffic steering during blue/green cuts), DevOps (sidecar ingress without changing container listen), AI/MLOps (expose notebook portal 8008 behind corporate standard 80)
- **Prerequisite:** Labs on **services**, **rich rules** basics, and awareness of **`ip_forward`** when forwarding through the host
- **Time Estimate:** 30 to 45 minutes
- **Difficulty arc:** Task 1 inventory · 2–3 runtime DNAT 80→8008 · 4 permanent + reload · 5 edge: list forward ports vs rich view · 6 capstone + remove forwarding cleanup

---

## Objective

Publish a service listening on **TCP 8008** as if it lived on **TCP 80** from the client's perspective — classic **port DNAT**. You will implement the redirect with **`firewalld`** (rich `forward-port` language), confirm listings, persist across reboots, and remove every forwarding artifact in cleanup.

The capstone states: *"Inbound TCP 80 on the firewall host should transparently land on local port 8008 — show permanent and runtime confirmation, then revert."*

> **Lab safety note:** Port 80 is sensitive on shared networks. Run listeners on **`127.0.0.1:8008`** or an isolated lab NIC. Do not conflict with a real production `httpd` without coordination.

---

## Concept: DNAT Rewrites the Destination Before the Socket Layer Sees It

Clients connect to **`:80`**. The kernel **rewrites** the packet to **`:8008`** (same IP in the simple local case) so the application bind on 8008 receives the flow.

```
   Client ──► :80 on RHEL ──► netfilter DNAT ──► local process :8008
                 ▲                                 │
                 └──────── same host ──────────────┘
```

`firewalld` expresses this as either a **rich rule** `forward-port` stanza or the shorthand **`--add-forward-port`**. RHCSA may phrase it either way; mastering both strings reduces exam anxiety.

> **Why this matters:** Legacy apps hard-coded to 8008 can still be presented on **standard HTTP ports** for corporate egress proxies that only allow 80/443.

---

## 📜 Why DNAT Landed in `firewalld` Abstractions — The Story

Before declarative firewalls, administrators maintained **`PREROUTING` NAT** tables by hand. One typo and inbound web traffic silently blackholed. `firewalld` wrapped the same netfilter hooks with **reviewable strings** (`forward-port`) that appear in **`--list-forward-ports`** — easier to screenshot in change records than raw `nft` translations.

Rich rules extended the grammar so **conditional** forwards ("only from this subnet") are expressible — the RHCSA baseline often stops at **unconditional local redirect** for simplicity.

> **The point of the story:** You are still doing **DNAT** — `firewalld` is the authoring surface, not a different protocol.

---

## 👪 The Port-Forward Family — Who Lives There

### Two equivalent authoring styles

| Style | Example shape |
|---|---|
| Rich rule | `rule family="ipv4" forward-port port="80" protocol="tcp" to-port="8008"` |
| Shorthand CLI | `firewall-cmd --add-forward-port=port=80:proto=tcp:toport=8008` |

### Listing commands

| Command | Shows |
|---|---|
| `--list-forward-ports` | Compact port forward summary |
| `--list-rich-rules` | If authored as rich rule text |
| `--list-all` | Zone summary including forwardings |

### Companion sysctl (when not local-only)

| Piece | Role |
|---|---|
| `net.ipv4.ip_forward` | Required when forwarding **through** router, not just local redirect |

> **The point of the family tree:** Pick **one** representation in production to avoid duplicate DNAT — double redirect misconfigurations are painful to tcpdump.

---

## 🔬 The Anatomy of `forward-port` — In One Diagram

```
$ sudo firewall-cmd --add-rich-rule='rule family="ipv4" forward-port port="80" protocol="tcp" to-port="8008"'
  │      │              │                         │
  │      │              │                         └─ rewrite destination port to 8008 (local default)
  │      │              └─ match inbound TCP 80 for IPv4 family
  │      └─ merges into runtime policy for active zone (unless `--zone=`)
  └─ root

Shorthand twin:
  sudo firewall-cmd --add-forward-port=port=80:proto=tcp:toport=8008
```

> **Reading rule:** `protocol="tcp"` is required context when using port numbers — omitting it yields invalid or ambiguous rules.

---

## 📚 Port Forwarding Reference Table

| Task | Command | Notes |
|---|---|---|
| Rich DNAT local | `--add-rich-rule='rule family="ipv4" forward-port port="80" protocol="tcp" to-port="8008"'` | Exam-flexible |
| Shorthand | `--add-forward-port=port=80:proto=tcp:toport=8008` | Terse |
| Remote target | add `:toaddr=w.x.y.z` in shorthand | Forward to another server — advanced |
| List | `--list-forward-ports` | Verify |
| Persist | `--permanent ...` + `--reload` | Required pattern |
| Remove rich | `--remove-rich-rule='...'` exact | Same text |
| Remove shorthand | `--remove-forward-port=port=80:proto=tcp:toport=8008` | Must match fields |

> **Rule one of DNAT:** Confirm **SELinux** / **process bind address** / **app listen socket** after firewall — the chain fails at the first broken hop.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Shows you can implement **documented `forward-port`** strings under time pressure. |
| **RHCE candidate** | Map shorthand keys to Ansible module args (`port`, `proto`, `toport`, `toip`). |
| **SRE / Platform** | Blue/green cuts often temporarily steer 80→8080 canary. |
| **DevOps** | Ingress controllers sometimes need host-level DNAT before CNI fully owns paths. |
| **AI / MLOps** | JupyterHub sometimes listens high; corporate clients expect 80/443 only. |

---

## 🔧 The 6 Tasks

> Build **list → runtime redirect → listener proof concept → permanent → reload → listing edge → cleanup remove**.

---

### Task 1 — Set up: capture baseline forward-port and rich-rule lists

**Purpose:** Prove the VM has **no** existing 80→8008 redirect before you begin — avoids duplicating rules.

```bash
sudo firewall-cmd --state

sudo firewall-cmd --list-forward-ports
sudo firewall-cmd --list-rich-rules

sudo firewall-cmd --permanent --list-forward-ports
sudo firewall-cmd --permanent --list-rich-rules
```

**Human-Readable Breakdown:** Four empties (or unrelated lines only) mean safe to proceed. If you see stale lab rules, remove them before adding new ones.

**Reading it left to right:** Again, runtime vs permanent comparison catches half-finished homework.

**The story:** Duplicate DNAT yields **connection refused** or **wrong backend** symptoms that look like app bugs — baseline first.

**Expected output:**

```text
running

```

**Switches**

| Token | Meaning |
|---|---|
| `--list-forward-ports` | Shorthand forward list |
| `--permanent --list-forward-ports` | Staged forward list |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `not running` | Start `firewalld` |

---

### Task 2 — Core A: add runtime rich-rule DNAT from 80 to 8008

**Purpose:** Immediate **80 → 8008** redirect using the **rich language** form the objectives cite.

```bash
sudo firewall-cmd --add-rich-rule='rule family="ipv4" forward-port port="80" protocol="tcp" to-port="8008"'

sudo firewall-cmd --list-rich-rules
sudo firewall-cmd --list-forward-ports
```

**Human-Readable Breakdown:** After success, `list-rich-rules` should echo the rule text. Depending on minor versions, `list-forward-ports` may also synthesize a readable line — do not panic if only rich listing appears; persistence still works.

**Reading it left to right:** Traffic arriving NEW to TCP 80 matches; kernel rewrites to 8008 before local delivery.

**The story:** This is the canonical exam string family — memorize **`forward-port port="80" protocol="tcp" to-port="8008"`** as a chunk.

**Expected output:**

```text
success
rule family="ipv4" forward-port port="80" protocol="tcp" to-port="8008"
port=80:proto=tcp:toport=8008
```

**Note:** Your system may show **only** the rich line or **both** — either is acceptable if traffic behaves; listings vary slightly across `firewalld` releases.

**Switches**

| Token | Meaning |
|---|---|
| `forward-port` | DNAT-like redirect stanza |
| `to-port="8008"` | Destination port after rewrite |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `INVALID_RULE` | Typo in `protocol` or missing quotes |
| Nothing listens on 8008 | Firewall OK — app layer fails later |

---

### Task 3 — Core B: start a disposable TCP listener on 8008 (localhost-safe)

**Purpose:** Give yourself **something to hit** after DNAT — use **`ncat`** from `nmap-ncat` if installed, or **`socat`** — here **`python3`** one-liner is common on classroom images.

```bash
nohup python3 -m http.server 8008 --bind 127.0.0.1 >/tmp/lab8008.log 2>&1 &
echo $!

sudo ss -ltnp 'sport = :8008'
sudo ss -ltnp 'sport = :80'
```

**Human-Readable Breakdown:** Background micro-server logs to `/tmp/lab8008.log`. `ss` proves something listens on **8008**; port **80** may show userspace only after connect tests — exact `ss` output varies.

**Reading it left to right:** Binding **127.0.0.1** avoids exposing the lab app to the LAN; DNAT from external NIC still works in advanced topologies, but local curl tests may need `-4` and interface thought — RHCSA often simplifies to **local verification mindset**.

**The story:** Firewall tasks still fail if **no daemon listens** — always pair NAT labs with a socket.

**Expected output:**

```text
12345
LISTEN 0  5  127.0.0.1:8008  userspace:(python3,...)
```

**Switches**

| Token | Meaning |
|---|---|
| `python3 -m http.server` | Quick HTTP listener |
| `--bind 127.0.0.1` | Localhost only |
| `ss -ltnp` | Listening TCP sockets with processes |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Address already in use` | Pick 8010 instead and adjust rules consistently |
| `python3: command not found` | Install `python3` or use `ncat -l` |

---

### Task 4 — Persistence: stage the same DNAT in `--permanent` and reload

**Purpose:** Reboot-safe forward — **`--reload`** aligns runtime with permanent backing store.

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" forward-port port="80" protocol="tcp" to-port="8008"'

sudo firewall-cmd --permanent --list-rich-rules

sudo firewall-cmd --reload

sudo firewall-cmd --list-rich-rules
sudo firewall-cmd --list-forward-ports
```

**Human-Readable Breakdown:** Permanent add, list staged rules, reload, list runtime again. You want the forward present after reload — proving disk → runtime path succeeded.

**Reading it left to right:** If you previously had **only runtime** rule from Task 2, reload might duplicate logically — ideally remove runtime-only duplicates before reload in real life; for teaching, removing everything in Task 6 resets state.

**The story:** Exam graders check **permanent list** and **post-reload runtime list** match expectations.

**Expected output:**

```text
success
rule family="ipv4" forward-port port="80" protocol="tcp" to-port="8008"
success
rule family="ipv4" forward-port port="80" protocol="tcp" to-port="8008"
port=80:proto=tcp:toport=8008
```

**Switches**

| Token | Meaning |
|---|---|
| `--permanent --add-rich-rule` | Persist rich DNAT |
| `--reload` | Apply |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Rule missing after reload | Permanent add failed — read stderr |
| Duplicate rules | Remove extras before reload |

---

### Task 5 — Edge case: add equivalent shorthand forward on a *different* high port (then remove)

**Purpose:** See the **`--add-forward-port`** spelling so Ansible's `port:` / `toport:` keys feel familiar — **immediately remove** to avoid double-DNAT confusion on port 80.

```bash
sudo firewall-cmd --add-forward-port=port=2222:proto=tcp:toport=22

sudo firewall-cmd --list-forward-ports

sudo firewall-cmd --remove-forward-port=port=2222:proto=tcp:toport=22
```

**Human-Readable Breakdown:** Temporary **2222→22** SSH jump trick (classic admin pattern) demonstrates shorthand syntax **without colliding** with your HTTP DNAT on 80. Remove right away.

**Reading it left to right:** Removal string must match add string field-for-field.

**The story:** Interview bonus: cite shorthand vs rich rule trade-offs — readability vs composable conditions.

**Expected output:**

```text
success
port=80:proto=tcp:toport=8008
port=2222:proto=tcp:toport=22
success
port=80:proto=tcp:toport=8008
```

**Switches**

| Token | Meaning |
|---|---|
| `--add-forward-port=port=A:proto=tcp:toport=B` | Shorthand local DNAT |
| `--remove-forward-port=...` | Symmetric delete |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `INVALID_PORT` | Typo in port numbers |
| Removal fails | Mismatch in `proto` spelling (`tcp` vs `TCP` sensitivity) |

---

### Task 6 — Capstone + cleanup: kill listener, remove HTTP DNAT permanently, reload

**Purpose:** Document final **`list-forward-ports`**, stop **`python3 http.server`**, remove **80→8008** from permanent and runtime, reload, confirm empty.

```bash
pkill -f 'python3 -m http.server 8008' 2>/dev/null || true

RULE='rule family="ipv4" forward-port port="80" protocol="tcp" to-port="8008"'

sudo firewall-cmd --permanent --remove-rich-rule="$RULE"
sudo firewall-cmd --remove-rich-rule="$RULE" 2>/dev/null || true

sudo firewall-cmd --reload

sudo firewall-cmd --list-forward-ports
sudo firewall-cmd --list-rich-rules
```

**Human-Readable Breakdown:** `pkill` stops the lab listener. Variable holds exact rich text for symmetric removal. Both permanent and runtime removes cover variants. Reload + lists should show **no** 80→8008 forward.

**The story:** Leaving DNAT around is how the **next** student's Apache fails mysteriously on bind to 80.

**Expected output:**

```text
success
success

```

**Cleanup**

```bash
pkill -f 'python3 -m http.server 8008' 2>/dev/null || true
RULE='rule family="ipv4" forward-port port="80" protocol="tcp" to-port="8008"'
sudo firewall-cmd --permanent --remove-rich-rule="$RULE" 2>/dev/null || true
sudo firewall-cmd --remove-rich-rule="$RULE" 2>/dev/null || true
sudo firewall-cmd --reload
rm -f /tmp/lab8008.log
```

**Switches**

| Token | Meaning |
|---|---|
| `pkill -f` | Pattern-kill background server |
| `rm -f` | Remove log |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `pkill` removed unrelated python servers | Narrow pattern or `kill PID` from `ss -ltnp` |
| Rich rule still listed | Copy exact line from `--list-rich-rules` into remove |

---

## 🔍 DNAT Authoring Decision Guide

```
Need local port redirect on RHEL 9 firewalld?
  │
  ├── Simple unconditional forward?
  │       ├── rich: `forward-port port="80" protocol="tcp" to-port="8008"`
  │       └── shorthand: `--add-forward-port=port=80:proto=tcp:toport=8008`
  │
  ├── Need subnet conditions?
  │       └── extend rich rule with `source address=` prefix
  │
  └── Forward THROUGH router to another server?
          └── requires forwarding sysctl + full routing — advanced topology lab
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Baseline `--list-forward-ports` and `--list-rich-rules` (runtime + permanent)
- [ ] 02 Runtime rich DNAT **80→8008**
- [ ] 03 Background listener on **8008** + `ss` verification
- [ ] 04 `--permanent` rich DNAT + `--reload` + re-list
- [ ] 05 Shorthand **2222→22** demo + immediate removal
- [ ] 06 `pkill` listener, remove HTTP DNAT (runtime+permanent), reload, delete log

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| No backend listener | Connection refused | Task 3 listener |
| Double DNAT | Flaky connects | Remove duplicates |
| Quoting errors | `INVALID_RULE` | Copy working example |
| Forgot reload | Mismatch views | `--reload` |
| Conflicts with real `httpd` | Bind errors | Stop system Apache or pick another pair |
| Removed wrong rule | Still broken | Re-list and paste exact removal |
| Left `python3` server running | Port stuck open | `pkill` / `kill` |
| Tested only from self | Works nowhere else | Advanced: test from second VM |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize **rich DNAT fragment** + **permanent + reload + list** triad.

**RHCE candidate**
- Translate shorthand ↔ Ansible argument dict without guessing key spellings.

**SRE / Platform interview**
- Explain ordering: **DNAT happens before local INPUT policy** conceptually — enough for most panel questions.

**DevOps**
- Check Git diffs: forward-port lines should be as readable as code comments.

**AI / MLOps**
- Model-serving sidecars sometimes need corporate **443→8443** style forwards — same pattern, different integers.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Allow `http`/`https` | Often combined with publishing |
| Rich rules (deny host) | Conditional forwarding neighbors |
| IP forwarding | Router-mode prerequisites |
| Masquerading | Return-path symmetry in SNAT world |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
