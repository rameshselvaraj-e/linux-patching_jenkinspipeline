# linux-patching_jenkinspipeline
Yes. The best way is to make both Ansible and Jenkins show meaningful progress, rather than just displaying a long ansible-playbook log.

For your architecture:

GitHub
   ↓
Jenkins
   ↓
Ansible VM
   ↓
Linux servers

Jenkins console:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ Git checkout
✓ Ansible connectivity
✓ Pre-check
▶ Patching server01       [IN PROGRESS]
✓ server01 patched
▶ Patching server02       [IN PROGRESS]
✓ server02 patched
▶ Patching server03       [IN PROGRESS]
✓ server03 patched
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PATCHING COMPLETED: 3/3
1. Use serial: 1

This is important because it lets you see each server being processed individually.

In your playbook:

- name: Linux OS Patching
  hosts: linux_servers
  become: true
  serial: 1

With 5 servers:

server01 → patch → reboot → post-check → SUCCESS
server02 → patch → reboot → post-check → SUCCESS
server03 → patch → reboot → post-check → SUCCESS
server04 → patch → reboot → post-check → SUCCESS
server05 → patch → reboot → post-check → SUCCESS
2. Add clear progress messages to Ansible

You can add debug messages before each major operation.

For example:

- name: Show patch start
  ansible.builtin.debug:
    msg: "========== STARTING PATCHING: {{ inventory_hostname }} =========="

Then:

- name: Update RHEL packages
  ansible.builtin.dnf:
    name: "*"
    state: latest
    update_only: true
  register: patch_result
  when: ansible_facts.os_family == "RedHat"

Follow it with:

- name: Show package update result
  ansible.builtin.debug:
    msg:
      - "Server: {{ inventory_hostname }}"
      - "Package update completed"
      - "Changed: {{ patch_result.changed | default(false) }}"
3. Make Ansible output easier to read

I recommend using this format throughout the playbook:

- name: PRE-CHECK
  ansible.builtin.debug:
    msg: "[PRE-CHECK] {{ inventory_hostname }} - checking disk space"

- name: Check disk
  ansible.builtin.shell: |
    df -P / | awk 'NR==2 {print $5}' | tr -d '%'
  register: root_disk
  changed_when: false

- name: PRE-CHECK result
  ansible.builtin.debug:
    msg: "[PRE-CHECK] {{ inventory_hostname }} - disk usage {{ root_disk.stdout }}%"

Then your Jenkins console will show:

TASK [PRE-CHECK] ********************************************************

[PRE-CHECK] server01 - checking disk space

TASK [Check disk] *******************************************************

ok: [server01]

TASK [PRE-CHECK result] *************************************************

[PRE-CHECK] server01 - disk usage 42%
4. Show patching stages

I'd divide your playbook into these logical stages:

PRE-CHECK
   ↓
PACKAGE UPDATE
   ↓
REBOOT CHECK
   ↓
REBOOT
   ↓
POST-CHECK
   ↓
SUCCESS

For example:

- name: PATCH - Starting package update
  ansible.builtin.debug:
    msg: >
      [PATCH] {{ inventory_hostname }}
      - Starting package update

Then:

- name: PATCH - Update packages
  ansible.builtin.dnf:
    name: "*"
    state: latest
    update_only: true
  register: patch_result
  when: ansible_facts.os_family == "RedHat"

Then:

- name: PATCH - Package update completed
  ansible.builtin.debug:
    msg:
      - "[PATCH] {{ inventory_hostname }} - package update completed"
      - "Changed: {{ patch_result.changed }}"
5. Show reboot progress

This is particularly useful because reboot can take several minutes.

Before reboot:

- name: REBOOT - Server requires reboot
  ansible.builtin.debug:
    msg: >
      [REBOOT] {{ inventory_hostname }}
      - reboot required. Rebooting server...
  when:
    - ansible_facts.os_family == "RedHat"
    - rhel_reboot.rc | default(0) == 1

Then:

- name: REBOOT - Reboot server
  ansible.builtin.reboot:
    msg: "Ansible initiated reboot after Linux patching"
    reboot_timeout: 900
    connect_timeout: 10
    pre_reboot_delay: 5
    post_reboot_delay: 30
  when:
    - ansible_facts.os_family == "RedHat"
    - rhel_reboot.rc | default(0) == 1

