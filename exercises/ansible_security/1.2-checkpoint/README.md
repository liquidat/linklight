# Exercise 1.2 - Executing the first Check Point playbook

## Step 2.1 - Check Point Next Generation Firewall

To showcase how to automate the firewall in a security environment, this lab contains a Check Point Next Generaion Firewall (NGFW).

The NGFW is usually not managed directly, but via a central security management server (MGMT). The MGMT is a central tool to manage multiple NGFWs or other security tools in one spot.

There are multiple ways to interact with the MGMT. In our lab, two ways are important:

- API: Ansible mostly works with the API
- Windows client: the user interaction takes place in a Windows client.

In this lab the playbooks we write will be interacting with the API in the background. All actions will be verified in the Windows client UI.

## Step 2.2 - Accessing the Check Point MGMT server via a Windows workstation

Since access to the MGMT server requires a Windows client and we cannot be sure that each and every lab student has access to a Windows environment, we have provisioned a Windows workstation as part of this lab.

The Windows workstation can be reached via Remote Desktop Protocol (RDP). We recommend to use a native RDP client if available. If not, the Workstation is equipped with an HTML RDP client which enables lab participants to access the workstation via browser.

Test the access to the MGMT server now by pointing your RDP client to the `windows-ws` IP in your inventory.

If you do not have a RDP client available or want to test the HTML RDP client, please open the following URL in your browser: `http://<windows-wsIP>/myrtille`. Be sure to replace `<windows-wsIP>` with the IP for the Windows workstation from your inventory. In the login field, only provide the user name and the password: The user name is **Administrator**, the password is **Ansible+Red*Hat19!20** if not provided otherwise.

Upon first login there might be a message about a network interface in a blue bar on the right side. This message can be safely ignored and will disappear as soon as you click anywhere.

You now are accessing a default windows workstation with a Google Chrome browser installed.

> **Note**
>
> Directly after the login you might see a wide blue bar on the right side of the screen, about network configurations. You can safely ignore this, the question hides away if you click anywhere on the screen.

## Step 2.3 - Install SmartConsole

SmartConsole should already be installed on your system. Please check your desktop for an icon to launch SmartConsole and launch it. If this works, the following tasks are **not necessary** and you can proceed to **Step 2.4**!

If for any reason SmartConsole was not installed properly during the deployment of the lab, it is simple to do that yourself:

- Inside your Windows workstation, open the Chrome browser
- Point the browser to `https://<checkpointIP>`, where `<checkpointIP>` is the IP for the checkpoint entry in your inventory
- A warning page will open since Check Point MGMT server is by default installed with a self signed certificate. Accept the certificate by clicking on **Advanced** and afterwards by clicking on the link named **Proceed to 11.22.33.44 (unsafe)**, with your `checkpoint` IP instead of **11.22.33.44**
- Login with user name `admin` and password `admin123`
- Accept the message of the day with a click on the **Ok** button
- On top of the page, click on the green **Download Now!** button
- The download starts immediately, the file is downloaded to the **Downloads** folder of the administrator
- Find the file and launch the installer via double click
- Accept all default values and finish the installation

## Step 2.4 - Access the SmartConsole UI

Launch the Check Point SmartConsole via the desktop icon. In the following window, as username use `admin` and as password `admin123` if not instructed otherwise. The IP address to enter is the one from the **checkpoint** entry of your inventory.

![SmartConsole login window](images/smartconsole-login-window.png)

Press the **Login** button. Afterwards you need to verify the server fingerprint by clicking the **PROCEED** button.

> **Note**
>
> In a production environment, you would first figure out the fingerprint of the server and would only proceed after you confirmed that the fiungerprint shown is identical with the one from the server. In our demo setup with the short lived instances we can assume that the fingerprints are good.

You are now viewing the Check Point SmartConsole management interface. There might be a Internet Explorer Warning visible upon start. This can safely be closed and is due to limitations in the way IE works.

![SmartConsole main window](images/smartconsole-main-window.png)

Next, on the left side, click on **SECURITY POLICIES** and note that there is currently only one rule installed: to drop all traffic. Now you have a first idea of how Check Point looks like in term of the management interface. We will interact more with it - but first we go back to the command line to learn how to write Ansible playbooks interacting with Check Point.

