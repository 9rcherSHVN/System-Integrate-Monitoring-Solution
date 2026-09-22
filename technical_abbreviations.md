## Abbreviations and Acronyms Used

The following list consolidates the abbreviations used throughout the pilot-scope discussion.

| Abbreviation | Full term | Description |
|---|---|---|
| **AD** | Active Directory | Microsoft directory service used to manage users, computers, groups, policies, and service identities in an on-premises Windows environment. |
| **AMA** | Azure Monitor Agent | Microsoft monitoring agent for collecting Windows performance counters, event logs, and other telemetry into Azure Monitor. |
| **API** | Application Programming Interface | A defined interface through which applications communicate, commonly using HTTP/HTTPS REST calls. |
| **APM** | Application Performance Monitoring | Monitoring of application availability, response time, errors, dependencies, and transaction behavior. |
| **ARM** | Azure Resource Manager | Azure’s management and deployment framework for creating and managing resources, including templates and resource groups. |
| **Azure Arc** | Azure Arc-enabled servers | Azure service that represents on-premises or other-cloud servers as Azure-managed resources without migrating them to Azure VMs. |
| **Azure VM** | Azure Virtual Machine | A virtual machine hosted in Microsoft Azure. |
| **CCA** | Cross-Correlation Analysis | In this context, correlating application, network, server, and SQL telemetry to identify the root cause of an incident. |
| **CPU** | Central Processing Unit | Server processor resource used to execute operating-system and application workloads. |
| **DB** | Database | An organized collection of data managed by a database management system such as Microsoft SQL Server. |
| **DCE** | Data Collection Endpoint | Azure Monitor endpoint used by agents or custom collectors to send monitoring data to Azure. |
| **DCR** | Data Collection Rule | Azure Monitor configuration defining what data to collect, how to transform it, and where to send it. |
| **DMV** | Dynamic Management View | SQL Server system view or function that exposes runtime information such as sessions, waits, locks, requests, memory, and I/O. |
| **DNS** | Domain Name System | Service that resolves hostnames to IP addresses. DNS failures can appear as application or network failures. |
| **ERP** | Enterprise Resource Planning | Business application platform managing organizational processes such as finance, operations, inventory, purchasing, or human resources. |
| **gMSA** | Group Managed Service Account | Active Directory-managed service account whose password is automatically managed and rotated by Windows. |
| **HTTP** | Hypertext Transfer Protocol | Application-layer protocol used for web and API communication. |
| **HTTPS** | Hypertext Transfer Protocol Secure | HTTP protected by TLS encryption, normally used for secure web and REST API communication. |
| **I/O** | Input/Output | Data read and write activity, including disk, database, and network operations. |
| **IIS** | Internet Information Services | Microsoft Windows web server platform commonly used to host web applications and REST APIs. |
| **IP** | Internet Protocol | Network addressing and routing protocol used to identify systems and exchange network traffic. |
| **ITSM** | Information Technology Service Management | Processes and tools used to manage incidents, problems, changes, requests, and other IT support activities. |
| **JSON** | JavaScript Object Notation | Lightweight text-based data format commonly used by REST APIs and monitoring ingestion interfaces. |
| **KQL** | Kusto Query Language | Query language used in Azure Monitor Log Analytics to search, analyze, aggregate, and correlate telemetry. |
| **LAPS** | Local Administrator Password Solution | Microsoft capability for automatically managing and rotating local administrator passwords. |
| **LAW** | Log Analytics Workspace | Azure Monitor data store in which logs, events, performance data, and custom telemetry are collected and queried. |
| **MMA** | Microsoft Monitoring Agent | Legacy Microsoft monitoring agent, replaced by Azure Monitor Agent for current Azure Monitor implementations. |
| **OMS** | Operations Management Suite | Former Microsoft cloud monitoring and management product name; its capabilities were incorporated into Azure Monitor. |
| **OS** | Operating System | Core system software, such as Windows Server, that manages hardware and provides services to applications. |
| **PID** | Process Identifier | Numeric identifier assigned by Windows to a running process. It can be used to associate a network connection with a process. |
| **POC** | Proof of Concept | Limited implementation used to validate technical feasibility before broader deployment. |
| **RBAC** | Role-Based Access Control | Authorization model that grants permissions according to assigned roles. |
| **REST** | Representational State Transfer | Common architectural style for web APIs, usually implemented over HTTP/HTTPS with resources and standard methods such as GET and POST. |
| **RTT** | Round-Trip Time | Time required for a network request to travel to the destination and for a response to return. |
| **SAN** | Subject Alternative Name | Certificate field containing the hostnames or identities for which a TLS certificate is valid. |
| **SCOM** | System Center Operations Manager | Microsoft’s traditional enterprise monitoring platform, commonly used for on-premises infrastructure and application monitoring. |
| **SDK** | Software Development Kit | Development libraries and tools used to integrate an application with a platform or service. |
| **SQL** | Structured Query Language | Language used to query and manage relational databases. In this project, it primarily refers to Microsoft SQL Server. |
| **TCP** | Transmission Control Protocol | Reliable, connection-oriented network protocol used by services such as HTTPS and SQL Server. |
| **TLS** | Transport Layer Security | Cryptographic protocol used to encrypt network communications, including HTTPS connections. |
| **URL** | Uniform Resource Locator | Address identifying a web resource or API endpoint. |
| **VM** | Virtual Machine | Software-based computer running an operating system and applications. |
| **VM Insights** | Virtual Machine Insights | Azure Monitor capability providing performance, process, dependency, and health visibility for supported VMs and Arc-enabled servers. |
| **WAS** | Windows Process Activation Service | Windows service that supports application activation and management for IIS-hosted applications. |

## Common Technical Terms That Are Not Strictly Abbreviations

| Term | Description |
|---|---|
| **Azure Automation** | Azure service used to run PowerShell runbooks and automation workflows. A Hybrid Runbook Worker can execute automation inside the on-premises network. |
| **Azure Monitor** | Azure’s monitoring platform for metrics, logs, alerts, workbooks, and application/infrastructure observability. |
| **Azure Monitor Workbooks** | Interactive dashboards that combine Azure metrics, Log Analytics queries, charts, and operational controls. |
| **Connection Monitor** | Azure network-monitoring capability used to test connectivity, reachability, latency, and packet loss between endpoints. |
| **Data Collection Rule stream** | Logical input definition within a DCR that describes the format and destination of collected data. |
| **Hybrid Runbook Worker** | An on-premises or other-cloud execution host for Azure Automation runbooks. |
| **Logs Ingestion API** | Azure API used by custom applications or scripts to send structured telemetry into Azure Monitor custom tables. |
| **Managed identity** | Azure identity that allows an Azure resource to authenticate to other Azure services without storing credentials in application code. |
| **Named SQL instance** | SQL Server instance with a name other than the default instance; it may use a dynamically assigned TCP port. |
| **Service principal** | Application identity in Microsoft Entra ID used by automation or applications to authenticate to Azure resources. |
| **Synthetic monitoring** | Automated test that periodically simulates a user or application action, such as calling an API or testing a TCP port. |
| **Workbook** | See Azure Monitor Workbooks; an interactive operational dashboard. |

## Product and Service Name Clarifications

- **Azure Monitor** is the current Azure monitoring platform.
- **Azure Arc** enables management of on-premises servers from Azure without moving them into Azure.
- **AMA** is the current monitoring agent recommended for Azure Monitor.
- **MMA** is the legacy monitoring agent and should generally not be selected for a new implementation.
- **OMS / Operations Insights** are legacy Microsoft product terms that preceded the current Azure Monitor architecture.
- **SCOM** is a separate Microsoft monitoring product and is not required for the proposed Azure Arc and Azure Monitor pilot.
