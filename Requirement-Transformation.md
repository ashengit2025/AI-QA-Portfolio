# AI-QA Case Study: Requirement Transformation

## 1. The Raw Requirement (Input)
"We need a 'Cancel Order' button for the Kitch app. It should only work if the kitchen hasn't started cooking yet. If they already started, tell the user they can't cancel. Also, we need to make sure the refund happens automatically to their card. Oh, and send them a confirmation email."

## 2. The AI-Augmented Process
**Tools used:** Claude Sonnet 4.5 
**Technique:** System Prompting (Senior QA Persona)

Act as a Senior QA Automation Lead and Business Analyst. Your goal is to transform messy, raw requirements into professional, developer-ready Jira tickets.
For every requirement I provide, you must output the ticket in the following structure:
1. **Title**: [Component] - Brief, descriptive action (e.g., [Login] Implement Multi-Factor Authentication).
2. **User Story**: As a [User Role], I want [Action] so that [Value].
3. **Acceptance Criteria**: A numbered list of "Must-haves" (functional and non-functional).
4. **Technical Constraints**: Any API limits, security rules, or architectural requirements mentioned.
5. **Negative Test Scenarios**: A list of "What could go wrong" cases to test.
6. **Gherkin Scenarios**: At least 2 "Given-When-Then" scenarios for automation scripts.

Maintain a professional, technical, and objective tone. If a requirement is vague, add a "Clarification Questions" section at the bottom asking for missing details to help the Engineering team.


## 3. The Professional Output (Jira Ticket)

1. Title
[Order Management] - Implement Cancel Order with Pre-Cooking Validation and Automated Refund

2. User Story
As a Kitch app user,
 I want to cancel my order before the kitchen starts cooking,
 so that I can receive an automatic refund and avoid paying for food I no longer want.

3. Acceptance Criteria
A "Cancel Order" button must be visible on the active order details screen.
The button must only be enabled/clickable when the order status is in a pre-cooking state (e.g., "Pending", "Confirmed", "Preparing").
If the order status is "Cooking", "Ready", or "Completed", the button must be disabled or hidden.
When a user attempts to cancel an order that is already cooking, the system must display an error message: "Unable to cancel. Your order is already being prepared by the kitchen."
Upon successful cancellation, the system must automatically initiate a refund to the original payment method (card).
The order status must be updated to "Cancelled" in the database immediately after cancellation is confirmed.
A confirmation email must be sent to the user's registered email address containing:
Order ID
Cancellation timestamp
Refund amount
Expected refund processing time (e.g., "3-5 business days")
The cancellation action must be logged with user ID, order ID, and timestamp for audit purposes.
The UI must provide real-time feedback (e.g., loading spinner) during the cancellation process.
If the refund process fails, the system must log the error, notify the user, and queue the refund for manual review.

4. Technical Constraints
Order Status Dependency: Cancellation eligibility depends on real-time order status from the kitchen management system. Integration latency must be < 2 seconds.
Payment Gateway Integration: Refund processing must comply with the payment provider's API (e.g., Stripe, PayPal) with proper idempotency keys to prevent duplicate refunds.
Refund Processing Time: Acknowledge that actual refund to the customer's card may take 3-5 business days depending on the bank/card issuer.
Email Service: Use the existing email service provider (e.g., SendGrid, AWS SES) with proper error handling and retry logic.
Concurrency Handling: Implement optimistic locking or database transactions to prevent race conditions if the kitchen updates order status while cancellation is in progress.
Authorization: Only the user who placed the order can cancel it (validate user session/token against order ownership).
Idempotency: Prevent duplicate cancellation requests if the user clicks the button multiple times.

5. Negative Test Scenarios
User attempts to cancel an order that is already in "Cooking" status → System displays error message and prevents cancellation.
User attempts to cancel an order that has already been cancelled → System displays "Order already cancelled" message.
Network timeout during cancellation request → System retries or displays appropriate error message; order status remains unchanged.
Payment gateway refund API fails or times out → System logs error, notifies user of delay, and queues for manual processing.
User clicks "Cancel Order" multiple times rapidly → System processes only one cancellation request (idempotency check).
Email service is down or fails to send confirmation email → Cancellation and refund still proceed; email is queued for retry.
Order status changes from "Pending" to "Cooking" during cancellation processing → System detects status change and aborts cancellation with appropriate error message.
Unauthorized user attempts to cancel another user's order → System returns 403 Forbidden error.
User with insufficient network connectivity submits cancellation → System handles timeout gracefully with user-friendly error message.
Refund amount calculation error (e.g., promotions, discounts applied) → System calculates correct refund amount including all adjustments.
User's registered email address is invalid or bounces → System logs email failure but completes cancellation and refund.
Database connection fails during status update → Transaction rolls back; user sees error and can retry.

