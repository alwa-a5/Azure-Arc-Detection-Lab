# Azure Arc SSH Brute Force Detection Lab

- For this project, I connected a locally hosted Ubuntu virtual machine in VMware to Microsoft Azure using Azure Arc-enabled Servers. I deployed Azure Monitor Agent and configured a Data Collection Rule to collect Linux Syslog authentication events and send them to a Log Analytics Workspace. I then enabled Microsoft Sentinel on the workspace, generated repeated failed SSH login activity, and used Kusto Query Language (KQL) to identify the source IP address, targeted account, hostname, and number of failed attempts for a detection rule. After confirming the query returned the expected activity, I created a working Microsoft Sentinel scheduled analytics rule through the Microsoft Defender portal. The main goal of this lab was to demonstrate how a Linux machine outside Azure can be connected to a cloud SIEM, monitored for suspicious authentication activity, and protected with a repeatable detection.

# Network Topology

<p align="center">
  <img src="https://kroki.io/plantuml/svg/eNqFVdtu2kAQffdXTNO-RdSGJOQihOKQ0ESFVgWSqFKlaFkPZsV6F-2uoaTqaz-gn9gv6dgmYIilWALsM2dmzoxnlkvrmHFpIr13QnGZRggtl85ZZI78iVauxpZodYK1pu_QJEIx2S5RpY619dNxqlxaxrnUaaTj1HdaS-svkiUzWCaw59SgH2bfHZ0kWr029pliMSaoXMELDe8zPhWqKhDpWjnBbUHt6XiDPGozs3PGK7yGyFMj3KpwGlImCk71eXYm1JwZlsCY8VlsdKqijpbawHIqHJbsEU5YKl2XWvWFJQihEUyW7HbKIr0UKoYJk3bHcy6g3jgpIUpHaHEOR2XQMDXLwaAEjoSTmOUcimeExlmlya0kwljLqMpaVPO-nl8lQldres-b0PXjStva-yi_SozQGL1cGxv5tW8cTQWfKbQWyn4GuWMqJr2_PICryp4Trk2EZjc8wOtqCqzQ36SnQRHKKDTUcQL6QokkTR5F5KZQPwu20C2KeOqgfh54vz2vF37_ej966t10R0-Du0-3I89l_YN8XKgeDsPhLVyZ1CGlNBzhGh2VIrSCHht73raugxbPNF6se9ZufdisU8vPTe0fqjVuZwFD5zCZO9vyxxloqZCLeqM9wDkyhxFNkpD0Q7snFJEye_sAmAXmHLWuKuvN-clxI6Cs61V9yQnwwmgGp8HpGTHWu7qj6j73gof-nqSHfsaFQ-gJlf6E4cqSqLKiRVKlJghOG50m5drf652k2y4fQtgP91J3tFIUmfqwdoYwpgUm7nVnsNMUw9_WUH1g7MghCmw4e2LK5YNjY4llBQTatyVsjp-drH3BjbZ64uDFvpf687cemJTC_vvzF0KJJmvBneIiIn5ZhV37e14xJlCrtbP3s0jyu6xN9Mnvc8HZV_60deR0Rzs0yc8B6BZjmE0so5lf0EEKE6mXFiZGJ-CmSIE4k7CZHhDKaXhdEkU0dJKuV-ejd4kqor-k_1PORG8" width="100%" alt="Azure Arc SSH brute force detection lab topology" />
</p>

- The Ubuntu VM remained in my local VMware environment. The Azure Connected Machine Agent registered it as an Azure Arc-enabled server and allowed Azure extensions, including Azure Monitor Agent, to be deployed to it. The Data Collection Rule defined which Linux Syslog events Azure Monitor Agent collected and sent to the Log Analytics Workspace. Microsoft Sentinel analyzed the stored data, while the Microsoft Defender portal provided the interface where I investigated the logs and managed the scheduled analytics rule.

# Creating the Azure Environment

- To begin, I created a dedicated resource group named `rg-detectlab` to keep the Azure Arc, monitoring, and Microsoft Sentinel resources for the lab organized in one place.

<img width="789" height="311" alt="Creating the rg-detectlab resource group" src="https://github.com/user-attachments/assets/cc3f0d5f-92df-434c-99f7-9ebcf0e0ccd0" />

- Next, I created the Log Analytics Workspace that would store the Linux Syslog events collected from the Ubuntu VM.
<img width="207" height="73" alt="image" src="https://github.com/user-attachments/assets/f05009dd-4430-4211-a8c0-17b43a969e8c" />



 - I then started the Azure Arc server onboarding process for the Ubuntu machine.
<img width="861" height="700" alt="Screenshot 2026-07-31 204637" src="https://github.com/user-attachments/assets/6c1f00aa-d08c-4932-8a96-4e6c1675e087" />

<img width="1015" height="730" alt="Screenshot 2026-07-31 204701" src="https://github.com/user-attachments/assets/d3769a3f-4dde-4714-838b-81ebe7cab58c" />

<img width="814" height="578" alt="Screenshot 2026-07-31 201708" src="https://github.com/user-attachments/assets/3524d245-5249-4004-a8a8-0c08ecc35a82" />

- I enabled Microsoft Sentinel on the workspace so the collected logs could be queried, analyzed, and used in an analytics rule.
<img width="1883" height="330" alt="Screenshot 2026-08-04 181915" src="https://github.com/user-attachments/assets/b2031785-9053-45fd-9ace-46aa5a1d9cb8" />

# Connecting Ubuntu to Azure Arc