## Step 2.5 - First example playbook

In Ansible, automation is described in playbooks. Playbooks are files which describe the desired configurations or steps to implement on managed hosts. Playbooks can change lengthy, complex administrative tasks into easily repeatable routines with predictable and successful outcomes.

A playbook is a repeatable set of *plays* and *tasks*.

A playbook can have multiple plays and a play can have one or multiple tasks. In a task a *module* is called, like the modules in the previous chapter. The goal of a *play* is to map a group of hosts.  The goal of a *task* is to implement modules against those hosts.

If you are not very familiar with Ansible, see the following example of a playbook:

```yaml
---
- name: install and start apache
  hosts: web
  become: yes
  vars:
    http_port: 80

  tasks:
    - name: httpd package is present
      yum:
        name: httpd
        state: latest

    - name: latest index.html file is present
      template:
        src: files/index.html
        dest: /var/www/html/

    - name: httpd is started
      service:
        name: httpd
        state: started
```

> **Tip**
>
> Here is a nice analogy: When Ansible modules are the tools in your workshop, the inventory is the materials and the playbooks are the instructions.

We will now write a playbook to change the configuration of the Check Point setup. We will start with a simple example where we will add a whilte entry in the firewall configuration to allow traffic from a certain machine to another. In our example we will allow the machine called **attacker** to send traffic to our machine **snort**.

The playbook will be written and run on the Ansible control host. Use SSH to access your control host. On there, open an editor of your choice and create a file with the name `whitelist_attacker.yml`.

First, a playbook needs a name and the hosts it should be executed on. So let's add those:

```yaml
---
- name: Whitelist Attacker
  hosts: checkpoint
```

> **Note**
>
> It is a good practice to make playbooks more reusable by pointing them at `hosts: all` and limit the execution later on the command line or via Tower. But for now we simplify the process by naming hosts in the playbook directly.

As mentioned, in this a simple example we will add a whitelist entry. A simple whitelist entry consists of a source IP address, a destination IP address and the rule to prevent access between those.

For this, we add the source and destination IP as variables to the playbook. Since Ansible knows all the machines from the inventory and since the IPs are listed in the inventory, we can just reference those information as variables:

<!-- {% raw %} -->
```yaml
---
- name: Whitelist Attacker
  hosts: checkpoint

  vars:
    source_ip: "{{ hostvars['attacker']['private_ip2'] }}"
    destination_ip: "{{ hostvars['snort']['private_ip2'] }}"
```
<!-- {% endraw %} -->

Note that we use the second private IP - those belong to a network whihc is specifically routed via the FW for application traffic. The first private IP belongs to the management network.

Next, we need to add the tasks where the actual changes on the target machines are done. This happens in three steps: first we create a source object, than a destination object, and finally the access rule between those two.

Let's start with a task to define the source object:

<!-- {% raw %} -->
```yaml
---
- name: Whitelist attacker
  hosts: checkpoint

  vars:
    source_ip: "{{ hostvars['attacker']['private_ip2'] }}"
    destination_ip: "{{ hostvars['snort']['private_ip2'] }}"

  tasks:
    - name: Create source IP host object
      checkpoint_host:
        name: "asa-{{ source_ip }}"
        ip_address: "{{ source_ip }}"
```
<!-- {% endraw %} -->

As you can see, the task itself has a name - just like the play itself - and references a module, here `checkpoint_hosts`. The module has parameters, here `name` and `ip_address`. Each module has individual parameters, often some of them are required while others are optional. To get more information about a module, you can call the help:

```bash
[student<X>@ansible ~]$ ansible-doc checkpoint_host
```

> **Tip**
>
> In `ansible-doc` leave by pressing the button `q`. Use the `up`/`down` arrows to scroll through the content.

In the same way we defined the source IP host object, we will now add the destination IP host object:

<!-- {% raw %} -->
```yaml
---
- name: Whitelist attacker
  hosts: checkpoint

  vars:
    source_ip: "{{ hostvars['attacker']['private_ip2'] }}"
    destination_ip: "{{ hostvars['snort']['private_ip2'] }}"

  tasks:
    - name: Create source IP host object
      checkpoint_host:
        name: "asa-{{ source_ip }}"
        ip_address: "{{ source_ip }}"

    - name: Create destination IP host object
      checkpoint_host:
        name: "asa-{{ destination_ip }}"
        ip_address: "{{ destination_ip }}"
```
<!-- {% endraw %} -->

Last, we are defining the actual access rule between those two host objects:

<!-- {% raw %} -->
```yaml
---
- name: Whitelist attacker
  hosts: checkpoint

  vars:
    source_ip: "{{ hostvars['attacker']['private_ip2'] }}"
    destination_ip: "{{ hostvars['snort']['private_ip2'] }}"

  tasks:
    - name: Create source IP host object
      checkpoint_host:
        name: "asa-{{ source_ip }}"
        ip_address: "{{ source_ip }}"

    - name: Create destination IP host object
      checkpoint_host:
        name: "asa-{{ destination_ip }}"
        ip_address: "{{ destination_ip }}"

    - name: Create access rule to allow access from source to destination
      checkpoint_access_rule:
        auto_install_policy: yes
        auto_publish_session: yes
        layer: Network
        position: top
        name: "asa-accept-{{ source_ip }}-to-{{ destination_ip }}"
        source: "asa-{{ source_ip }}"
        destination: "asa-{{ destination_ip }}"
        action: accept

    - name: Install policy
      cp_mgmt_install_policy:
        policy_package: standard
        install_on_all_cluster_members_or_fail: yes
      failed_when: false
```
<!-- {% endraw %} -->

The last task ensures that the policy is installed in any case. Since sometimes this task fails if another installations is already running we ignore possible errors here.

## Step 2.6 - Run the playbook

Playbooks are executed using the `ansible-playbook` command on the control node. Before you run a new playbook it’s a good idea to check for syntax errors:

```bash
[student<X>@ansible ansible-files]$ ansible-playbook --syntax-check whitelist_attacker.yml
```

Now you should be ready to run your playbook:

```bash
[student<X>@ansible ansible-files]$ ansible-playbook whitelist_attacker.yml

PLAY [Whitelist attacker] *********************************************************

TASK [Gathering Facts] ************************************************************
ok: [checkpoint]

TASK [Create source IP host object] ***********************************************************************************
changed: [checkpoint]

TASK [Create destination IP host object] ***********************************************************************************
changed: [checkpoint]

TASK [Create access rule to allow access from source to destination] ***********************************************************************************
changed: [checkpoint]

PLAY RECAP ************************************************************************
checkpoint  : ok=4 changed=3 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

## Step 2.7 - Verify changes in UI

Now it's time to check if the changes really did take place, if the actual Check Point MGMT server configuration was really altered.

Access the Windows workstation and open the SmartConsole interface. On the right side, underneath **Object Categories**, click on **Network Objects**, then pick **Hosts**. It should list both new host entries.

![SmartConsole Hosts list](images/smartconsole-hosts-list.png)

Next, on the left side, click on **SECURITY POLICIES** and note the additional access control policy entry in the middle of the field compared to the first time we looked at this. This time it allows traffic, thus has another entry in the **Action** column and also a different color.

![SmartConsole Policy Entries](images/smartconsole-policy-entry.png)

Also note in the bottom left corner that there is a green bar indicating that changes were applied to the entire system.

## Step 2.8 - Turn on Logging for the new policy

To see how changes are usually performed in a typical interaction with Check Point in contrast, let's just do a small change which will come in handy later on: by default, Check Point does not turn on logging for new rules. Let's activate the logging for our new policy. On the left side of the main window, click on **SECURITY POLICIES**. There are both rules listed. In the column **Track**, hover with your mouse over the **None** entry of our newly created rule. Right click on it, and in the box appearing pick **Log**.

![SmartConsole, change logging](images/smartconsole-change-logging.png)

Afterwards, click on the **Install Policy** button at the top of the list of policies, confirm the dialog which opens with **Publish & Install** and in the last dialog, click **Install** again.

As a result, in the left corner a small window pops up informing you of the progress of the deployment of the change.

As you see, even doing a rather small change the configuraiton required multiple clicks and interactions with the user - the more of these steps can be automated, the better.
----

[Click Here to return to the Ansible Security Automation Workshop](../README.md#section-1---introduction-to-ansible-security-automation-basics)