6. Gherkin Scenarios
Scenario 1: Successful Order Cancellation Before Cooking
gherkin
Feature: Cancel Order Before Kitchen Starts Cooking


  Scenario: User successfully cancels order in pending status
    Given the user has an active order with ID "ORD-12345"
    And the order status is "Pending"
    And the user is authenticated and owns the order
    When the user clicks the "Cancel Order" button
    Then the system should display a confirmation dialog
    And when the user confirms the cancellation
    Then the order status should update to "Cancelled"
    And a refund of "$25.99" should be initiated to the original payment method
    And a confirmation email should be sent to "user@example.com"
    And the email should contain order ID "ORD-12345" and refund amount "$25.99"
    And the user should see a success message: "Your order has been cancelled and refund is being processed"
Scenario 2: Failed Cancellation Attempt After Cooking Started
gherkin
Feature: Prevent Order Cancellation After Cooking Starts


  Scenario: User attempts to cancel order that is already cooking
    Given the user has an active order with ID "ORD-67890"
    And the order status is "Cooking"
    And the user is authenticated and owns the order
    When the user navigates to the order details page
    Then the "Cancel Order" button should be disabled or hidden
    And when the user attempts to trigger cancellation via API
    Then the system should return an error response with status code 400
    And the error message should be "Unable to cancel. Your order is already being prepared by the kitchen."
    And the order status should remain "Cooking"
    And no refund should be initiated
    And no confirmation email should be sent
Scenario 3: Refund Processing with Email Confirmation
gherkin
Feature: Automated Refund and Email Notification


  Scenario: System processes refund and sends confirmation email
    Given an order with ID "ORD-11111" is successfully cancelled
    And the original payment amount was "$42.50"
    And the user's email is "customer@kitch.com"
    When the refund is initiated through the payment gateway
    Then the payment gateway should receive a refund request for "$42.50"
    And the refund should include idempotency key "REFUND-ORD-11111-TIMESTAMP"
    And upon successful refund initiation
    Then an email should be sent to "customer@kitch.com"
    And the email subject should be "Your Kitch Order #ORD-11111 Has Been Cancelled"
    And the email body should contain:
      | Field                   | Value                        |
      | Order ID                | ORD-11111                    |
      | Refund Amount           | $42.50                       |
      | Refund Processing Time  | 3-5 business days            |
      | Cancellation Timestamp  | [Current DateTime]           |
Scenario 4: Concurrent Status Change During Cancellation
gherkin
Feature: Handle Race Condition During Cancellation


  Scenario: Order status changes to cooking while cancellation is processing
    Given the user has an active order with ID "ORD-99999"
    And the order status is "Preparing"
    And the user initiates a cancellation request
    When the kitchen system updates the order status to "Cooking" during cancellation processing
    Then the system should detect the status change
    And the cancellation transaction should be rolled back
    And the user should receive an error message: "Unable to cancel. Your order is already being prepared by the kitchen."
    And no refund should be processed
    And the order status should remain "Cooking"

7. Clarification Questions
Order Status Definitions: What are the exact order status values in the system? (e.g., Pending, Confirmed, Preparing, Cooking, Ready, Completed, Cancelled). Which statuses allow cancellation?
Partial Refunds: Are there any scenarios where partial refunds apply (e.g., cancellation fees, service charges)? Or is it always a full refund?
Refund Failure Handling: If the automated refund fails, what is the escalation process? Should this create a support ticket? Who is notified?
Cancellation Window: Is there a time limit for cancellation? For example, can users cancel within 5 minutes of placing the order regardless of kitchen status?
Payment Methods: Do we support all payment methods for refunds (credit cards, debit cards, digital wallets)? Any payment methods that require manual refund processing?
User Confirmation: Should we show a confirmation dialog before cancelling (e.g., "Are you sure you want to cancel this order?") or proceed immediately?
Kitchen Notification: Should the kitchen be notified when an order is cancelled (via in-app notification, dashboard alert, etc.)?
Analytics/Tracking: Should cancellation events be tracked in analytics tools for business insights (cancellation rate, reasons, etc.)?
Multi-Item Orders: If an order contains multiple items, is it all-or-nothing cancellation, or can users cancel individual items?
Restaurant Compensation: Are there any business rules around restaurant compensation for cancelled orders (e.g., does the restaurant still receive a fee)?

## 4. QA Insight: Why this is better
By using an AI-QA workflow, I identified 3-4 risks not mentioned in the original request:
* **Race Conditions:** 
What if the kitchen starts cooking at the exact millisecond the user hits 'Cancel'?
What happens if 2 people click cancel on the same order at the same time?
* **Partial Refunds:** How do we handle service fees?
* **State Management:** The UI must update immediately to disable the button once clicked.
