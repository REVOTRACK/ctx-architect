# CORTEX_ARCHITECTURE_DIAGRAMS.md
**Visual Architecture, Execution Flows & Boundary Maps**  
**Audit Date:** 2026-10-11  
**Operating Mode:** Read-Only Verified Documentation

---

## 1. System Component Topology & Boundaries

```mermaid
flowchart TB
    subgraph Client_Plane ["Ingress & Inbound Traffic"]
        Operator["Operator Web Browser (HTTPS :443)"]
        Recipient["Recipient (Tracking Pixel / Click :8082)"]
        SystemCron["Systemd Cron (* * * * * artisan schedule:run)"]
    end

    subgraph Laravel_Tier ["Laravel 10 Application Layer (/var/www/cortex)"]
        Apache["Apache 2.4 (VirtualHost / SSL)"]
        HttpKernel["Http Kernel & Middleware Pipeline\n(CheckAllowedIp, ApiKeyAuthentication)"]
        Controllers["Controllers & Livewire 3 Components\n(EmailCampaignController, EmailApiProviderManager)"]
        Services["Domain Services\n(EmailCampaignProcessService, PowerMtaConfigGenerator, etc.)"]
        QueueWorkers["Laravel Queue Workers\n(google-workspace queue, 5 procs)"]
    end

    subgraph Go_Core ["Core Microservices Engine (/var/www/cortex/microservices)"]
        MasterBin["cortex-master\n(-op=run-campaign, -op=run-pmta-accounting)"]
        TrackingBin["cortex-tracking\n(HTTP Daemon :8082)"]
        ToolsBin["cortex-tools\n(DNSBL & Scraping CLI)"]
        AffiliateDrivers["networks_apis/*\n(cortex-cake, cortex-hasoffers, etc.)"]
    end

    subgraph Storage_Tier ["Isolated Persistence Layer"]
        DBCortex[("MySQL: cortex\n(App State, Campaigns, Server Interfaces)")]
        DBCortexData[("MySQL: cortex_data\n(Subscribers, Granular Engagements)")]
        DBCortexSupp[("MySQL: cortex_suppression_lists\n(offer_supp_{id} MD5 Tables)")]
        Valuestore["Valuestore: storage/app/settings.json\n(PMTA Templates, API Keys)"]
    end

    subgraph Remote_Nodes ["Distributed Delivery Nodes (PowerMTA)"]
        PowerMTA["PowerMTA Daemon (Port 25)\n(Virtual MTAs & Sockets)"]
        RelayDaemon["cortex-relay Go Daemon (:2525)\n(Local SMTP -> HTTP API Bridge)"]
    end

    subgraph Upstream_APIs ["External Ecosystem"]
        DNSAPIs["DNS APIs\n(Cloudflare, GoDaddy, Namecheap)"]
        CloudVPS["Cloud VPS APIs\n(Hetzner, Linode, OVH, Scaleway)"]
        AffiliateNets["Affiliate Networks\n(Cake, Everflow, Optizmo)"]
        ESPUpstream["Upstream Email APIs\n(AWS SES, Elastic Email, Postmark)"]
    end

    Operator --> Apache
    Apache --> HttpKernel
    HttpKernel --> Controllers
    Controllers --> Services
    SystemCron --> Services

    Services --> MasterBin
    Services --> AffiliateDrivers
    Services --> DNSAPIs
    Services --> CloudVPS

    Recipient --> TrackingBin
    TrackingBin --> DBCortexData
    TrackingBin --> DBCortex

    MasterBin --> DBCortex
    MasterBin --> DBCortexData
    MasterBin --> DBCortexSupp
    MasterBin --> PowerMTA

    PowerMTA -- "route 127.0.0.1:2525" --> RelayDaemon
    RelayDaemon --> ESPUpstream

    AffiliateDrivers --> AffiliateNets
    QueueWorkers --> Upstream_APIs
```

---

