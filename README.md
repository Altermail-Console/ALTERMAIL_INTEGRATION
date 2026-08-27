### The Ultimate Altermail SMTP & Webhook Integration Blueprint (Node.js)

This guide provides a comprehensive, production-ready blueprint for configuring **Altermail** inside a Node.js environment. It is explicitly optimized to bypass DigitalOcean port blocks, handle the Developer Plan's concurrency settings, verify cryptographic webhook signatures, and manage event retrieval.

* * *

### 1\. Environment Configuration

Create a `.env` file in the root folder of your Node.js application to safely store your secure platform credentials.

env

    ALTERMAIL_SMTP_USERNAME="yourdomain.com" # Your verified sender domain name
    ALTERMAIL_SMTP_PASSWORD="your_altermail_api_key" # Your secret API key acting as SMTP password
    ALTERMAIL_API_TOKEN="your_altermail_api_key" # Used for authenticating Event API requests
    ALTERMAIL_WEBHOOK_SECRET="your_webhook_secret_key" # Obtained from your Altermail Domains dashboard
    

Use code with caution.

* * *

### 2\. Production SMTP Client (Developer Plan Optimized)

The Developer plan restricts traffic to **5 concurrent connections**, **50 messages per session**, and **30 sends per minute**.

Instead of manual array throttling loops, configure **Nodemailer's connection pool engine**. This allows your app to fire thousands of emails simultaneously while Nodemailer manages connections safely in the background.

javascript

    import nodemailer from 'nodemailer';
    
    // Configure the Developer Plan Connection Pool
    const transporter = nodemailer.createTransport({
      host: '://altermail-console.com.ng',
      port: 2525,       // Bypasses DigitalOcean standard SMTP outbound port blocks completely
      secure: false,     // Must be false for port 2525 to support standard STARTTLS upgrade
      requireTLS: true,  // Mandatory in production according to Altermail specification
      auth: {
        user: process.env.ALTERMAIL_SMTP_USERNAME, 
        pass: process.env.ALTERMAIL_SMTP_PASSWORD, 
      },
      
      // DEVELOPER PLAN RATE-LIMIT ENFORCEMENT
      pool: true,             // Enables active persistent connection streaming
      maxConnections: 4,      // Safely stays under the plan's absolute 5 connection ceiling
      maxMessages: 45,        // Auto-recycles the socket cleanly before hitting the 50 message cap
      rateDelta: 60000,       // Sets the evaluation window to exactly 1 minute (60,000ms)
      rateLimit: 28           // Stays safely below the 30 sends-per-minute throttling mark
    });
    
    /**
     * Dispatches an email through the managed connection pool
     * @param {string} recipient - Target email address
     * @param {string} subject - Email subject line
     * @param {string} htmlContent - HTML layout content
     */
    async function sendDeveloperMail(recipient, subject, htmlContent) {
      const mailOptions = {
        // CRITICAL: The domain in 'from' must exactly match your authenticated user domain name
        from: `"Production App" <no-reply@${process.env.ALTERMAIL_SMTP_USERNAME}>`, 
        to: recipient, // Envelope limit: Total To + CC + BCC cannot exceed 50 addresses per message
        subject: subject,
        html: htmlContent,
      };
    
      try {
        const info = await transporter.sendMail(mailOptions);
        // info.response captures Altermail's response string (e.g., '250 OK messageId=UUID')
        console.log(`Dispatched to ${recipient}: ${info.response}`);
        return info;
      } catch (error) {
        console.error(`SMTP Send Failure for ${recipient}:`, error.message);
        throw error;
      }
    }
    
    // Example usage: Firing bulk emails concurrently 
    const bulkList = ['user1@example.com', 'user2@example.com', 'user3@example.com'];
    bulkList.forEach(email => {
      sendDeveloperMail(email, 'Transaction Confirmed', '<h1>Success</h1><p>Your order is processing.</p>');
    });
    

Use code with caution.

* * *

### 3\. Real-Time Delivery Webhook Receiver (Express)

The Developer plan limits delivery log retention on Altermail’s servers to **3 days**. To retain your logs permanently for user profiles or audit logs, utilize **Delivery Status Webhooks**.

This middleware reads the raw text body natively, computes the `"sha256="` prefix signature format, and drops the request if the validation fail matches.