- From the Azure portal, I generated the Azure Arc installation script and ran it in the Ubuntu terminal. The script downloaded and installed the Azure Connected Machine Agent.

<img width="1718" height="820" alt="Downloading the Azure Connected Machine Agent on Ubuntu" src="https://github.com/user-attachments/assets/b8766200-f2d4-4d40-9b64-85995caa9e35" />

- I completed the authentication and connection process, which registered the local Ubuntu VM with the `rg-detectlab` resource group.

<img width="1718" height="820" alt="Connecting the Ubuntu VM to Azure Arc" src="https://github.com/user-attachments/assets/f7746209-700c-4b5a-a7f5-6c095dcd7868" />

- After the onboarding script completed, I confirmed that the machine appeared in Azure as the Arc-enabled server `user-VMware-Virtual-Platform` in the North Central US region.

<img width="1685" height="835" alt="Ubuntu VM listed as an Azure Arc-enabled server" src="https://github.com/user-attachments/assets/b98d6d5a-4eb8-4e9b-b79e-d017262983d0" />

- I also confirmed that Azure could communicate with the connected machine before moving on to log collection.



# Configuring Linux Syslog Collection

 - With the Ubuntu machine connected through Azure Arc, I began configuring Azure Monitor Agent and the Data Collection Rule needed for Linux log ingestion.

<img width="520" height="770" alt="Beginning the Azure Monitor Agent and Data Collection Rule setup" src="https://github.com/user-attachments/assets/c18756f2-8a38-4422-8a71-b64068cac04c" />

 - Successful download
<img width="1647" height="119" alt="Screenshot 2026-08-11 202527" src="https://github.com/user-attachments/assets/c00b2d75-18b2-46dd-954c-e26a4c923ffc" />


 - I selected the Arc-enabled Ubuntu machine as the resource that the Data Collection Rule would monitor.

<img width="585" height="392" alt="Selecting the Arc-enabled Ubuntu server as the monitored resource" src="https://github.com/user-attachments/assets/6b9a43ab-c699-4fb2-95fc-22efbc3e2bcd" />

- I then configured Linux Syslog as the data source and the Log Analytics Workspace as the destination. Azure Monitor Agent used this configuration to forward the selected authentication events into the `Syslog` table.

<img width="580" height="881" alt="Configuring Linux Syslog collection and the Log Analytics destination" src="https://github.com/user-attachments/assets/988dc5d1-b634-418f-9d31-f73ca78baf0c" />

# Creating the Microsoft Sentinel Analytics Rule

 - After validating the KQL query, I used it to create a scheduled analytics rule in Microsoft Sentinel through the Microsoft Defender portal.

<img width="694" height="534" alt="634570858-88a66b54-3fdc-42e7-b89a-fe9fb01f31e4" src="https://github.com/user-attachments/assets/4d33bc67-f524-453b-82b3-4265fda84467" />

- I configured the rule to run automatically and evaluate the failed-login threshold defined by the query.

<img width="638" height="371" alt="Configuring the analytics rule scheduling and detection logic" src="https://github.com/user-attachments/assets/78fa985d-da98-4485-851c-283dcbde766f" />

- Mapped the rule to MITRE ATT&CK T1110.001 Password Guessing.
  
<img width="579" height="295" alt="Screenshot 2026-08-05 184512" src="https://github.com/user-attachments/assets/6d21f6bb-fd7e-4de5-90a3-cc295cbf410c" />

- Used KQL to identify failed SSH authentication events from Ubuntu Syslog.

<img width="704" height="493" alt="Screenshot 2026-08-05 184853" src="https://github.com/user-attachments/assets/a4bad9ed-7af9-4e81-a143-d9b9b3ee861b" />

- Added FailedAttempts, FirstSeen, and LastSeen as alert details.


<img width="638" height="371" alt="Screenshot 2026-08-05 185047" src="https://github.com/user-attachments/assets/317e54cf-6bc1-4f93-b11b-9e6f6967da60" />

- Configured the rule to run every 5 minutes and generate alerts.

<img width="764" height="579" alt="Screenshot 2026-08-05 185253" src="https://github.com/user-attachments/assets/82dbc25f-e883-438e-81b3-be99c8ed29fe" />

- I reviewed the rule settings and completed the creation process so that future matching activity could generate an alert and be grouped into an incident for investigation.

# Generating Failed SSH Login Activity

- To test the monitoring pipeline, I generated repeated failed SSH logins against the Ubuntu system. These attempts produced `Failed password` messages in the Linux authentication logs, giving me realistic events to investigate in Microsoft Sentinel.

<img width="1718" height="820" alt="Generating failed SSH login activity against Ubuntu" src="https://github.com/user-attachments/assets/eeb03d8b-9f0d-454f-8d0e-ef972b868be3" />


- Finally, I confirmed that the completed SSH detection rule appeared in the Microsoft Defender portal. This verified that the rule was created successfully and that the lab's monitoring and detection workflow worked from end to end.

<img width="1117" height="875" alt="Screenshot 2026-08-05 200031" src="https://github.com/user-attachments/assets/b6688ac8-6c6b-4ef0-899b-95143f54b873" />

# Project Outcomes

- Connected an Ubuntu VMware VM to Azure Arc and registered it in the `rg-detectlab` resource group.
- Configured Azure Monitor Agent, Syslog collection, and Log Analytics for Microsoft Sentinel ingestion.
- Generated failed SSH logins and used KQL to analyze source IPs, accounts, timestamps, and failed-attempt counts.
- Created and enabled a Microsoft Sentinel scheduled analytics rule for SSH brute-force detection.

Built and Documented by Aluseni Waritay