Afterward:

- name: REBOOT - Server is back online
  ansible.builtin.debug:
    msg: >
      [REBOOT] {{ inventory_hostname }}
      - server is back online
6. Show post-check progress
- name: POST-CHECK - Starting
  ansible.builtin.debug:
    msg: >
      [POST-CHECK] {{ inventory_hostname }}
      - starting post-patching validation

Then:

- name: POST-CHECK - Verify connectivity
  ansible.builtin.ping:

Kernel:

- name: POST-CHECK - Get kernel
  ansible.builtin.command: uname -r
  register: kernel_after
  changed_when: false

Finally:

- name: POST-CHECK - SUCCESS
  ansible.builtin.debug:
    msg:
      - "=============================================="
      - "SUCCESS: {{ inventory_hostname }}"
      - "Kernel: {{ kernel_after.stdout }}"
      - "=============================================="
7. Jenkins should show the Ansible output

Your Jenkins stage can simply execute:

stage('Linux Patching') {

    steps {

        sshagent(credentials: ['ansible-vm-ssh']) {

            sh """
                ssh \
                  -o StrictHostKeyChecking=no \
                  ${ANSIBLE_USER}@${ANSIBLE_HOST} \
                  "cd ${REMOTE_DIR} && \
                   ansible-playbook \
                   -i ansible/inventory/production.ini \
                   ansible/playbooks/patch.yml \
                   --limit ${TARGET}"
            """
        }
    }
}

The output from Ansible is streamed into the Jenkins console.

So Jenkins will show something like:

