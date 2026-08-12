# Azure Arc SSH Brute Force Detection Lab

For this project, I connected an on-premises Ubuntu virtual machine running in VMware to Microsoft Azure using Azure Arc-enabled Servers. I then used Azure Monitor Agent and a Data Collection Rule to send Linux Syslog authentication data into a Log Analytics Workspace connected to Microsoft Sentinel. After confirming the logs were reaching Azure, I generated repeated failed SSH login activity and used Kusto Query Language (KQL) to identify the source IP address, targeted account, hostname, and number of failed attempts. I then created a working detection rule in the Microsoft Defender portal so repeated SSH authentication failures would automatically generate a security alert instead of remaining hidden in the raw logs. The lab's primary goal was to demonstrate how an on-premises Linux system can be brought into a cloud-based SOC workflow for centralized monitoring, threat detection, and investigation.

# Network Topology

```mermaid
flowchart TD
    A["Failed SSH login attempts"] --> B["Ubuntu VM<br/>VMware home lab"]
    B --> C["Linux Syslog authentication logs"]
    B --> D["Azure Arc-enabled server<br/>Connected Machine Agent"]
    C --> E["Azure Monitor Agent<br/>Data Collection Rule"]
    E --> F["Log Analytics Workspace"]
    F --> G["Microsoft Sentinel<br/>Defender portal"]
    G --> H["SSH brute force detection rule"]
    H --> I["Security alert and investigation"]
```

The Ubuntu VM remained inside my local VMware environment while Azure Arc provided the connection between the machine and Azure. The Azure Connected Machine Agent registered the server as an Azure resource, while Azure Monitor Agent collected the selected Syslog events. A Data Collection Rule routed those events into the Log Analytics Workspace, where Microsoft Sentinel and the Microsoft Defender portal could query the data, apply the detection rule, and generate an alert.

# Creating the Azure Environment

To begin, I created a dedicated Azure resource group named `rg-detectlab`. Keeping the lab resources in one group made the Arc-enabled server, monitoring configuration, and security resources easier to organize and manage.

<img width="789" height="311" alt="Creating the rg-detectlab resource group" src="https://github.com/user-attachments/assets/cc3f0d5f-92df-434c-99f7-9ebcf0e0ccd0" />

I then began setting up Azure Arc so my locally hosted Ubuntu VM could be managed and monitored as an Azure-connected resource.

<img width="814" height="578" alt="Setting up Azure Arc for the Ubuntu server" src="https://github.com/user-attachments/assets/1c892235-00c0-4210-8012-70a254f4f4b8" />

Next, I created the monitoring resources required to centralize the Linux security logs in Azure.

<img width="861" height="700" alt="Configuring the Azure monitoring workspace" src="https://github.com/user-attachments/assets/57d5ea00-b304-468c-aeb8-f303192211cb" />

I connected Microsoft Sentinel to the Log Analytics Workspace, giving me a cloud-based SIEM where I could query the Ubuntu logs and build detections.

<img width="1015" height="730" alt="Connecting Microsoft Sentinel to the Log Analytics Workspace" src="https://github.com/user-attachments/assets/6ea291b3-2af4-4f04-81ba-bd87333db807" />

# Connecting the Ubuntu VM with Azure Arc

From the Azure portal, I generated the Azure Arc onboarding script and ran it from the Ubuntu terminal. This downloaded the Azure Connected Machine Agent needed to connect the local server to Azure.

<img width="1718" height="820" alt="Downloading the Azure Connected Machine Agent on Ubuntu" src="https://github.com/user-attachments/assets/b8766200-f2d4-4d40-9b64-85995caa9e35" />

I completed the installation and authentication process from Ubuntu, allowing the machine to register with my Azure subscription and the `rg-detectlab` resource group.

<img width="1718" height="820" alt="Installing and connecting the Azure Arc agent" src="https://github.com/user-attachments/assets/f7746209-700c-4b5a-a7f5-6c095dcd7868" />

After the script completed, I confirmed that the Ubuntu VM appeared in Azure as the Arc-enabled server `user-VMware-Virtual-Platform` in the North Central US region.

<img width="1685" height="835" alt="Ubuntu machine connected as an Azure Arc-enabled server" src="https://github.com/user-attachments/assets/b98d6d5a-4eb8-4e9b-b79e-d017262983d0" />

I also verified the server's connection and agent status to make sure Azure could communicate with the machine before configuring log collection.

<img width="1883" height="330" alt="Verifying the Azure Arc server and agent status" src="https://github.com/user-attachments/assets/3b4d5d01-09cb-4ef5-a307-4cc46fd486f7" />

# Configuring Linux Log Collection

