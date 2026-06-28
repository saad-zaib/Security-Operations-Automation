# Security Operations Automation Lab

**Credit:** Big thanks to the [DFIR YouTube channel](https://www.youtube.com/@mydfir) — their tutorials were a huge help in putting together this lab.

## 1. Introduction

### 1.1 Overview
This project builds an automated SOC pipeline that handles event monitoring, alerting, and response with minimal manual work. It combines Wazuh (event collection/alerting), Shuffle (workflow automation/SOAR), and TheHive (case management) on top of a Windows 10 client running Sysmon for rich event telemetry.

![SOC Automation Diagram](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/SOC_Automation_Diagram.png)

### 1.2 Goals
- **Real-time collection & analysis** — catch threats proactively instead of manually digging through logs.
- **Automated alerting** — route alerts to the right place fast, with less chance of something slipping through.
- **Automated response actions** — react to incidents consistently and quickly.
- **Less manual SOC overhead** — free analysts to focus on real investigations instead of repetitive tasks.

## 2. Prerequisites

### 2.1 Hardware
- A host machine that can run several VMs at once.
- Enough CPU/RAM/disk to handle the workload comfortably.

### 2.2 Software
- **VMware Workstation/Fusion** — for the VMs.
- **Windows 10** — generates the security events we'll test against.
- **Ubuntu 22.04** — hosts Wazuh and TheHive.
- **Sysmon** — detailed Windows telemetry/logging.

### 2.3 Tools/Platforms
- **Wazuh** — core SIEM: collection, detection, alerting.
- **Shuffle** — SOAR automation for alert handling and response.
- **TheHive** — incident/case management platform.
- **VirusTotal** — multi-engine file/URL reputation checks.
- **Cloud or extra VMs** — Wazuh and TheHive can run on cloud infra or local VMs, whatever you've got available.

### 2.4 Assumed Knowledge
- Comfortable spinning up and managing VMs.
- Basic Linux CLI (installing packages, managing services).
- General familiarity with SOC concepts: monitoring, logging, IR basics.

## 3. Setup

### 3.1 Step 1 — Windows 10 + Sysmon

**3.1.1 Install Windows 10 in VMware**

![Windows 10 Installation](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603131110.png)

**3.1.2 Grab Sysmon**

![Sysmon Installation](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603131150.png)

**3.1.3 Grab a Sysmon config from [sysmon-modular](https://github.com/olafhartong/sysmon-modular)**

![Sysmon Modular Config](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603131815.png)
![Sysmon Modular Config Files](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603132002.png)

**3.1.4 Unzip Sysmon, open PowerShell as admin, cd into the extracted folder**

![Extract Sysmon Zip](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603133020.png)

**3.1.5 Drop the config file into that same Sysmon folder.**

**3.1.6 Check it's not already installed** — look in Services, and in Event Viewer → Applications and Services Logs → Microsoft → Windows.

![Check Sysmon Installation](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603133433.png)

**3.1.7 Install it:**

```
.\Sysmon64.exe -i .\sysmonconfig.xml
```

![Install Sysmon](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603154817.png)

**3.1.8 Confirm it's running**

![Verify Sysmon Installation](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603154953.png)

Windows client + Sysmon done. Next: Wazuh.

### 3.2 Step 2 — Wazuh Server

**3.2.1 Spin up a Droplet on DigitalOcean** (any cloud/VM works fine)

![Create Droplet](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603215218.png)

Pick Ubuntu 22.04:

![Select Ubuntu](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603220120.png)

Set a root password, name it "Wazuh", create it:

![Create Wazuh Droplet](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603220521.png)

**3.2.2 Lock it down with a firewall** — Networking → Firewall → Create Firewall:

![Create Firewall](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603220742.png)

Restrict inbound to just your IP:

![Set Inbound Rules](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603220920.png)

Attach the firewall to the Wazuh droplet:

![Apply Firewall](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603221926.png)
![Firewall Protection](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603222113.png)

**3.2.3 SSH in** via Droplets → Wazuh → Access → Launch Droplet Console:

![Launch Droplet Console](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603223020.png)

**3.2.4 Update the box:**
```
sudo apt-get update && sudo apt-get upgrade
```

**3.2.5 Install Wazuh:**
```
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

![Wazuh Installation](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603224933.png)

Save the generated admin password it prints out:
```
User: admin
Password: *******************
```

**3.2.6 Log into the Wazuh dashboard** by browsing to the droplet's public IP over `https://`:

![Wazuh Login](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603225355.png)

Click through the self-signed cert warning ("Proceed" → "Continue"):

![Wazuh Login Continue](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603225424.png)

Log in as `admin` with the generated password:

![Wazuh Dashboard](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603225621.png)

Client + Wazuh server are live. Next up: TheHive.

### 3.3 Step 3 — Install TheHive

**3.3.1 New droplet for TheHive** (Ubuntu 22.04, same firewall rules applied):

![Create TheHive Droplet](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603230036.png)

**3.3.2 Install dependencies:**
```
apt install wget gnupg apt-transport-https git ca-certificates ca-certificates-java curl software-properties-common python3-pip lsb-release
```

![Install Dependencies](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603230509.png)

**3.3.3 Install Java:**
```
wget -qO- https://apt.corretto.aws/corretto.key | sudo gpg --dearmor -o /usr/share/keyrings/corretto.gpg
echo "deb [signed-by=/usr/share/keyrings/corretto.gpg] https://apt.corretto.aws stable main" | sudo tee -a /etc/apt/sources.list.d/corretto.sources.list
sudo apt update
sudo apt install java-common java-11-amazon-corretto-jdk
echo JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto" | sudo tee -a /etc/environment
export JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto"
```

**3.3.4 Install Cassandra** (TheHive's database):
```
wget -qO - https://downloads.apache.org/cassandra/KEYS | sudo gpg --dearmor -o /usr/share/keyrings/cassandra-archive.gpg
echo "deb [signed-by=/usr/share/keyrings/cassandra-archive.gpg] https://debian.cassandra.apache.org 40x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list
sudo apt update
sudo apt install cassandra
```

**3.3.5 Install Elasticsearch** (indexing/search):
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
sudo apt-get install apt-transport-https
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update
sudo apt install elasticsearch
```

**3.3.6 (Optional) Tune Elasticsearch** — create `/etc/elasticsearch/jvm.options.d/jvm.options`:
```
-Dlog4j2.formatMsgNoLookups=true
-Xms2g
-Xmx2g
```

**3.3.7 Install TheHive:**
```
wget -O- https://archives.strangebee.com/keys/strangebee.gpg | sudo gpg --dearmor -o /usr/share/keyrings/strangebee-archive-keyring.gpg
echo 'deb [signed-by=/usr/share/keyrings/strangebee-archive-keyring.gpg] https://deb.strangebee.com thehive-5.2 main' | sudo tee -a /etc/apt/sources.list.d/strangebee.list
sudo apt-get update
sudo apt-get install -y thehive
```

Default login (port 9000):
```
Username: admin@thehive.local
Password: secret
```

![TheHive Login](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603231319.png)

### 3.4 Step 4 — Configure TheHive & Wazuh

**3.4.1 Configure Cassandra:**
```
nano /etc/cassandra/cassandra.yaml
```

![Cassandra Configuration](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603231723.png)

Set `listen_address` to TheHive's public IP:

![Listen Address](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603232121.png)

Set the `rpc_address` the same way, then set `seeds` under `seed_provider` to TheHive's public IP too:

![Seed Provider](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603232508.png)

Stop Cassandra:
```
systemctl stop cassandra.service
```
Wipe the old data (fresh install via package manager):
```
rm -rf /var/lib/cassandra/*
```
Start it back up:
```
systemctl start cassandra.service
```
Check it's healthy:
```
systemctl status cassandra.service
```

![Cassandra Service Status](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603232813.png)

**3.4.2 Configure Elasticsearch:**
```
nano /etc/elasticsearch/elasticsearch.yml
```

Optional: rename the cluster. Uncomment `node.name`. Uncomment `network.host` and point it at TheHive's public IP.

![Elasticsearch Configuration](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603233522.png)

Optionally uncomment `http.port` (default 9200) and `cluster.initial_master_nodes` (drop `node-2` if you don't need it).

Start + enable it:
```
systemctl start elasticsearch
systemctl enable elasticsearch
```
Check status:
```
systemctl status elasticsearch
```

![Elasticsearch Service Status](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603233935.png)

**3.4.3 Configure TheHive:**

Check permissions on its install dir first:
```
ls -la /opt/thp
```

![TheHive Directory Permissions](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603234256.png)

Fix ownership if `root` owns it instead of the `thehive` user:
```
chown -R thehive:thehive /opt/thp
```

![Change TheHive Directory Permissions](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603234450.png)

Now edit TheHive's config:
```
nano /etc/thehive/application.conf
```

In the database/index sections: set `hostname` to TheHive's public IP, match `cluster.name` to your Cassandra cluster name (e.g. "Test Cluster"), set `index.search.hostname` to the public IP too, and at the bottom set `application.baseUrl` to the public IP.

Cortex and MISP integrations are enabled by default.

![TheHive Configuration](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603235441.png)

Start + enable:
```
systemctl start thehive
systemctl enable thehive
```

![TheHive Service Status](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603235616.png)

> If TheHive won't load, double check Cassandra, Elasticsearch, *and* TheHive are all running — it won't start otherwise.

Browse to it:
```
http://143.198.56.201:9000/login
```

![TheHive Login](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240603235840.png)

Login: `admin@thehive.local` / `secret`

![TheHive Dashboard](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604000101.png)

### 3.5 Step 5 — Configure Wazuh

**3.5.1 Add the Windows agent** — in the Wazuh UI, "Add agent" → Windows, point it at the Wazuh server's public IP:

![Add Wazuh Agent](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604000526.png)

Run the generated install command in PowerShell on the Windows client:

![Wazuh Agent Installation](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604002346.png)

Start the agent service (`net start wazuhsvc`, or via Services):

![Wazuh Agent Service](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604002550.png)
![Wazuh Agent Service Start](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604002624.png)

**3.5.2 Confirm it's connected** — should show "Active" in the Wazuh UI:

![Wazuh Agent Connected](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604002744.png)
![Wazuh Agent Status](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604002929.png)

You're now ready to query events from the agent.

## 4. Generating Telemetry & Custom Alerts

### 4.1 Forward Sysmon Events to Wazuh

**4.1.1** On the Windows client, open `C:\Program Files (x86)\ossec-agent\ossec.conf`:

![Ossec Configuration](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604144648.png)

**4.1.2** Add a `<localfile>` block for the Sysmon log (check its exact name in Event Viewer first):

![Sysmon Event Log](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604150516.png)
![Ossec Sysmon Configuration](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604150556.png)

Optional: you can also forward PowerShell/Application/Security/System logs — this lab strips those out to focus purely on Sysmon.

**4.1.3** Save with an admin-elevated Notepad (required for this file).

**4.1.4** Restart the Wazuh agent service to apply changes:

![Restart Wazuh Agent Service](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604151539.png)

> Any time you edit the agent config, restart the service.

**4.1.5** Confirm events are flowing — check "Events" in the Wazuh UI for Sysmon entries:

![Search Sysmon Events](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604152334.png)

### 4.2 Generate Mimikatz Telemetry

**4.2.1** Download Mimikatz on the Windows client (you'll likely need to exclude the folder from Defender):

![Disable Windows Defender](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604152850.png)

**4.2.2** Run it from PowerShell:

![Start Mimikatz](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604153518.png)

**4.2.3** By default Wazuh only logs events tied to a rule. To capture everything, edit the manager's config:
```
cp /var/ossec/etc/ossec.conf ~/ossec-backup.conf
```

![Wazuh Ossec Configuration](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604154130.png)

Flip `<logall>` and `<logall_json>` to "yes", then restart:
```
systemctl restart wazuh-manager.service
```
This dumps everything into `/var/ossec/logs/archives/`.

**4.2.4** Enable archive ingestion in Filebeat:
```
nano /etc/filebeat/filebeat.yml
```

![Filebeat Configuration](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604154620.png)

Set the "archives" input's `enabled` to `true`, restart Filebeat.

**4.2.5** Create an index for the archives in Wazuh — Stack Management → Index Management:

![Create Wazuh Index](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604154910.png)

Name it `wazuh-archives-*`:

![Wazuh Archives Index](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604155026.png)

Pick "timestamp" as the time field:

![Create Wazuh Archives Index](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604155124.png)

Then browse it under Discover:

![Discover Wazuh Archives](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604155204.png)

**4.2.6** Sanity-check the archive logs directly:
```
cat /var/ossec/logs/archives/archives.log | grep -i mimikatz
```

![Check Mimikatz Logs](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604155940.png)

Nothing here just means Mimikatz hasn't fired yet — that's expected at this point.

**4.2.7** Run Mimikatz again and confirm Sysmon picks it up:

![Mimikatz Sysmon Event](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604160828.png)

Re-check the archive log:

![Mimikatz Logs Generated](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604164746.png)
![Mimikatz Logs](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604164803.png)

### 4.3 Build a Custom Mimikatz Detection Rule

**4.3.1** Look at the captured logs and pick a reliable field — here we use `originalfilename`, since it survives the attacker renaming the binary:

![Mimikatz Original Filename](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604165046.png)

**4.3.2** You can write the rule via CLI or the web UI:

![Wazuh Rules](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604165457.png)

In the UI: "Manage rule files" → filter by "sysmon" → view an existing Event ID 1 rule as a template:

![Sysmon Rules](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604165717.png)

Custom rule:
```xml
<rule id="100002" level="15">
  <if_group>sysmon_event1</if_group>
  <field name="win.eventdata.originalFileName" type="pcre2">(?i)\\mimikatz\.exe</field>
  <description>Mimikatz Usage Detected</description>
  <mitre>
    <id>T1003</id>
  </mitre>
</rule>
```

Drop this into "Custom rules" → `local_rules.xml`:

![Custom Mimikatz Rule](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604171432png)
![Custom Mimikatz Rule Added](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604171857.png)

Save, restart the manager service.

**4.3.3** Test it — rename the Mimikatz executable to something else:

![Renamed Mimikatz](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604172430.png)

Run it:

![Mimikatz Started](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604172710.png)

Confirm the rule still fires despite the rename:

![Mimikatz Alert Triggered](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604173838.png)

## 5. Automation with Shuffle + TheHive

### 5.1 Set Up Shuffle

**5.1.1** Create an account at shuffler.io:

![Shuffle Account](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604213121.png)

**5.1.2** New workflow (any template is fine for now):

![Create Shuffle Workflow](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604213428.png)

**5.1.3** Add a Webhook trigger — drag it in, connect to "Change Me," name it, and copy the Webhook URI (you'll need this for Wazuh):

![Shuffle Webhook](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604213723.png)

**5.1.4** Set "Change Me" to "Repeat back to me," call option = "Execution argument." Save.

![Shuffle Workflow Settings](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/2024-06-04%2021_41_05-Workflow%20-%20SOC%20Automation%20Lab.png)

**5.1.5** On the Wazuh manager, add the Shuffle integration:
```
nano /var/ossec/etc/ossec.conf
```
```xml
<integration>
  <name>shuffle</name>
  <hook_url>https://shuffler.io/api/v1/hooks/webhook_0af8a049-f2cb-420b-af58-5ebc3c40c7df</hook_url>
  <level>3</level>
  <alert_format>json</alert_format>
</integration>
```
Swap `<level>` for `<rule_id>100002</rule_id>` to only fire on the Mimikatz rule.

![Wazuh Shuffle Integration](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604225725.png)

Restart:
```
systemctl restart wazuh-manager.service
```

**5.1.6** Trigger Mimikatz again, hit "Start" on the webhook in Shuffle, and confirm the alert lands there.

![Shuffle Webhook Start](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604230243.png)

### 5.2 Build the Full Mimikatz Workflow

**Flow:**
1. Mimikatz alert → Shuffle
2. Shuffle extracts the SHA256 hash from the alert
3. Hash gets checked against VirusTotal
4. Details get pushed to TheHive as an alert
5. SOC analyst gets an email

**5.2.1 Extract the SHA256 hash**

The raw hash field comes back prefixed (e.g. `sha1=hashvalue`), so it needs parsing before VirusTotal will accept it.

On "Change Me," switch to "Regex capture group," input = "hashes," regex = `SHA256=([0-9A-Fa-f]{64})`. Save.

![Shuffle Regex](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604234504.png)

Check the extraction worked via "Show execution":

![Extracted Hash Value](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604234729.png)

**5.2.2 Hook up VirusTotal**

Create a VT account for API access:

![VirusTotal Account](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240604235908.png)

In Shuffle, drag in the VirusTotal app from "Apps" — it auto-connects:

![VirusTotal App](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605000230.png)

Paste your API key (or use "Authenticate VirusTotal v3"):

![VirusTotal Authentication](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605000358.png)

Set "ID" to the SHA256Regex output from the previous step:

![VirusTotal ID](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605001826.png)

Save and rerun:

![VirusTotal Results](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605001925.png)

Expand to see detection counts:

![VirusTotal Detection](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605002209.png)

**5.2.3 Hook up TheHive**

Drag the TheHive app into the workflow, connect via its public IP + port 9000:

![TheHive Login](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605112834.png)

Default login: `admin@thehive.local` / `secret`

**5.2.4 Configure TheHive users**

Create an org + user:

![TheHive Organizations](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605113123.png)

Add more users/profiles as needed:

![TheHive Users](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605113317.png)

Set passwords. For the Shuffle integration user specifically, generate an API key:

![TheHive API Key](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605113655.png)

Store that key securely — it's what Shuffle authenticates with. Then log out of admin and into the regular user account:

![TheHive Dashboard](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605115228.png)

**5.2.5 Connect Shuffle → TheHive**

Click "Authenticate TheHive," paste the API key, set the URL to TheHive's public IP + port:

![Shuffle TheHive Authentication](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605115759.png)

Under "Find actions" → TheHive → "Create alerts." Example payload:

```json
{
  "description": "Mimikatz Detected on host: DESKTOP-HS8N3J7",
  "externallink": "",
  "flag": false,
  "pap": 2,
  "severity": "2",
  "source": "Wazuh",
  "sourceRef": "Rule:100002",
  "status": "New",
  "summary": "Details about the Mimikatz detection",
  "tags": [
    "T1003"
  ],
  "title": "Mimikatz Detection Alert",
  "tlp": 2,
  "type": "Internal"
}
```

Set this in the "Body" section:

![Shuffle TheHive Body](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605141313.png)
![Shuffle TheHive Payload](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605145351.png)

Save, rerun — alert should land in TheHive:

![TheHive Alert](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605143428.png)

> Nothing showing up? Make sure TheHive's cloud firewall allows inbound on port 9000.

![TheHive Alert Details](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605145839.png)

Want richer alerts? Add more fields to the payload — e.g. technique ID and command line in the summary:

![Shuffle TheHive Summary](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605165035.png)
![Shuffle TheHive Body](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605165143.png)

Save, rerun, check the updated alert:

![TheHive Alert Updated](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605164956.png)

**5.2.6 Add Email Notifications**

Connect the "Email" app after VirusTotal:

![Shuffle Email](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605165431.png)

Set recipient/subject/body with the relevant alert details:

![Shuffle Email Configuration](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605170704.png)

Save, rerun:

![Shuffle Workflow Final](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605170809.png)

Confirm the email arrives with the right info:

![Email Received](https://github.com/saad-zaib/Security-Operations-Automation/blob/main/images/Pasted%20image%2020240605170915.png)

## 6. Conclusion

This lab stitches together Wazuh, TheHive, and Shuffle into a working automated SOC pipeline — from event generation through alerting, enrichment, case creation, and analyst notification. Recap:

1. Windows 10 + Sysmon for rich telemetry.
2. Wazuh as the central detection/alerting engine.
3. TheHive for case management.
4. Custom Mimikatz detection logic.
5. Shuffle tying it all together as the SOAR layer.
6. End-to-end automated workflow: hash extraction → VirusTotal lookup → TheHive alert → analyst email.

This is a solid base to build on — keep refining detection rules and workflows as threats and tooling evolve, and look into adding more threat intel sources or expanding the playbooks over time.

## 7. References
- https://www.mydfir.com/
- https://www.youtube.com/watch?v=Lb_ukgtYK_U
