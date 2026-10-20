# BEC Attack Simulation

## Overview

This lab demonstrates a Business Email Compromise (BEC)-related credential phishing
scenario using Kali Linux as the attacker system and Windows 11 as the victim endpoint.

The simulation is delivered in phases. **Phase 1** (browser-based access) is complete.
**Phase 2** (infrastructure realism) is complete. **Phase 3** (email delivery) is in
progress.

## Lab Environment

- **Attacker:** Kali Linux VM
- **Victim:** Windows 11 VM
- **Network:** VMware host-only (`192.168.23.0/24`)
- **Web Server:** Apache2
- **Server-Side Processing:** PHP
- **DNS:** dnsmasq (local resolver)
- **Monitoring Platform:** Wazuh (integrated)

All activity is conducted within a controlled virtual lab using test credentials.

## Attack Chain

### Phase 1 — Browser-Based Simulation (Complete)

1. **Reconnaissance:** Prepared the simulated target environment.
2. **Lure Development:** Created a Microsoft 365-themed login page.
3. **Infrastructure Setup:** Configured Apache2 and a PHP handler to receive submissions.
4. **Delivery:** Accessed the phishing page directly from the Windows VM browser.
5. **Victim Interaction:** Entered test credentials into the simulated login form.
6. **Credential Capture:** Submitted credentials through HTTP POST to `login.php`.
7. **Redirect:** Redirected the browser after submission.

### Phase 2 — Infrastructure Realism (Complete)

8. **DNS Infrastructure:** Deployed dnsmasq to serve a lookalike domain, replacing the
   raw IP address with a plausible hostname.
9. **Page Rebuild:** Rebuilt the login page as a Microsoft Entra ID SSO screen with a
   two-stage email-then-password flow.
10. **Landing Page:** Added a supplier billing portal and invoice.

### Phase 3 — Email Delivery (In Progress)

11. **Mail Infrastructure:** Deploy a local mail server for realistic delivery.
12. **Message Construction:** Build a business-realistic BEC lure.
13. **Delivery:** Send the message to the victim mailbox.
14. **Authentication Analysis:** Examine SPF, DKIM and DMARC outcomes.
15. **Detection:** Observe what Sysmon, Suricata and Wazuh record.

## Phishing Infrastructure

### DNS Layer

A local dnsmasq resolver provides the lookalike domain `northwlndtraders.com`, which
differs from the legitimate `northwindtraders.com` by a single character.

Full detail: [DNS Setup](../environment/DNS-Setup.md)

### Phishing Page

- **File Location:** `/var/www/html/invoice/NW-4471/index.html`
- **Source Location:** `assets/web/phishing.html`
- **Form Fields:** Email and password
- **Submission Method:** HTTP POST
- **Form Handler:** `login.php`

The page mimics a Microsoft Entra ID SSO prompt, framed as *"Sign in to continue to
Northwind Traders."* This framing explains why a supplier's billing portal presents a
Microsoft login, and mirrors the `login_hint` behaviour of real SSO links by prefilling
the victim address.

The flow is two-stage — email, then password — matching genuine Microsoft
authentication behaviour.

![Phishing Page](../screenshots/phishing-page-entra.png)

![Two-Stage Login](../screenshots/login-two-stage.png)

### Form Submission

Both fields reside in a single form, so the server receives one POST containing both
values despite the two-screen presentation.

![Form Action](../screenshots/form-action.png)

### Credential Capture

The PHP handler records the submitted data plus investigation metadata:

- Submitted email address
- Submitted password
- Source IP address
- Browser user-agent
- Timestamp

![Capture Script](../screenshots/capture-script.png)

### Credential Storage

Captured data is appended to `/var/www/html/stolen_creds.txt`.

- **Format:** One block per submission, including timestamp, source IP and user-agent
- **Purpose:** Demonstrate server-side capture and provide an incident timeline

![Captured Credentials](../screenshots/captured-creds-v2.png)

### Landing Page

After submission, the handler issues an HTTP 302 redirect to a simulated supplier
billing portal. This is more realistic than returning the user to the login screen —
real SSO redirects onward to the target application.

![Portal Landing](../screenshots/portal-landing.png)

![Invoice Page](../screenshots/invoice-page.png)

## Attack Execution

### Step 1: Access the Phishing Page

Reached via the lookalike domain rather than a raw IP:
