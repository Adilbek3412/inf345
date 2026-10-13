# Practice 07 — Your First Playbook

**Objective:** describe a small "deployment" as Ansible code — an
inventory, one play, four real modules, variables — and prove it's
idempotent: the second run changes nothing. The skills from today's
lecture.

**Timebox:** ~30 min of actual work within today's session.

> **How to submit:** same as last week — on **Maru**. Click **Accept**
> on "Practice 07", accept the GitHub invite, clone your private repo
> under `weeebdev-edu`, and work there. Commit and push to `main` —
> GitHub Actions grades every push automatically, in about a minute.

## Setup: getting Ansible

Ansible's control node runs on **Linux or macOS** (on Windows, use WSL).

```bash
pipx install ansible-core        # or: pip install --user ansible-core
ansible --version
```

No Ansible on your machine? Run it in a container instead — same result:

```bash
docker run --rm -it -v "$PWD":/work -w /work python:3.12-slim \
  sh -c "pip install -q ansible-core && ansible-playbook site.yml"
```

## Task

Your repo has these already done, which you do not need to touch:

- `ansible.cfg` — points Ansible at `inventory.ini`.
- `files/index.html` — a page to deploy.
- `templates/app.conf.j2` — a Jinja2 template for the app's config, using
  `app_name`, `app_port` and `app_env`.

Your job is two files, both with `TODO` comments:

1. **`inventory.ini`** — a group called `web` with one host, `localhost`.
   Ansible must reach it **without SSH**: give the host
   `ansible_connection=local`.
2. **`site.yml`** — one play targeting `web` that, under
   `app_root` (`/tmp/inf345-p07/app`):
   1. creates the directory, mode `0755` — module `file`
   2. copies `files/index.html` into it, mode `0644` — module `copy`
   3. renders `templates/app.conf.j2` to `app.conf` — module `template`
   4. makes sure `.env` contains the line `APP_ENV=production`, creating
      the file if it's missing — module `lineinfile`

   Define `app_name: inf345`, `app_port: 8080` and `app_env: production`
   as **variables** (in the play's `vars:`, or as group vars in the
   inventory). Never hardcode them in a task.

**Real modules only** — no `shell`, `command`, `raw` or `script`. Those
run whatever you give them and report "changed" every time, so Ansible
can't tell you whether anything actually changed.

Test it locally before you push:

```bash
ansible-inventory --graph                  # @web -> localhost
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff   # dry run: what WOULD change
ansible-playbook site.yml                  # changed=4
ansible-playbook site.yml                  # changed=0  <- idempotent
cat /tmp/inf345-p07/app/app.conf

ansible-playbook site.yml -e app_port=9090 # extra vars win: port = 9090
```

## Definition of done

- [ ] `ansible-inventory --graph` shows `localhost` in `@web`, with a local connection
- [ ] `ansible-playbook site.yml --syntax-check` passes
- [ ] The playbook runs and creates the directory (0755), `index.html`
      (0644, identical to `files/index.html`), `app.conf` and `.env`
- [ ] `app.conf` is the template rendered with your variables
- [ ] `-e app_port=9090` changes the port in `app.conf`
- [ ] A second run reports `changed=0`
- [ ] No `shell` / `command` / `raw` / `script` tasks
- [ ] Pushed to `main` before the session ends (this is also your
      attendance signal)

## Common mistakes

- **"Could not match supplied host pattern" / "skipping: no hosts
  matched".** The play's `hosts:` must be the group name from your
  inventory (`web`), and `ansible.cfg` must be in the directory you run
  from.
- **"Failed to connect to the host via ssh".** You forgot
  `ansible_connection=local` — without it Ansible tries SSH, even for
  `localhost`.
- **`.env` fails with "Destination ... does not exist".** `lineinfile`
  only edits existing files unless you say `create: true`.
- **Second run says `changed=1`.** Something isn't declarative — a
  `shell`/`command` task, or a `copy` with content that differs each run.
- **YAML indentation.** Tasks are a list (`- name: ...`); module arguments
  are indented under the module name. `--syntax-check` tells you the line.

## Grading

Real autograding — GitHub Actions installs Ansible, runs your playbook
against the runner itself, and checks what it did, out of 100 points:

| Check | Points |
|---|---|
| Inventory: `web` → `localhost`, local connection | 10 |
| `--syntax-check` passes | 10 |
| Playbook runs and does work on `localhost` | 15 |
| Directory, `index.html` and `.env` are right | 15 |
| `app.conf` rendered from the template with your variables | 15 |
| `-e app_port=9090` changes the rendered config | 10 |
| Idempotent: second run `changed=0` | 15 |
| Only real modules — no shell/command/raw/script | 10 |

Check the **Actions** tab in your repo for the run, or the workflow
run's **Summary** for the exact score. Workflow:
`.github/workflows/classroom.yml` (in your repo, not this one).

This score is this session's grade within the **Weekly practice
sessions** category — 1 point per practice, your best 10 count (see
`SYLLABUS.md`).

## Connection to Lab 02

Lab 02 is Red Hat Academy **RH294** (Red Hat Enterprise Linux Automation
with Ansible), due **Wed, Nov 4**. Today's material is its first chapters
— this practice is a good warm-up for its guided exercises.
