# Frappe Mail Enhanced Report

## Overview
**Frappe Mail** is a comprehensive email management platform integrated with the **Frappe Framework**, designed to streamline email communication for businesses using ERPNext or other Frappe-based applications. Built to handle sending, receiving, and managing emails, it offers robust features like SMTP/IMAP connection pooling, DKIM signing, email queuing, and advanced reporting. The app leverages the **Stalwart Mail Server** as its backend **Mail Agent** to ensure reliable and secure email transactions, making it scalable for businesses of all sizes.

This report provides a detailed analysis of Frappe Mail’s architecture, functionality, setup process, and real-time email flow. It is intended for developers and consultants with expertise in ERPNext and the Frappe Framework, offering technical insights, best practices, and step-by-step instructions for implementation.

## Architecture
Frappe Mail operates as a client-server system, with the **Frappe Mail App** (running on a Frappe Bench) interacting with the **Stalwart Mail Server** for email processing. The architecture is designed for scalability, security, and seamless integration with the Frappe ecosystem.

### Components
1. **Frappe Mail App**:
   - Built on the Frappe Framework (Python, MariaDB, JavaScript).
   - Provides a user-friendly frontend UI (`/mail`) and RESTful APIs for email management.
   - Manages email domains, accounts, queues, and settings via Frappe DocTypes.
   - Integrates with ERPNext for notifications, workflows, and CRM.
2. **Stalwart Mail Server**:
   - A Rust-based mail server handling SMTP and IMAP operations.
   - Manages email queuing, DKIM signing, and connection pooling.
   - Ensures secure email delivery with support for SPF, DKIM, and DMARC.
3. **Frappe Ecosystem**:
   - Enables centralized email management within ERPNext or other Frappe apps.
   - Supports custom integrations via APIs and Server Scripts.

### Technical Stack
- **Backend**: Frappe Framework (Python 3.8+, MariaDB, Redis), Stalwart Mail Server (Rust).
- **Frontend**: Frappe’s JavaScript-based UI (likely Vue.js or jQuery).
- **APIs**: RESTful endpoints for authentication, sending, and receiving emails.
- **Dependencies**: Frappe Bench, Node.js (for development tools), Docker (optional).

## Key Features
Frappe Mail offers a robust set of features for email management:
- **SMTP/IMAP Connection Pooling**: Optimizes performance for high-volume email traffic.
- **Email Security**: Supports DKIM, SPF, DMARC for authenticated and secure emails.
- **Email Queuing**: Manages outbound and inbound emails with retry mechanisms for failed deliveries.
- **Advanced Reporting**: Tracks delivery status, bounce rates, and performance metrics.
- **Frontend UI**: Provides an intuitive interface for composing, sending, and managing emails (Inbox, Sent, Drafts).
- **APIs**: Enables programmatic email sending (`/outbound/send`, `/outbound/send-raw`) and receiving (`/inbound/pull`, `/inbound/pull-raw`).
- **Scalability**: Handles high email volumes with load-balanced Mail Agents.
- **Future Features**: Customizable email templates (upcoming).

## Email Flow
The flow for sending and receiving emails in Frappe Mail involves interactions between the Frappe Mail App, Stalwart Mail Server, and external mail servers. Below is a detailed breakdown of the real-time email flow.

### Sending Emails
1. **User Action**:
   - Users compose emails via:
     - **Mail UI**: Access `/mail`, click **Compose**, and enter recipient, subject, and body.
     - **Desk Interface**: Create an **Outgoing Mail** document in Frappe Desk.
     - **API**: Send a `POST` request to `/outbound/send` or `/outbound/send-raw`.
   - Example API request:
     ```json
     {
       "from_": "user@example.com",
       "to": "recipient@example.com",
       "subject": "Test Email",
       "html": "<p>Hello, this is a test email!</p>"
     }
     ```
2. **Frappe Mail App**:
   - Validates sender permissions and email ownership (`/auth/validate`).
   - Creates an **Outgoing Mail** document in the Frappe database.
   - Adds the email to the **outbound queue**.
3. **Stalwart Mail Server**:
   - Retrieves the email from the queue.
   - Applies DKIM signing (if enabled) using the domain’s private key.
   - Sends the email via SMTP to the recipient’s mail server.
4. **Delivery**:
   - The recipient’s mail server receives the email.
   - If delivery fails, the email is moved to an **error queue** for retry or manual resolution.
   - The Frappe Mail app updates the **Outgoing Mail** document with the status (Sent, Queued, Failed).
5. **Response**:
   - For API requests, returns a UUID (e.g., `"019300a4-91fc-741f-9fe5-9ade8976637f"`).

### Receiving Emails
1. **Email Arrival**:
   - An external mail server sends an email to the configured domain (e.g., `user@example.com`).
   - The Stalwart Mail Server receives the email via IMAP, authenticates it (SPF, DKIM, DMARC), and places it in the **inbound queue**.
