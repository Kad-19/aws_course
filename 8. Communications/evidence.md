arn:aws:sns:us-east-1:598387452623:training-notifications

Message "ID": c3fca9b4-a69f-5cc8-b386-cb485d753aa1

# Lab Evidence: Communications (SES, SNS, and AWS End User Messaging)

### 1. SNS Topic ARN with account partially masked
`arn:aws:sns:us-east-1:<REDACTED_ACCOUNT_ID>:training-notifications`

### 2. SNS publish message ID
`c3fca9b4-a69f-5cc8-b386-cb485d753aa1`

### 3. SES sandbox/identity observation and whether a real email was sent
I observed that the SES account is currently in the sandbox environment and I do not have a verified identity. Because of these constraints, a real email was not sent. Before sending a production email, I would need to either verify both the sender and recipient email addresses (to send within the sandbox), which would allow sending to unverified recipients.

### 4. End User Messaging readiness checklist
Since account registration is incomplete, AWS End User Messaging was not available. Before sending SMS or user messages, the following setup is required:
*   **Sender Identity:** A dedicated origination identity (e.g., 10DLC, Short Code, Toll-Free Number, or alphanumeric Sender ID) must be secured based on the destination country requirements.
*   **Opt-in:** A compliant system must be established to track user consent before sending messages and to properly handle mandatory carrier keyword responses like STOP or HELP.
*   **Spend Limit:** The default sandbox enforces a strict $1.00 monthly spending limit. A quota increase must be formally requested via the AWS Service Quotas console to allow for production volume.
*   **Channel Registration:** Brand and Campaign use-case registrations (such as 10DLC registration in the US) must be fully approved by telecommunication carriers before message traffic is allowed.

### 5. Three Use-Case Decisions:

*   **Receipt Email:** **Amazon SES** is the appropriate choice. It is specifically designed for high-volume, programmatic outbound email, providing SMTP interfaces, template support, and detailed bounce/complaint deliverability metrics.
*   **Internal Fanout Event:** **Amazon SNS** is the appropriate choice. Its pub/sub architecture natively excels at taking a single internal system event and distributing it to multiple downstream subscribers (like SQS, Lambda, or admin emails) simultaneously.
*   **SMS OTP:** **AWS End User Messaging SMS** is the appropriate choice. It natively handles the complex global telecommunications layer, including compliance registrations, routing, and sender ID management required for reliable One-Time Password delivery.