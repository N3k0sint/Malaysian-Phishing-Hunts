# [PH-2026-01031] Quishing Campaign Leveraging E-Wallet "Money Packet" Lure for Telegram Account Takeover

* **PhishHunt Case ID:** `PH-2026-01031`
* **Status:** Reported (First Submitter)
* **Classification:** `TLP:CLEAR` | `PAP:AMBER`
* **Impersonated Brand:** E-Wallet (Touch 'n Go eWallet)
* **Attack Vector:** QR Code Phishing (Quishing) via Meta Threads → Telegram Session Hijacking (ATO)
* **Discovery Date:** November 13, 2026

---

## Executive Summary

During routine threat hunting on Malaysian social media feeds, an active QR code phishing (**quishing**) campaign was discovered on **Meta Threads** (`@sitt.ifatimahh`). The lure impersonates a legitimate **E Wallet (Touch 'n Go eWallet) Money Packet** giveaway (*"Alhamdulillah Harini ada sikit rezeki RM100"*), instructing victims to download and scan the QR code from their mobile gallery.

Static triage via **AegisDome**, dynamic analysis in an isolated virtual machine, and infrastructure enrichment on **PhishHunt** revealed that the campaign does not steal eWallet credentials directly. Instead, it uses the fake reward screen as a social-engineering funnel to execute a **Telegram Account Takeover (ATO)** by harvesting Malaysian (`+60`) phone numbers and OTP verification codes. Infrastructure pivoting uncovered a dual-layer setup connecting a 12-day-old Cloudflare-fronted domain (`app-nf[.]com`) to an Indonesian-hosted campaign cluster (`mobile20[.]my[.]id` on `AS135444`).

---

## Step-by-Step Investigation

### Step 1: Threat Discovery & Social Media Lure Analysis
While monitoring the *Malaysia Threads* feed, a post by account `@sitt.ifatimahh` was flagged displaying a graphic designed to mimic an official Touch 'n Go eWallet Money Packet transfer.

* **Psychological Hook:** The caption uses religious gratitude phrasing (*"Doakan saya dimurahkan rezeki n dipermudahkan segala urusan..."*) to establish trust and lower skepticism.
* **Filter Evasion:** By embedding the malicious URL inside an image-based QR code and instructing users to *"Imbas dari galeri setelah anda muat turun imej Kod QR"* (Scan from gallery after downloading), the threat actor bypasses Meta Threads' automated URL text scanning and pushes the user onto their mobile device.

<p align="">
  <img width="450" alt="image" src="https://github.com/user-attachments/assets/c30881d3-583b-4a64-be14-150e39b40693" />
</p>


---

### Step 2: QR Decoding & Static Triage (AegisDome)
Instead of scanning the QR code with a mobile phone, the image was extracted and passed into **AegisDome** to decode the URI safely and perform passive domain intelligence checks.

  <img width="1000" height="600" alt="image" src="https://github.com/user-attachments/assets/7faea4f1-444c-4d6f-8449-328f4f96c21d" />


* **Decoded Payload:**
  ```text
  hxxps://bantuan-tng[.]app-nf[.]com/gf?utm_campaign=brand_awareness_2026&utm_creative_format=mobile_interstitial&device=desktop&gclid=GL_40716e71010b176b1867b9b1&variant_id=B&utm_source_platform=manual_gen&lang=id&utm_content=video_short&utm_term=beli_sekarang_diskon&utm_id=CAMP_823&session_id=uj8a6acbba5958
  ```

  **Key Triage Findings:**
  
    * **Brand Typosquatting:** Subdomain bantuan-tng combined with page title "Sampul Wang Kejutan untuk anda!" directly impersonates TNG eWallet.

    * **Suspicious Query Strings:** Synthetic ad-tracking parameters (gclid, utm_campaign, session_id) are appended to make the URL look like a legitimate marketing deep link and evade simple regex blocklists. Notably, lang=id and utm_term=beli_sekarang_diskon indicate Indonesian-language kit origins.

    * **Newly Registered Domain (NRD):** Root domain app-nf[.]com was registered on November 1, 2026 (12 days prior to detection) via Spaceship, Inc. and masked behind Cloudflare (172.67.190[.]144, AS13335).

### Step 3: Dynamic Sandbox Execution (VM Detonation)
  To observe the phishing kit's client-side behavior and credential-harvesting objective, the decoded URL was opened inside an isolated Virtual Machine:

   **Stage 1:** Fake Reward Landing Page: The page renders a blue gift-box animation branded with the Touch 'n Go eWallet logo and a "Claim Now" button.
    
  <p align="">
  <img width="500" height="450" alt="image" src="https://github.com/user-attachments/assets/98d4f546-992a-4bf8-aa20-47327d2e8ec4" />
  </p>

    **Stage 2:** Randomized Reward Reveal: Clicking Claim Now triggers a simulated packet-opening animation displaying a randomized cash prize (observed values: RM 23.30 and RM 40.12 across separate sessions) with a "Lihat Butiran" (View Details) prompt.

  <p align="">
  <img width="500" height="450" alt="image" src="https://github.com/user-attachments/assets/c124befb-4f16-40f0-a697-3366d65ba084" />
  </p>

    **Stage 3:** Pivot to Telegram Account Takeover: Clicking Lihat Butiran abandons the eWallet theme entirely and loads a "Hubungkan Telegram" (Connect Telegram) verification screen requesting the victim's +60 phone number.

  <p align="">
  <img width="500" height="450" alt="image" src="https://github.com/user-attachments/assets/0a811824-870b-4865-b5d6-fcdf23eb51db" />
  </p>

    **Stage 4:** Input Validation Testing: Entering an invalid or over-length phone number triggers client-side regex validation ("Nombor tidak sah"), confirming the backend actively validates Malaysian MSISDN formatting before forwarding valid numbers to an automated Telegram API session-hijacking script (which subsequently requests the victim's login OTP and 2FA password).

  <p align="">
    <img width="500" height="450" alt="image" src="https://github.com/user-attachments/assets/e2eacf79-f975-4bc3-b23a-4c70208d9e82" />
  </p>


### Step 4: PhishHunt Submission & Infrastructure Enrichment
  The quishing URL was submitted to PhishHunt by Rectifyq as the first reporter (PH-2026-01031), followed by automated and manual telemetry lookups (RDAP, DoH, CertSpotter, RIPEstat, Shodan InternetDB, and Lookyloo).
  
  Enrichment revealed that the frontend (bantuan-tng[.]app-nf[.]com) connects into a broader Indonesian-hosted phishing cluster centered on mobile20[.]my[.]id:

  ### Infrastructure Enrichment

| Indicator / Attribute | Observed Value | Source |
| :--- | :--- | :--- |
| **Cluster Root Domain** | `mobile20[.]my[.]id` (Registered `2026-05-11` via `PT JC Indonesia`) | RDAP |
| **Origin IPv4 & rDNS** | `103[.]219[.]251[.]170` (`anoa[.]kencang[.]com`) | DoH / Lookyloo |
| **ASN / Hosting Provider** | `AS135444` (`IDNIC-IKUBARU-AS-ID - PT Ikubaru Indonesia`) | RIPEstat |
| **Name Servers** | `srv1[.]arenhost[.]com`, `srv2[.]arenhost[.]com` | RDAP |
| **Related Campaign Subdomains** | `reward-h3[.]mobile20[.]my[.]id`, `reward-h5[.]mobile20[.]my[.]id`, `reward-h7[.]mobile20[.]my[.]id`, `reward-h9[.]mobile20[.]my[.]id`, `reward-h10[.]mobile20[.]my[.]id` | Lookyloo / CertSpotter |
| **Co-Hosted Domains (SAN Cert)** | `*[.]bestincomeaffiliate[.]com`, `*[.]kontenkupro[.]com`, `www[.]indahcargo[.]mobile20[.]my[.]id` | CertSpotter (Let's Encrypt) |
| **Exposed Host Services** | Ports `21, 22, 53, 80, 443, 2082, 2083, 2086, 2087, 3306, 7080, 8001` (cPanel/WHM & MySQL exposed) | Shodan InternetDB |

---

### MITRE ATT&CK Mapping

| Tactic | Technique ID | Technique Name | Campaign Implementation |
| :--- | :--- | :--- | :--- |
| **Resource Development** | `T1583.001` | Acquire Infrastructure: Domains | Registered `app-nf[.]com` 12 days prior to launch; staged wildcard subdomains (`reward-h3`–`h10`) on `mobile20[.]my[.]id`. |
| **Initial Access** | `T1566.002` | Phishing: Spearphishing Link | Distributed malicious URL encoded inside a QR image on Meta Threads to evade link scanners. |
| **Defense Evasion** | `T1036.005` | Masquerading: Match Legitimate Name or Location | Impersonated Touch 'n Go eWallet branding, UI colors, and "Money Packet" terminology. |
| **Credential Access** | `T1111` | Multi-Factor Authentication Interception | Harvests `+60` phone numbers via fake Telegram verification portal to intercept SMS/in-app login codes. |

### Takeaways & Defensive Recommendations

* For Users: Legitimate Touch 'n Go eWallet Money Packets are claimed directly inside the official TNG eWallet app and never require linking or verifying a Telegram account.

* For Defenders / CTI Analysts: Monitor Certificate Transparency logs for reward-h*.mobile20[.]my[.]id and newly registered domains using bantuan-tng prefixes behind Cloudflare. Block outbound traffic to 103[.]219[.]251[.]170 and related cluster domains listed in iocs.csv.