2. **Frappe Mail App**:
   - Periodically pulls emails using the `/inbound/pull` API.
   - Example API request:
     ```http
     GET /inbound/pull?email=user@example.com&limit=50
     ```
   - Stores emails as **Incoming Mail** documents, classified into Inbox or Spam.
3. **User Access**:
   - Users view emails via the **Mail UI** (`/mail`) or **Incoming Mail** documents in Frappe Desk.
   - API responses include email details (e.g., subject, body, `created_at`).
4. **Queue Management**:
   - The inbound queue ensures orderly processing, with error handling for issues like invalid credentials.

### Key DocTypes
- **Mail Settings**: Configures root domain, DNS provider, and email limits.
- **Mail Agent**: Connects to the Stalwart Mail Server (API key or credentials).
- **Mail Agent Group**: Groups multiple agents for scalability.
- **Mail Domain**: Manages domains and DNS records (SPF, DKIM, DMARC).
- **Mail Account**: Links email addresses to Frappe users.
- **Outgoing Mail**: Stores sent emails and their status.
- **Incoming Mail**: Stores received emails and metadata.
- **Mail Queue**: Manages outbound and inbound email queues.

## Setup and Configuration
To explore Frappe Mail in real-time, follow these steps to set up the app and Stalwart Mail Server, configure a domain, and send/receive emails. This assumes a Ubuntu 20.04+ server, a Frappe Bench environment, and a domain (e.g., `example.com`) with DNS access.

### Prerequisites
- **System**:
  - Ubuntu 20.04+ or compatible OS.
  - Frappe Bench (Python 3.8+, MariaDB, Redis, Node.js).
  - Docker (optional for containerized setup).
  - Domain name with DNS access (e.g., DigitalOcean, Namecheap).
- **Tools**:
  - Terminal (SSH for production).
  - Browser for Frappe Desk and Mail UI.
  - Text editor (e.g., VS Code).
  - API client (e.g., Postman, `curl`).

### Step 1: Install Stalwart Mail Server
1. **Install**:
   - SSH into your server as root.
   - Run:
     ```bash
     curl --proto '=https' --tlsv1.2 -sSf https://get.stalw.art/install.sh -o install.sh
     sudo sh install.sh
     ```
   - Note the administrator credentials.
2. **Configure DNS**:
   - Set a **Reverse DNS (rDNS)** record for your server’s IP (e.g., `mail.example.com`).
   - Example: For IP `192.0.2.1`, set rDNS to `mail.example.com` via your hosting provider.
3. **Configure Admin Panel**:
   - Access the Stalwart Admin Panel (e.g., `https://mail.example.com:8080`).
   - Set **Network Settings > Hostname** to `mail.example.com`.
   - Configure **TLS Certificates** (Server > TLS > ACME Providers) with Let’s Encrypt or upload your own.
   - Restart:
     ```bash
     sudo systemctl restart stalwart
     ```
4. **Verify**:
   - Check status:
     ```bash
     sudo systemctl status stalwart
     ```

### Step 2: Install Frappe Mail
1. **Set Up Frappe Bench** (if not installed):
   ```bash
   sudo apt-get update
   sudo apt-get install python3-dev python3-venv python3-pip redis-server mariadb-server
   pip3 install frappe-bench
   bench init frappe-bench
   cd frappe-bench
   ```
   - Configure MariaDB:
     ```sql
     CREATE DATABASE frappe;
     GRANT ALL PRIVILEGES ON frappe.* TO 'frappe'@'localhost' IDENTIFIED BY 'frappe_password';
     ```
2. **Install Frappe Mail**:
   ```bash
   bench get-app mail
   bench new-site mail.example.com --db-name frappe --db-password frappe_password --install-app mail
   ```
3. **Production Setup** (optional):
   ```bash
   bench setup production frappe
   ```
4. **Access Site**:
   - URL: `http://mail.example.com`.
   - Credentials: `Administrator` / `admin`.

### Step 3: Configure Frappe Mail
1. **Mail Settings**:
   - In Frappe Desk, go to **Mail Settings**.
   - Set **Root Domain Name** (e.g., `example.com`).
   - Enable DigitalOcean DNS integration (if applicable) or prepare for manual DNS setup.
2. **Mail Agent Group**:
   - Create a new **Mail Agent Group**.
   - Set **Host** to `mail.example.com` (or `localhost` for local setups).
3. **Mail Agent**:
   - Create a new **Mail Agent**.
   - Select the Mail Agent Group.
   - Provide the Stalwart API key or username/password.
4. **DNS Records**:
   - Create a **Mail Domain** for `example.com`.
   - Add DNS records to your registrar:
     - **MX**: `mail.example.com` (priority 10).
     - **SPF**: `v=spf1 a mx include:mail.example.com ~all`.
     - **DKIM**: Use the key provided by Frappe Mail (e.g., `selector._domainkey`).
     - **DMARC**: `v=DMARC1; p=none; rua=mailto:dmarc@example.com;`.
   - Verify records in the **Mail Domain** document (Actions > Verify DNS Records).