## 2. Campaign Lifecycle Sequence (Creation to Dispatch)

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Operator / Cron
    participant Web as EmailCampaignController
    participant Audit as EmailCampaignAuditService
    participant Process as EmailCampaignProcessService
    participant DB as MySQL (cortex / cortex_suppression_lists)
    participant Engine as cortex-master (Go Binary)
    participant MTA as PowerMTA Delivery Server

    Admin->>Web: POST /emailCampaigns/control/{id} (action='publish')
    Web->>Audit: revision($campaignData, $userId)
    
    Audit->>DB: Query allowed_test_emails (Whitelist Check)
    Audit->>DB: Query blacklists (Domain Reputation Check)
    Audit->>DB: Check table cortex_suppression_lists.offer_supp_{offer_id}
    
    alt Audit Checks Fail
        Audit-->>Web: auditPassed = false + error message
        Web-->>Admin: Abort dispatch with notification
    else Audit Checks Pass
        Audit-->>Web: auditPassed = true
        Web->>Process: runEmailCampaign($campaign)
        Process->>Process: Compile parameters & custom tags
        Process->>Process: Write /var/www/cortex/microservices/campaigns/{id}.json
        Process->>Engine: exec("cd microservices; sudo nohup ./cortex-master -op=run-campaign -emailCampaignID={id} &")
        Process-->>Web: Return success = true
        Web-->>Admin: Return HTTP 200 (Campaign Running)
        
        par Background Sending Execution
            Engine->>DB: Stream subscriber rows from cortex_data.email_subscribers
            Engine->>DB: Match MD5 hashes against cortex_suppression_lists.offer_supp_{id}
            Engine->>Engine: Embed tracking tokens & personalized headers
            Engine->>MTA: Inject messages into Virtual MTAs over SMTP
            Engine->>DB: Update cortex.email_campaigns (progress, messages_sent)
        end
    end
```

---

## 3. Provider Configuration & Relay Link Sequence

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Operator
    participant Livewire as EmailApiProviderManager
    participant Service as EmailApiProviderManagerService
    participant InterfaceModel as DeliveryServerInterface
    participant ConfigGen as PowerMtaConfigGenerator
    participant Settings as storage/app/settings.json
    participant DB as MySQL: cortex

    Admin->>Livewire: Enters Provider (Elastic Email, AWS SES, Postmark)
    Livewire->>DB: EmailThroughApiProvider::create([...])
    Admin->>Livewire: Selects Provider + Delivery Server -> Clicks "Link Server"
    Livewire->>Service: linkWithServer($providerId, $serverId)
    
    Service->>InterfaceModel: Instantiates new DeliveryServerInterface
    Service->>InterfaceModel: Sets tag, ip, domain, interface_type = provider_name
    Service->>InterfaceModel: Sets interface_data = json_encode(providerCredentials)
    Service->>InterfaceModel: Sets dyn_relation = '{provider_name}-{id}'
    Service->>InterfaceModel: Sets ex_smtp_host = default_from_email (Sender Identity)
    Service->>InterfaceModel: save()
    
    InterfaceModel->>ConfigGen: renderConfig($interface)
    ConfigGen->>Settings: Fetch 'delivery_server_pmta_smtp_relay_template'
    Settings-->>ConfigGen: "<virtual-mta [tag]>\n... route 127.0.0.1:2525\n</virtual-mta>"
    ConfigGen->>ConfigGen: Replace placeholders: [tag_name], [private_ip], [hostname]
    ConfigGen->>DB: Save delivery_server_interfaces.pmta_config
    
    Service-->>Livewire: Link Completed Successfully
    Livewire-->>Admin: UI Notification ("Provider Linked to Server")
```

---

## 4. PowerMTA Delivery & `cortex-relay` Bridge

```mermaid
flowchart TD
    subgraph Master_Server ["Cortex Master Server"]
        MasterBin["cortex-master Engine\n(-op=run-campaign)"]
    end

    subgraph Delivery_Server ["Remote PowerMTA Delivery Server"]
        PMTAListener["PowerMTA Inbound SMTP Listener\n(Port 25 / Custom)"]
        
        subgraph PMTA_Routing ["Virtual MTA Evaluation"]
            VMTADirect["<virtual-mta vmta-direct>\nDirect Outbound SMTP\n(Port 25 to MX)"]
            VMTARelay["<virtual-mta api-relay>\nroute 127.0.0.1:2525\n(Local Bridge)"]
        end
        
        RelayDaemon["cortex-relay Go Daemon\n(Listening on 127.0.0.1:2525)\nExecStart=/home/cortex-relay"]
    end

    subgraph Internet_Ecosystem ["Internet & Upstream Providers"]
        RemoteMX["Destination Mail Servers\n(Gmail, Yahoo, Outlook MX)"]
        SESAPI["Amazon SES REST API\n(email.amazonaws.com)"]
        ElasticAPI["Elastic Email REST API\n(api.elasticemail.com)"]
        PostmarkAPI["Postmark REST API\n(api.postmarkapp.com)"]
    end

    MasterBin -- "Inject Email via SMTP" --> PMTAListener
    PMTAListener --> PMTA_Routing

    VMTADirect -- "Direct MX Delivery" --> RemoteMX
    VMTARelay -- "Local SMTP Injection" --> RelayDaemon

    RelayDaemon -- "*ses.Client HTTP Post" --> SESAPI
    RelayDaemon -- "Elastic Client HTTP Post" --> ElasticAPI
    RelayDaemon -- "Postmark Client HTTP Post" --> PostmarkAPI
```

---

## 5. Recipient Tracking & Engagement Attribution

```mermaid
sequenceDiagram
    autonumber
    actor User as Recipient / Mail User Agent
    participant Daemon as cortex-tracking (Go :8082)
    participant MMDB as Local MaxMind GeoIP DB
    participant DBData as MySQL: cortex_data
    participant DBApp as MySQL: cortex
    actor Offer as Affiliate Landing Page

    User->>Daemon: HTTP GET http://{host}:8082/rd/{encrypted_token}
    Daemon->>Daemon: Check rate limits ([protection] max_clicks_in_minute)
    Daemon->>MMDB: Resolve client IP to Country Code & ISP
    
    par Async Interaction Logging
        Daemon->>DBData: INSERT INTO email_campaign_engagements\n(campaign_id, subscriber_id, click_type_id, geo_id, ip, timestamp)
        Daemon->>DBApp: UPDATE email_campaign_statistics\nSET clicks = clicks + 1 WHERE campaign_id = {id}
    end
    
    Daemon-->>User: HTTP 302 Redirect (Location: https://offer.affiliate.com/...)
    User->>Offer: Loads Affiliate Offer Page
```

---

## 6. Physical Database Partitioning & Ownership Boundaries

```mermaid
erDiagram
    CORTEX_APP ||--o{ DELIVERY_SERVERS : manages
    DELIVERY_SERVERS ||--o{ DELIVERY_SERVER_INTERFACES : binds
    CORTEX_APP ||--o{ EMAIL_CAMPAIGNS : configures
    EMAIL_CAMPAIGNS ||--o{ EMAIL_CAMPAIGN_STATISTICS : rolls_up
    CORTEX_APP ||--o{ SPONSORS : connects
    SPONSORS ||--o{ OFFERS : owns
    
    CORTEX_DATA ||--o{ EMAIL_LISTS : stores
    EMAIL_LISTS ||--o{ EMAIL_SUBSCRIBERS : contains
    EMAIL_SUBSCRIBERS ||--o{ EMAIL_CAMPAIGN_ENGAGEMENTS : logs

    CORTEX_SUPPRESSION ||--o{ OFFER_SUPP_TABLES : dynamically_creates

    CORTEX_APP {
        string users "System operators & permissions"
        string delivery_servers "Remote MTA nodes & SSH credentials"
        string delivery_server_interfaces "VMTAs, IPs, domains, relay stanzas"
        string email_campaigns "Campaign metadata, schedules, commands"
        string email_through_api_providers "Third-party API credentials"
        string api_keys "X-API-Key records & permission scopes"
    }

    CORTEX_DATA {
        string email_lists "List definitions, categories, verticals"
        string email_subscribers "Millions of subscriber records & custom fields"
        string email_campaign_engagements "Granular opens, clicks, unsubscribes, leads"
    }

    CORTEX_SUPPRESSION {
        string offer_supp_offer_id "MD5 & plain suppression hashes per offer"
        string global_suppression_lists "System-wide suppression hashes"
    }
```