With the Ubuntu VM connected, I deployed Azure Monitor Agent to the Arc-enabled server. This agent was responsible for collecting the Linux security events needed for the detection.

<img width="520" height="770" alt="Deploying Azure Monitor Agent to the Arc-enabled server" src="https://github.com/user-attachments/assets/c18756f2-8a38-4422-8a71-b64068cac04c" />

I created a Data Collection Rule to define which Linux logs should be collected and where they should be sent.

<img width="585" height="392" alt="Creating the Linux Data Collection Rule" src="https://github.com/user-attachments/assets/6b9a43ab-c699-4fb2-95fc-22efbc3e2bcd" />

I selected the Arc-enabled Ubuntu server as the monitored resource and configured Linux Syslog as the data source. This ensured that SSH authentication activity would be forwarded to Azure.

<img width="580" height="881" alt="Adding the Ubuntu Arc server and Linux Syslog data source" src="https://github.com/user-attachments/assets/988dc5d1-b634-418f-9d31-f73ca78baf0c" />

Once the configuration was applied, I confirmed that the Azure Monitor Agent extension was successfully associated with the Ubuntu server and ready to forward events.

<img width="1647" height="119" alt="Confirming the Azure Monitor Agent extension status" src="https://github.com/user-attachments/assets/eaa4b0c1-6fa2-4adc-af44-791d144c08db" />

# Generating SSH Authentication Activity

To test the monitoring pipeline, I generated repeated failed SSH login attempts against the Ubuntu system. These attempts created failed-password messages in the Linux authentication logs, giving me realistic telemetry to search for in Microsoft Sentinel.

<img width="1718" height="820" alt="Generating failed SSH login activity against Ubuntu" src="https://github.com/user-attachments/assets/eeb03d8b-9f0d-454f-8d0e-ef972b868be3" />

# Investigating the Logs with KQL

In the Microsoft Defender portal, I opened the Sentinel logs and queried the `Syslog` table for messages containing failed SSH passwords. This confirmed that the Ubuntu authentication events were successfully traveling from the local VM into Azure.

<img width="579" height="295" alt="Querying Linux Syslog events in Microsoft Sentinel" src="https://github.com/user-attachments/assets/c07163f6-ca14-4a1c-afbb-e6b5f91f9bb8" />

I refined the KQL query to extract useful investigation fields from each event, including the source IP address, targeted account, hostname, first-seen time, last-seen time, and total number of failed attempts.

<img width="694" height="534" alt="Using KQL to extract SSH authentication details" src="https://github.com/user-attachments/assets/88a66b54-3fdc-42e7-b89a-fe9fb01f31e4" />

The summarized results made the activity easier to investigate by grouping repeated login failures instead of requiring an analyst to review every raw Syslog message individually.

<img width="704" height="493" alt="Reviewing summarized failed SSH login results" src="https://github.com/user-attachments/assets/45a56df8-487c-43b5-bd89-84e886b6bac7" />

# Creating the Detection Rule in Microsoft Defender

After validating the query, I created a scheduled detection rule in the Microsoft Defender portal. The rule uses the KQL logic to identify multiple failed SSH logins from the collected Ubuntu Syslog data.

<img width="638" height="371" alt="Creating the SSH brute force detection rule in Microsoft Defender" src="https://github.com/user-attachments/assets/78fa985d-da98-4485-851c-283dcbde766f" />

I configured the rule's detection settings so the activity would be evaluated automatically and produce a security alert when the failed-login threshold was reached.

<img width="764" height="579" alt="Configuring the Defender detection rule settings" src="https://github.com/user-attachments/assets/adeb8d6a-be7c-45e3-aff1-b29b567f9352" />

The completed rule confirmed that the full detection pipeline worked from end to end: local Linux activity was collected from Ubuntu, forwarded through Azure Arc and Azure Monitor Agent, stored in Log Analytics, analyzed by Microsoft Sentinel, and surfaced through the Microsoft Defender portal.

# Project Outcomes

- Connected an on-premises Ubuntu VMware virtual machine to Azure using Azure Arc-enabled Servers
- Deployed Azure Monitor Agent and configured a Data Collection Rule for Linux Syslog ingestion
- Centralized Ubuntu SSH authentication logs in a Log Analytics Workspace connected to Microsoft Sentinel
- Generated failed SSH login activity to validate the complete telemetry pipeline
- Used KQL to extract and summarize the source IP, targeted account, hostname, timestamps, and failed-attempt count
- Created a working scheduled detection rule in the Microsoft Defender portal for repeated SSH authentication failures
- Gained hands-on experience with hybrid-cloud monitoring, SIEM log analysis, detection engineering, and security alerting

Built and Documented by Aluseni Waritay