5. **Mail Account**:
   - Create a new **Mail Account**.
   - Set:
     - **Domain**: `example.com`.
     - **User**: A Frappe user with **Mail User** role.
     - **Email**: `user@example.com`.

### Step 4: Send a Real-Time Email
1. **Mail UI**:
   - Go to `http://mail.example.com/mail`.
   - Click **Compose**.
   - Enter:
     - **From**: `user@example.com`.
     - **To**: `test@gmail.com`.
     - **Subject**: “Frappe Mail Test”.
     - **Body**: “This is a test email from Frappe Mail.”
   - Click **Send**.
   - Verify receipt in `test@gmail.com`’s inbox.
2. **Desk Interface**:
   - In Frappe Desk, create a new **Outgoing Mail** document.
   - Set:
     - **From**: `user@example.com`.
     - **To**: `test@gmail.com`.
     - **Subject**: “Desk Test Email”.
     - **Body**: `<p>Test email from Desk.</p>`.
   - Save and submit to send.
3. **API**:
   - Generate an API key for the Administrator:
     - In Frappe Desk, go to **User > Administrator > API Access**.
   - Send a `POST` request:
     ```bash
     curl -X POST http://mail.example.com/api/method/mail.api.outbound.send \
       -H "Authorization: Bearer <api_key>:<api_secret>" \
       -H "Content-Type: application/json" \
       -d '{
         "from_": "user@example.com",
         "to": "test@gmail.com",
         "subject": "API Test Email",
         "html": "<p>Test email via API.</p>"
       }'
     ```
   - Verify the response and email receipt.

### Step 5: Receive a Real-Time Email
1. Send an email from `test@gmail.com` to `user@example.com`.
2. Check the **Mail UI** (`/mail`) or **Incoming Mail** documents in Frappe Desk.
3. Use the API to pull emails:
   ```bash
   curl -X GET "http://mail.example.com/api/method/mail.api.inbound.pull?email=user@example.com&limit=10" \
     -H "Authorization: Bearer <api_key>:<api_secret>"
   ```
   - Expected response:
     ```json
     {
       "message": {
         "mails": [
           {
             "id": "<UUID>",
             "folder": "Inbox",
             "created_at": "2025-08-13 13:42:00+05:30",
             "subject": "Reply to Test Email",
             "html": "<p>Hi, this is a reply!</p>",
             "from": "test@gmail.com",
             "to": ["user@example.com"]
           }
         ],
         "last_synced_at": "2025-08-13 13:42:55+05:30",
         "last_synced_mail": "<UUID>"
       }
     }
     ```

### Step 6: Monitor and Troubleshoot
- **Queue**: Check **Mail Queue** for outbound/inbound emails and resolve error queue issues.
- **Logs**:
  - Frappe: `frappe-bench/logs/`.
  - Stalwart: `/var/log/stalwart`.
- **DNS**: Verify records with `dig` or `mxtoolbox.com`:
  ```bash
  dig TXT example.com
  dig TXT selector._domainkey.example.com
  ```
- **Issues**:
  - Check Stalwart status: `sudo systemctl status stalwart`.
  - Ensure ports (25, 465, 587, 143, 993) are open.
  - Test deliverability with `mail-tester.com`.

## Best Practices
1. **Security**:
   - Use strong DMARC policies (`p=quarantine` or `p=reject`) in production.
   - Store API keys in Frappe’s encrypted fields.
   - Enable TLS in Stalwart for secure transmission.
2. **Scalability**:
   - Use multiple Mail Agents in a **Mail Agent Group** for load balancing.
   - Optimize connection pooling in **Mail Settings**.
3. **Integration**:
   - Create Server Scripts for ERPNext workflows:
     ```python
     import frappe
     from frappe.utils import get_request_session

     def send_invoice_email(doc, method):
         session = get_request_session()
         response = session.post(
             "http://mail.example.com/api/method/mail.api.outbound.send",
             headers={"Authorization": "Bearer <api_key>:<api_secret>"},
             json={
                 "from_": "billing@example.com",
                 "to": doc.customer_email,
                 "subject": f"Invoice {doc.name}",
                 "html": f"<p>Dear {doc.customer_name},<br>Your invoice {doc.name} is ready.</p>"
             }
         )
         frappe.log(response.json())
     ```
4. **Testing**:
   - Use a test domain to avoid production disruptions.
   - Test with major providers (Gmail, Outlook) to ensure deliverability.
5. **Maintenance**:
   - Monitor **Mail Queue** for stuck emails.
   - Regularly update Frappe Mail and Stalwart for security patches.

## Conclusion
Frappe Mail is a powerful email management solution for the Frappe Framework, offering seamless integration with ERPNext, robust security, and scalability. By following the setup and testing steps, developers can implement real-time email functionality and integrate it with business workflows. This report provides a comprehensive guide for mastering Frappe Mail, with actionable steps for configuration, usage, and customization.

For further assistance, refer to the [Frappe Mail GitHub repository](https://github.com/frappe/mail) or contact the Frappe community for support.