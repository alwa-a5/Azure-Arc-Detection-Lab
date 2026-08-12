# Azure Arc SSH Brute Force Detection Lab

For this project, I connected a locally hosted Ubuntu virtual machine in VMware to Microsoft Azure using Azure Arc-enabled Servers. I deployed Azure Monitor Agent and configured a Data Collection Rule to collect Linux Syslog authentication events and send them to a Log Analytics Workspace. I then enabled Microsoft Sentinel on the workspace, generated repeated failed SSH login activity, and used Kusto Query Language (KQL) to identify the source IP address, targeted account, hostname, and number of failed attempts. After confirming the query returned the expected activity, I created a working Microsoft Sentinel scheduled analytics rule through the Microsoft Defender portal. The main goal of this lab was to demonstrate how a Linux machine outside Azure can be connected to a cloud SIEM, monitored for suspicious authentication activity, and protected with a repeatable detection.

# Network Topology

```mermaid
flowchart TD
    A["SSH login attempts"] --> B["Ubuntu VM<br/>VMware home lab"]
    B --> C["Linux authentication events<br/>Syslog"]
    B --> D["Azure Arc-enabled server<br/>Connected Machine Agent"]
    D --> E["Azure Monitor Agent<br/>configured by a DCR"]
    C --> E
    E --> F["Log Analytics Workspace"]
    F --> G["Microsoft Sentinel<br/>in the Defender portal"]
    G --> H["Scheduled analytics rule"]
    H --> I["Security alert and incident"]
```

The Ubuntu VM remained in my local VMware environment. The Azure Connected Machine Agent registered it as an Azure Arc-enabled server and allowed Azure extensions, including Azure Monitor Agent, to be deployed to it. The Data Collection Rule defined which Linux Syslog events Azure Monitor Agent collected and sent to the Log Analytics Workspace. Microsoft Sentinel analyzed the stored data, while the Microsoft Defender portal provided the interface where I investigated the logs and managed the scheduled analytics rule.

# Creating the Azure Environment

To begin, I created a dedicated resource group named `rg-detectlab` to keep the Azure Arc, monitoring, and Microsoft Sentinel resources for the lab organized in one place.

<img width="789" height="311" alt="Creating the rg-detectlab resource group" src="https://github.com/user-attachments/assets/cc3f0d5f-92df-434c-99f7-9ebcf0e0ccd0" />

I then started the Azure Arc server onboarding process for the Ubuntu machine.

<img width="814" height="578" alt="Starting the Azure Arc server onboarding process" src="https://github.com/user-attachments/assets/1c892235-00c0-4210-8012-70a254f4f4b8" />

Next, I created the Log Analytics Workspace that would store the Linux Syslog events collected from the Ubuntu VM.

<img width="861" height="700" alt="Creating the Log Analytics Workspace" src="https://github.com/user-attachments/assets/57d5ea00-b304-468c-aeb8-f303192211cb" />

I enabled Microsoft Sentinel on the workspace so the collected logs could be queried, analyzed, and used in an analytics rule.

<img width="1015" height="730" alt="Enabling Microsoft Sentinel on the Log Analytics Workspace" src="https://github.com/user-attachments/assets/6ea291b3-2af4-4f04-81ba-bd87333db807" />

# Connecting Ubuntu to Azure Arc

From the Azure portal, I generated the Azure Arc installation script and ran it in the Ubuntu terminal. The script downloaded and installed the Azure Connected Machine Agent.

<img width="1718" height="820" alt="Downloading the Azure Connected Machine Agent on Ubuntu" src="https://github.com/user-attachments/assets/b8766200-f2d4-4d40-9b64-85995caa9e35" />

I completed the authentication and connection process, which registered the local Ubuntu VM with the `rg-detectlab` resource group.

<img width="1718" height="820" alt="Connecting the Ubuntu VM to Azure Arc" src="https://github.com/user-attachments/assets/f7746209-700c-4b5a-a7f5-6c095dcd7868" />

After the onboarding script completed, I confirmed that the machine appeared in Azure as the Arc-enabled server `user-VMware-Virtual-Platform` in the North Central US region.

<img width="1685" height="835" alt="Ubuntu VM listed as an Azure Arc-enabled server" src="https://github.com/user-attachments/assets/b98d6d5a-4eb8-4e9b-b79e-d017262983d0" />

I also confirmed that Azure could communicate with the connected machine before moving on to log collection.

<img width="1883" height="330" alt="Confirming the Azure Arc machine connection" src="https://github.com/user-attachments/assets/3b4d5d01-09cb-4ef5-a307-4cc46fd486f7" />

# Configuring Linux Syslog Collection