javascript

    import express from 'express';
    import crypto from 'crypto';
    
    const app = express();
    
    /**
     * Cryptographically verifies Altermail's webhook signature
     * @param {string} rawBody - Unparsed text payload string
     * @param {string} signature - Value of the X-Altermail-Signature header
     * @param {string} secret - Your webhook secret signing key
     */
    function verifyWebhook(rawBody, signature, secret) {
      const expected = "sha256=" + crypto
        .createHmac("sha256", secret)
        .update(rawBody)
        .digest("hex");
        
      // Using timingSafeEqual protects your web app endpoint from timing attacks
      return crypto.timingSafeEqual(
        Buffer.from(expected, 'utf8'),
        Buffer.from(signature, 'utf8')
      );
    }
    
    // CRITICAL: Express must process the raw request payload to maintain signature integrity
    app.post("/webhook", express.raw({ type: "*/*" }), (req, res) => {
      const sig = req.headers["x-altermail-signature"];
      
      if (!sig) {
        return res.status(401).send("Missing signature header.");
      }
    
      // Verify signature using the raw text buffer string
      const rawBodyString = req.body.toString('utf8');
      if (!verifyWebhook(rawBodyString, sig, process.env.ALTERMAIL_WEBHOOK_SECRET)) {
        console.error("⚠️ Webhook validation failed! Fraudulent payload dropped.");
        return res.status(401).send("Invalid signature");
      }
    
      // Parse payload object safely after authentication passes
      const payload = JSON.parse(rawBodyString);
      const { event, messageId, timestamp, data } = payload;
      
      console.log(`[Webhook Verified] Event: ${event} | MessageId: ${messageId}`);
    
      // CRITICAL: Webhooks timeout after 3 seconds. Handle heavy DB logic asynchronously!
      processAsyncEvent(event, messageId, data).catch(err => console.error("DB Async Error:", err));
    
      // Return an immediate response to remain clear of the 3-second timeout window
      res.sendStatus(200);
    });
    
    async function processAsyncEvent(event, messageId, data) {
      const { recipientEmail, senderDomain } = data;
      
      switch (event) {
        case 'email.delivered':
          console.log(`[Success] Message ${messageId} successfully arrived at ${recipientEmail}`);
          // UPDATE YOUR DATABASE HERE: e.g., set status = 'delivered'
          break;
    
        case 'email.bounced':
          console.warn(`[Bounce Alert] Message ${messageId} to ${recipientEmail} hard bounced.`);
          // UPDATE YOUR DATABASE HERE: e.g., flag email as dead or suspend user notifications
          break;
    
        case 'email.sent':
          console.log(`[Queue Handoff] Message ${messageId} accepted by Altermail edge engine.`);
          break;
      }
    }
    
    app.listen(3000, () => console.log('Secure Altermail Webhook Processor active on Port 3000'));
    

Use code with caution.

* * *

### 4\. Querying the Delivery Events API (Fallback Engine)

Because Altermail webhooks do not automatically retry failed deliveries, use the **Delivery Events API** as a pull-based fallback option or for manual on-demand dashboard lookups.

javascript

    import fetch from 'node-fetch';
    
    /**
     * Actively pulls explicit delivery logs using your platform token
     * @param {string} messageId - The unique message UUID to investigate
     */
    async function queryEmailEventLog(messageId) {
      const baseUrl = 'https://altermail-console.com.ng';
      const url = `${baseUrl}?messageId=${encodeURIComponent(messageId)}`;
    
      try {
        const response = await fetch(url, {
          method: 'GET',
          headers: {
            // AUTHENTICATION: Altermail explicitly uses x-api-token for the Event query API
            'x-api-token': process.env.ALTERMAIL_API_TOKEN,
            'Accept': 'application/json'
          }
        });
    
        if (!response.ok) {
          throw new Error(`HTTP Operational Failure: ${response.status}`);
        }
    
        const eventLogs = await response.json();
        console.log(`--- Log Event Response for ${messageId} ---`);
        console.log(JSON.stringify(eventLogs, null, 2));
        return eventLogs;
      } catch (error) {
        console.error('Failed to extract logs from Event API:', error.message);
      }
    }
    
    // Call dynamically using the messageId retrieved during your SMTP or REST dispatch
    // queryEmailEventLog('71688732-3522-48fe-a86a-b3bdd19540ae');
    

Use code with caution.

* * *

### 5\. Architectural Checklist

*   **DigitalOcean Infrastructure, AWS and Azure:** Standard outbound email ports (25, 465, 587) are heavily firewalled on cloud nodes. Routing via **Port 2525** guarantees flawless delivery handshakes with zero custom firewall rule modifications.
*   **Payload Size Constraints:** Keep all text data, layout elements, and physical asset attachments below a hard combined size cap of **25 MB** per single envelope payload.
*   **Envelope Cap Rules:** Ensure that the sum total of your `To`, `CC`, and `BCC` fields does not pass **50 recipient addresses** inside a single transaction trigger.