[Pipeline] stage
[Pipeline] { (Linux Patching)

PLAY [Linux OS Patching] **********************************************

TASK [Gather facts] ***************************************************
ok: [server01]

TASK [Display server information] *************************************
ok: [server01] =>
  OS: RedHat
  Version: 9.6

TASK [PRE-CHECK] ******************************************************
ok: [server01] =>
  [PRE-CHECK] server01 - checking disk space

TASK [Check disk] *****************************************************
ok: [server01]

TASK [PRE-CHECK result] ***********************************************
ok: [server01] =>
  disk usage 42%

TASK [PATCH - Update packages] ***************************************
changed: [server01]

TASK [PATCH - Package update completed] ******************************
ok: [server01] =>
  Package update completed

TASK [REBOOT - Server requires reboot] *******************************
ok: [server01] =>
  Reboot required

TASK [REBOOT - Reboot server] ****************************************
changed: [server01]

TASK [REBOOT - Server is back online] *******************************
ok: [server01]

TASK [POST-CHECK - Verify connectivity] ******************************
ok: [server01]

TASK [POST-CHECK - SUCCESS] *******************************************
ok: [server01] =>
  SUCCESS: server01

PLAY RECAP ************************************************************
server01 : ok=15 changed=2 failed=0

Then:

TASK [Gather facts]
ok: [server02]

...

SUCCESS: server02
8. Add a progress counter

If you have 20 servers, it is useful to display:

Patching server 4 of 20

You can calculate this in Ansible.

At the beginning:

- name: Calculate server number
  ansible.builtin.set_fact:
    server_number: "{{ ansible_play_hosts_all.index(inventory_hostname) + 1 }}"

Then:

- name: Show progress
  ansible.builtin.debug:
    msg: >
      [PROGRESS]
      Patching server {{ server_number }}
      of {{ ansible_play_hosts_all | length }}:
      {{ inventory_hostname }}

Output:

[PROGRESS] Patching server 1 of 20: server01

Then:

[PROGRESS] Patching server 2 of 20: server02

etc.

9. Add an overall progress percentage

You can also show:

[PROGRESS] 25% - server05

Use:

- name: Calculate progress
  ansible.builtin.set_fact:
    server_number: "{{ ansible_play_hosts_all.index(inventory_hostname) + 1 }}"
    total_servers: "{{ ansible_play_hosts_all | length }}"

- name: Display progress
  ansible.builtin.debug:
    msg: >
      [PROGRESS]
      {{ ((server_number | int) / (total_servers | int) * 100) | round(0) }}%
      - Processing {{ inventory_hostname }}
      ({{ server_number }}/{{ total_servers }})

For 10 servers:

[PROGRESS] 10% - Processing server01 (1/10)
[PROGRESS] 20% - Processing server02 (2/10)
[PROGRESS] 30% - Processing server03 (3/10)
...
[PROGRESS] 100% - Processing server10 (10/10)
10. Better Jenkins stage names

You can also make Jenkins itself show stages:

stage('1. Git Checkout') {
    steps {
        checkout scm
    }
}

stage('2. Deploy Ansible Code') {
    steps {
        // rsync to Ansible VM
    }
}

stage('3. Pre-Checks') {
    steps {
        // ansible pre-check
    }
}

stage('4. Approval') {
    steps {
        input message: 'Proceed with patching?'
    }
}

stage('5. Linux Patching') {
    steps {
        // ansible-playbook
    }
}

stage('6. Post-Checks') {
    steps {
        // validation
    }
}

Jenkins then shows:

✓ 1. Git Checkout
✓ 2. Deploy Ansible Code
✓ 3. Pre-Checks
✓ 4. Approval
▶ 5. Linux Patching
   ├── server01 ✓
   ├── server02 ✓
   ├── server03 ▶
   ├── server04 -
   └── server05 -
○ 6. Post-Checks
11. One important consideration

If you want a real graphical progress bar such as:

Linux Patching

████████████░░░░░░░░ 60%

6 / 10 servers completed

server01  ✓
server02  ✓
server03  ✓
server04  ✓
server05  ✓
server06  ✓
server07  ▶ Patching
server08  -
server09  -
server10  -

that's different from normal Jenkins console output. You'd want either a Jenkins plugin/UI approach or a small reporting mechanism that updates build status.

For your current architecture, I recommend first implementing serial patching + [PROGRESS] x/y messages + clear pre-check/patch/reboot/post-check messages. It gives you very good visibility without adding another component.

If you want, I can give you a 
complete copy/paste project with the final Jenkinsfile, patch.yml, inventory.ini, and ansible.cfg that produces exactly this progress output for RHEL servers.

i want to show progress status while applying patching  to execute playbook

Yes. If you want the Jenkins console to show live progress while the Ansible playbook is actually running, don't hide the ansible-playbook output. Stream the SSH command directly to Jenkins and make Ansible process servers one at a time.

For your setup:

GitHub
   ↓
Jenkins VM
   │
   │ SSH
   ▼
Ansible VM
   │
   │ Ansible SSH
   ▼
Linux servers

The Jenkins console can show:

[PRECHECK] server01
[PATCH]   server01 - Updating packages
[PATCH]   server01 - 37 packages updated
[REBOOT]  server01 - Reboot required
[REBOOT]  server01 - Server back online
[POST]    server01 - SUCCESS

[PRECHECK] server02
[PATCH]   server02 - Updating packages
...
1. Use serial: 1

In patch.yml:

---
- name: Linux Patching
  hosts: linux_servers
  become: true

  serial: 1

  tasks:

    - name: Show server being patched
      ansible.builtin.debug:
        msg: "========== PATCHING {{ inventory_hostname }} =========="

    - name: Pre-check - ping
      ansible.builtin.ping:

    - name: Pre-check - disk space
      ansible.builtin.shell: |
        df -P / | awk 'NR==2 {print $5}'
      register: disk_usage
      changed_when: false

    - name: Show disk usage
      ansible.builtin.debug:
        msg: "[PRECHECK] {{ inventory_hostname }} disk usage: {{ disk_usage.stdout }}"

    - name: Patch RHEL server
      ansible.builtin.dnf:
        name: "*"
        state: latest
        update_only: true
      register: patch_result
      when: ansible_facts.os_family == "RedHat"

    - name: Show patch result
      ansible.builtin.debug:
        msg:
          - "[PATCH] {{ inventory_hostname }} completed"
          - "Changed: {{ patch_result.changed | default(false) }}"

    - name: Check reboot requirement
      ansible.builtin.command:
        cmd: needs-restarting -r
      register: reboot_required
      failed_when: false
      changed_when: false
      when: ansible_facts.os_family == "RedHat"

    - name: Show reboot status
      ansible.builtin.debug:
        msg: >
          [REBOOT CHECK] {{ inventory_hostname }} -
          {{
            'REBOOT REQUIRED'
            if reboot_required.rc | default(0) == 1
            else 'REBOOT NOT REQUIRED'
          }}

    - name: Reboot server
      ansible.builtin.reboot:
        msg: "Ansible reboot after patching"
        reboot_timeout: 900
        post_reboot_delay: 30
      when:
        - ansible_facts.os_family == "RedHat"
        - reboot_required.rc | default(0) == 1

    - name: Server is back online
      ansible.builtin.debug:
        msg: "[REBOOT] {{ inventory_hostname }} is back online"

    - name: Post-check - ping
      ansible.builtin.ping:

    - name: Post-check - kernel
      ansible.builtin.command: uname -r
      register: kernel
      changed_when: false

    - name: Post-check - SUCCESS
      ansible.builtin.debug:
        msg:
          - "=========================================="
          - "[SUCCESS] {{ inventory_hostname }}"
          - "Kernel: {{ kernel.stdout }}"
          - "=========================================="
2. Make Jenkins stream the output

This part is important.

Your Jenkinsfile should not redirect the output to a file.

Use:

stage('Execute Ansible Playbook') {

    steps {

        sshagent(credentials: ['ansible-vm-ssh']) {

            sh """
                ssh \
                  -o StrictHostKeyChecking=no \
                  ${ANSIBLE_USER}@${ANSIBLE_HOST} \
                  "cd ${REMOTE_DIR} && \
                   ansible-playbook \
                   -i ansible/inventory/production.ini \
                   ansible/playbooks/patch.yml \
                   --limit ${TARGET}"
            """
        }
    }
}

The Jenkins sh step streams stdout/stderr from SSH into the Jenkins build console.

3. Jenkins console will show live output

For example, if you have 3 servers:

[Pipeline] stage
[Pipeline] { (Execute Ansible Playbook)

PLAY [Linux Patching] ***************************************

TASK [Show server being patched] ****************************
ok: [server01] =>
  [PATCHING server01]

TASK [Pre-check - ping] ************************************
ok: [server01]

TASK [Pre-check - disk space] *******************************
ok: [server01]

TASK [Show disk usage] **************************************
ok: [server01] =>
  [PRECHECK] server01 disk usage: 42%

TASK [Patch RHEL server] ************************************
changed: [server01]

TASK [Show patch result] ************************************
ok: [server01] =>
  [PATCH] server01 completed
  Changed: true

TASK [Check reboot requirement] *****************************
ok: [server01]

TASK [Show reboot status] ***********************************
ok: [server01] =>
  [REBOOT CHECK] server01 - REBOOT REQUIRED

TASK [Reboot server] ****************************************
changed: [server01]

TASK [Server is back online] ********************************
ok: [server01] =>
  [REBOOT] server01 is back online

TASK [Post-check - ping] ************************************
ok: [server01]

TASK [Post-check - SUCCESS] *********************************
ok: [server01] =>
  [SUCCESS] server01
  Kernel: 5.14.0-...

TASK [Show server being patched] ****************************
ok: [server02]

...

That's live progress while the playbook is executing.

4. Don't use -q

Make sure you're not doing something like:

ansible-playbook ... -q

You want normal Ansible output.

You can also explicitly increase verbosity:

ansible-playbook \
  -i ansible/inventory/production.ini \
  ansible/playbooks/patch.yml \
  --limit ${TARGET} \
  -v

For normal production use, I would stay with no -v unless you're troubleshooting.

5. Add a server progress counter

If you have 20 servers, you can display:

[PROGRESS] 1/20 - server01
[PROGRESS] 2/20 - server02
[PROGRESS] 3/20 - server03

Add this near the beginning of the play:

- name: Calculate progress
  ansible.builtin.set_fact:
    server_number: "{{ ansible_play_hosts_all.index(inventory_hostname) + 1 }}"
    total_servers: "{{ ansible_play_hosts_all | length }}"

- name: Show progress
  ansible.builtin.debug:
    msg: >
      [PROGRESS]
      {{ server_number }}/{{ total_servers }}
      - Starting {{ inventory_hostname }}

Then Jenkins displays:

[PROGRESS] 1/20 - Starting server01
...
[SUCCESS] server01

[PROGRESS] 2/20 - Starting server02
...
[SUCCESS] server02

[PROGRESS] 3/20 - Starting server03
6. Add percentage

You can make it:

[PROGRESS] 10% - server01 - 1/10
[PROGRESS] 20% - server02 - 2/10
[PROGRESS] 30% - server03 - 3/10

Use:

- name: Calculate progress
  ansible.builtin.set_fact:
    server_number: "{{ ansible_play_hosts_all.index(inventory_hostname) + 1 }}"
    total_servers: "{{ ansible_play_hosts_all | length }}"

- name: Show patching progress
  ansible.builtin.debug:
    msg: >
      [PROGRESS]
      {{ ((server_number | int) / (total_servers | int) * 100) | round(0) }}%
      - {{ inventory_hostname }}
      - {{ server_number }}/{{ total_servers }}
7. Recommended output

I'd make your Jenkins output look like this:

============================================================
              LINUX PATCHING STARTED
============================================================

Total Servers: 10

[PROGRESS] 10% - server01 - 1/10

[PRECHECK] server01
  ✓ Connectivity
  ✓ Disk space: 42%
  ✓ Uptime

[PATCH] server01
  ▶ Updating packages
  ✓ Package update completed

[REBOOT] server01
  ✓ Reboot required
  ▶ Rebooting
  ✓ Server back online

[POSTCHECK] server01
  ✓ Connectivity
  ✓ Kernel
  ✓ Disk
  ✓ Uptime

[SUCCESS] server01

------------------------------------------------------------

[PROGRESS] 20% - server02 - 2/10

...

The key pieces are:

serial: 1

in Ansible, plus:

ssh ... "ansible-playbook ..."

in Jenkins without redirecting stdout.

That gives you live playbook progress directly in the Jenkins console while the patching is happening.

Ad
Ads do not influence the answers you get from ChatGPT. Your chats stay private. 
Learn about ads and personalization
Today 5:59 PM
status code 128:
stdout:
stderr: Load key "/tmp/jenkins-gitclient-ssh2773604965954661025.key": error in libcrypto
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.


This error is happening before Jenkins reaches your Ansible VM. Jenkins is failing at the GitHub SSH authentication / private-key loading step.

The important line is:

Load key "/tmp/jenkins-gitclient-ssh....key": error in libcrypto

followed by:

git@github.com: Permission denied (publickey).

Most commonly, the private key stored in Jenkins is malformed, is in an unsupported format, or has Windows/CRLF line endings.

1. First check the key on the Jenkins VM

If your Jenkins VM is Linux, find the key you're using for GitHub:

sudo -u jenkins ls -la /var/lib/jenkins/.ssh/

Then test the private key:

sudo -u jenkins ssh-keygen \
  -y \
  -f /var/lib/jenkins/.ssh/github_key

If the key is valid, this should output a public key.

If you get:

Load key "...": error in libcrypto

then the key itself/format is the problem.

2. Check the beginning and end of the private key

Run:

sudo head -n 1 /var/lib/jenkins/.ssh/github_key

and:

sudo tail -n 1 /var/lib/jenkins/.ssh/github_key

For a modern Ed25519 key, you should see:

-----BEGIN OPENSSH PRIVATE KEY-----

and:

-----END OPENSSH PRIVATE KEY-----

If you see something like:

-----BEGIN RSA PRIVATE KEY-----

that's also usable, but make sure the entire key is intact.

There must not be extra text such as:

Private-Key:
Comment:

inside the key.

3. Check for Windows line endings

This is a very common cause if you created/copied the key on Windows.

Run:

file /var/lib/jenkins/.ssh/github_key

If you see:

with CRLF line terminators

convert it:

sudo sed -i 's/\r$//' /var/lib/jenkins/.ssh/github_key

Then:

sudo chmod 600 /var/lib/jenkins/.ssh/github_key
sudo chown jenkins:jenkins /var/lib/jenkins/.ssh/github_key

Test again:

sudo -u jenkins ssh-keygen \
  -y \
  -f /var/lib/jenkins/.ssh/github_key
4. Easiest solution: generate a new key on Jenkins

If you're not sure about the existing key, I recommend generating a fresh key.

On the Jenkins VM:

sudo -u jenkins mkdir -p /var/lib/jenkins/.ssh

Then:

sudo -u jenkins ssh-keygen \
  -t ed25519 \
  -C "jenkins-github" \
  -f /var/lib/jenkins/.ssh/github_key

When prompted for a passphrase, you can leave it empty if this is a dedicated Jenkins automation key.

You'll get:

/var/lib/jenkins/.ssh/github_key
/var/lib/jenkins/.ssh/github_key.pub
5. Add the public key to GitHub

Display it:

sudo cat /var/lib/jenkins/.ssh/github_key.pub

It should look like:

ssh-ed25519 AAAAC3... jenkins-github

In GitHub:

Repository
   ↓
Settings
   ↓
Deploy keys
   ↓
Add deploy key

Give it a name such as:

Jenkins VM

Paste the public key.

For checkout only, leave Allow write access disabled.

6. Test GitHub directly from Jenkins

This is important. Don't test through Jenkins yet.

Run:

sudo -u jenkins ssh \
  -i /var/lib/jenkins/.ssh/github_key \
  -o IdentitiesOnly=yes \
  -T git@github.com

GitHub should respond with something similar to:

Hi! You've successfully authenticated, but GitHub does not provide shell access.

If this fails, Jenkins will also fail.

7. Test the actual repository

Now run:

sudo -u jenkins git ls-remote \
  git@github.com:YOUR_ORG/linux-patching.git

If your repository is, for example:

git@github.com:ramesh/linux-patching.git

then:

sudo -u jenkins git ls-remote \
  git@github.com:ramesh/linux-patching.git

You should see:

abc123...        HEAD
abc123...        refs/heads/main

Once this works, GitHub connectivity is good.

8. Fix the Jenkins credential

Now go to:

Jenkins
 → Manage Jenkins
 → Credentials
 → Global
 → github-ssh

Make sure the credential is:

Kind:
SSH Username with private key

Username:
git

ID:
github-ssh

Private Key:
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----

Don't paste the .pub file.

You need the private key:

github_key

not:

github_key.pub
9. Verify your Jenkins Git URL

For SSH authentication, use:

git@github.com:YOUR_ORG/linux-patching.git

Not:

https://github.com/YOUR_ORG/linux-patching.git

Your Jenkins configuration should look like:

SCM:
Git

Repository URL:
git@github.com:YOUR_ORG/linux-patching.git

Credentials:
github-ssh

Branch:
*/main

Script Path:
Jenkinsfile
10. Important: don't confuse your three SSH keys

For your architecture, you should have three separate authentication relationships:

                  GitHub
                     ▲
                     │
              github-ssh key
                     │
                Jenkins VM
                     │
                     │ ansible-vm-ssh key
                     ▼
                Ansible VM
                     │
                     │ linux_patch_key
                     ▼
              Linux servers

So:

Key	Purpose	Location
github_key	Jenkins → GitHub	Jenkins credential
ansible_vm_key	Jenkins → Ansible VM	Jenkins credential
linux_patch_key	Ansible VM → Linux	Ansible VM

Don't use the same private key for all three.

11. If you are using a Jenkins "SSH Credentials" entry

The /tmp/jenkins-gitclient-ssh...key in your error is a clue that Jenkins did retrieve the credential and created a temporary key file.

So the problem is likely the contents/format of the Jenkins GitHub private-key credential, rather than the Jenkins Git plugin itself.

I would do this:

1. Generate fresh Ed25519 key on Jenkins VM
                 ↓
2. Add .pub to GitHub Deploy Keys
                 ↓
3. Test ssh -T manually as jenkins
                 ↓
4. Test git ls-remote manually as jenkins
                 ↓
5. Replace Jenkins github-ssh credential
                 ↓
6. Run Jenkins checkout

If step 3 works but Jenkins still gives error in libcrypto, then we should inspect the Jenkins credential configuration / Git client setup, rather than regenerating keys again.