With the Ubuntu machine connected through Azure Arc, I began configuring Azure Monitor Agent and the Data Collection Rule needed for Linux log ingestion.

<img width="520" height="770" alt="Beginning the Azure Monitor Agent and Data Collection Rule setup" src="https://github.com/user-attachments/assets/c18756f2-8a38-4422-8a71-b64068cac04c" />

I selected the Arc-enabled Ubuntu machine as the resource that the Data Collection Rule would monitor.

<img width="585" height="392" alt="Selecting the Arc-enabled Ubuntu server as the monitored resource" src="https://github.com/user-attachments/assets/6b9a43ab-c699-4fb2-95fc-22efbc3e2bcd" />

I then configured Linux Syslog as the data source and the Log Analytics Workspace as the destination. Azure Monitor Agent used this configuration to forward the selected authentication events into the `Syslog` table.

<img width="580" height="881" alt="Configuring Linux Syslog collection and the Log Analytics destination" src="https://github.com/user-attachments/assets/988dc5d1-b634-418f-9d31-f73ca78baf0c" />

# Generating Failed SSH Login Activity

To test the monitoring pipeline, I generated repeated failed SSH logins against the Ubuntu system. These attempts produced `Failed password` messages in the Linux authentication logs, giving me realistic events to investigate in Microsoft Sentinel.

<img width="1718" height="820" alt="Generating failed SSH login activity against Ubuntu" src="https://github.com/user-attachments/assets/eeb03d8b-9f0d-454f-8d0e-ef972b868be3" />

# Investigating the Events with KQL

In the Microsoft Defender portal, I opened Microsoft Sentinel logs and queried the `Syslog` table for messages containing `Failed password`. Seeing the Ubuntu events in the results confirmed that the collection path from the local VM to Log Analytics was working.

<img width="579" height="295" alt="Searching the Syslog table for failed SSH logins" src="https://github.com/user-attachments/assets/c07163f6-ca14-4a1c-afbb-e6b5f91f9bb8" />

I refined the KQL logic to extract the source IP address and targeted account from each Syslog message. I then grouped repeated failures by hostname, source IP, and account so the activity could be evaluated as one pattern instead of as unrelated raw events.

<img width="694" height="534" alt="Building the KQL query for repeated failed SSH logins" src="https://github.com/user-attachments/assets/88a66b54-3fdc-42e7-b89a-fe9fb01f31e4" />

# Creating the Microsoft Sentinel Analytics Rule

After validating the KQL query, I used it to create a scheduled analytics rule in Microsoft Sentinel through the Microsoft Defender portal.

<img width="704" height="493" alt="Creating the scheduled analytics rule from the KQL query" src="https://github.com/user-attachments/assets/45a56df8-487c-43b5-bd89-84e886b6bac7" />

I configured the rule to run automatically and evaluate the failed-login threshold defined by the query.

<img width="638" height="371" alt="Configuring the analytics rule scheduling and detection logic" src="https://github.com/user-attachments/assets/78fa985d-da98-4485-851c-283dcbde766f" />

I reviewed the rule settings and completed the creation process so future matching activity could generate an alert and be grouped into an incident for investigation.

<img width="764" height="579" alt="Reviewing and completing the Microsoft Sentinel analytics rule" src="https://github.com/user-attachments/assets/adeb8d6a-be7c-45e3-aff1-b29b567f9352" />

Finally, I confirmed that the completed SSH detection rule appeared in the Microsoft Defender portal. This verified that the rule was created successfully and that the lab's monitoring and detection workflow worked from end to end.

<img width="1647" height="119" alt="Completed SSH detection rule in the Microsoft Defender portal" src="https://github.com/user-attachments/assets/eaa4b0c1-6fa2-4adc-af44-791d144c08db" />

# Project Outcomes

- Connected a locally hosted Ubuntu VMware VM to Azure using Azure Arc-enabled Servers
- Registered the machine as `user-VMware-Virtual-Platform` in the `rg-detectlab` resource group
- Deployed Azure Monitor Agent and configured a Data Collection Rule for Linux Syslog ingestion
- Sent Ubuntu authentication events to a Log Analytics Workspace connected to Microsoft Sentinel
- Generated failed SSH login activity to validate the collection pipeline
- Used KQL to extract and summarize the source IP address, targeted account, hostname, timestamps, and failed-attempt count
- Created and enabled a working Microsoft Sentinel scheduled analytics rule through the Microsoft Defender portal
- Gained hands-on experience with Azure Arc, hybrid-cloud monitoring, SIEM investigation, KQL, and detection engineering

Built and Documented by Aluseni Waritay
