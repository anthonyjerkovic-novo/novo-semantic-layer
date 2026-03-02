# **Project Lonestar PRD**

Related links: [Figma Designs](https://www.figma.com/design/AHASadZysQxMC00aXA2spH/Novo-Funding?node-id=38547-50946&t=beAARt4cxk7X57UD-0) | [\#project-lonestar](https://novoplatform.slack.com/archives/C09BFNH0BJN/p1755797299146369) | [Eng Timeline](https://docs.google.com/spreadsheets/d/1kiUk20ujhLSSxwHWFTVCJrHvIQtNMswHABhYEwauflM/edit?gid=956105189#gid=956105189)| [Release Strategy](https://docs.google.com/document/d/1ueCAuVY3OtXSNxF9HbVGHwLpXwr0xdg3XSrmpHwGOnE/edit?tab=t.0)| [Demo Video](https://drive.google.com/file/d/1UGeK_C5UINf5OpoazlJGcOZUKuYaYSw3/view?usp=sharing)  
DRIs: 	[Rohini Pandhi](mailto:rohini@novo.co) (Product) | [Ryan Graves](mailto:ryan.graves@novo.co) (Design) | [Max Hutchison](mailto:max@novo.co) (Legal)  |   
[Shikhar Varshney](mailto:shikhar@novo.co) (Eng) | [Fiona Blumin](mailto:fiona.blumin@novo.co) (Analytics) | [Rubina Singh](mailto:rubina@novo.co) (Marketing) 

**PRD Approval Status**

| Approver | Status | Comments |
| ----- | ----- | ----- |
| [Max Hutchison](mailto:max@novo.co) (Legal) | Approved |  |
| [Edem Tsiagbey](mailto:edem@novo.co) (Compliance) | Approved |  |
| [Jackson Barnes](mailto:jackson@novo.co) (Credit) | Approved |  |
| [Chelsye Toliver](mailto:chelsye@novo.co) (Payments) | Approved |  |
| [Rubina Singh](mailto:rubina@novo.co) (Marketing) | In Review |  |
| [Rares Crisan](mailto:rares.crisan@novo.co) (Engineering) | In Review |  |

---

## **Overview**

On June 20, 2025, Texas signed House Bill 700 into law, effective Sept 1, 2025, which prohibits automatic debit mechanisms from merchant deposit accounts for certain commercial sales-based financing transactions (including merchant cash advances and revenue-based financing).

For Novo, this means all Texas MCA customers must actively authorize each payment monthly. Project Lonestar ensures compliance while minimizing friction, maintaining payment rates, and preserving trust with our SMB customers in Texas.

Our solution: introduce a One-Time Authorization (OTA) flow where customers approve a single ACH debit each month. This requires new UX, logging/receipt infrastructure, and customer communication to ensure understanding and adoption.

**Success Metrics**

| *Metric* | *Goal* | *Current Value* |
| :---- | :---- | :---- |
| Adoption | \>90% of Texas MCA customers successfully authorize their monthly payment by month 2 | 87.01%  ([Query](https://metabase.novohq.com/question/33556-of-tx-mcas-that-authorize-repayment-by-month-2)) |
| Engagement | Email and push notification CTRs \> 40%  SMS response CTRs \> 50%  | Email notifications ([Query](https://metabase.novohq.com/question/33610-open-rate-click-through-rate-mca-emails-tx-customers)):  Open rate: 57% Clicked on link: 2.57% *Note: \~10% of our emails are undeliverable* Push Notifications ([Query](https://metabase.novohq.com/question/33653-open-rate-click-through-rate-mca-push-notifications-tx-customers)): Open rate: 3.16% |
| Delinquency | Keep delinquency rates for Texas MCA customers within \+5% of baseline pre-law-change rates | Current delinquency rate (defined as \>=30 days past due): 28.43% ([Query](https://metabase.novohq.com/question/33606-mca-delinquency-rate-texas-customers-as-of-date?report_date=2025-09-24)) |

## **Messaging**

Copy will be developed and finalized by [Rubina Singh](mailto:rubina@novo.co) (Marketing) and [Anthony Gutierrez](mailto:anthony.gutierrez@novo.co) (Copywriting). Current copy suggested by Legal: “You’re approving a one-time debit of $\[amount\] on or after \[MM/DD\] from \[account\] to Novo Funding.”

We will also need the [customer.io](http://customer.io) templates outlined in a spreadsheet format so that engineering can connect the right actions to each email CTA. Please review and update [this spreadsheet](https://docs.google.com/spreadsheets/d/1lpuu0O6ia1tsjCsNqgBn_mhfRKqat8umxtaRZD3-1jc/edit?gid=0#gid=0) to keep track of all messaging we’ll need to complete for Lonestar.

---

## **Timeline / Release Planning**

✅ **v1** (Sept 1, 2025 deadline)

* Stopped auto-debit flow in TX  
* Option to do a manual payment for customers


**🚧 v2** (Nov 1, 2025 deadline) \- see [Figma](https://www.figma.com/design/AHASadZysQxMC00aXA2spH/Novo-Funding?node-id=38547-50946&t=sdNzn4zErqSDtICA-0) for full UX

* Email, push, and SMS notifications with “one-click” approval  
* Web and mobile approval flows for the customer  
* Logging of approvals in our event database for audit purposes  
* Confirmation of transfer for customer receipt/acknowledgement

---

## **v2 Requirements**

### **Must-Haves**

#### **One-Time, Accruing Authorization Flow**

Texas customers must actively approve each payment. *Why?* Required by state law; failure risks legal penalties.

Texas (HB 700\) \-\> One-Time Authorization (OTA) Behavior: 

* Approval/authorization scope: Novo will send an authorization request to customers for the next upcoming installment payment. This authorization will be valid for that installment, even in the case where there are insufficient funds in their account. In those cases, the authorization will be applied to future ACH debit transactions inclusive of penalties and late fees.

  * The language on the approval module will need to include some verbiage to let the customer know that this approval will be valid for that particular installment payment and any future penalties or late fees that might need to be applied in the future. We will cover the extra approval elements in our [MCA agreement](https://docs.google.com/document/d/1R2gcJAQfBIC8fQcfLHFz21xytvgziX5z/edit?tab=t.0).

  * We will not mention a specific dollar amount (“$X”) on that approval for the same reasons as above; any individual approval will be valid and inclusive of late payments that accrue for that particular installment. We will NOT void the approval on any amount changes (like adding late fees to the total); that approval will be valid for that installment due and will never expire.

  * We will not mention a specific account, as we will attempt to get the payment across any of their linked accounts (Novo DDA account \+ external accounts).

  * We WILL mention the due date for that installment on the approval.

  * There will only be one ACTIVE authorization request per business at any given point in time (not multiple authorizations per business to avoid complexity displayed to customers). The authorization request that a business sees will be inclusive of any previous unpaid principal balances, simple interest, and other penalties or late fees.

  * We will only send new authorization request:  
    * In case a new MCA is drawn and because of it the due amount becomes greater than the current pending authorisation request. In this case we will INVALIDATE the previous request.   
    * In case the customer becomes due in another cycle.  
    * In case of ACH returns.  
    * In case the Business Address is changed to TX, we will send an immediate authorization request if the customer has an upcoming payment or is past due. 

* Cancelations: Customers will not be able to cancel any approvals they have previously authorized through their dashboard or mobile app. If they really need to cancel their payment, they can contact our customer support at least 72 hrs ahead of the due date to file a ticket. Customer Support can then contact engineering (specifically [Shikhar Varshney](mailto:shikhar@novo.co)) to manually cancel the upcoming installment pull. 

* Partial capture (business debits): We may capture ≤ approved cap. In those cases, mark the approval used (partial). We will not need to send new authorizations to capture the remaining balance of a partial payment. We will use the same authorization request till the approval amount is fully utilized and link all the payments associated with that approval.

* Allow account waterfall: We can hop to other funding sources for a customer with different accounts for each approval. Each approval will have multiple payments associated with it.

* Manual payments: If the customer makes a manual payment, we will not link that payment to any authorization request. We will debit the remaining balance (≤ cap) with the subsequent successful approval and log that partial payment in our database. 

* Collections and Comms: If an installment is unpaid after the due date, then mark it as delinquent, suspend funding as necessary, display past-due banner/CTA, and send reminders (email/SMS/push). Late fees per policy apply.

* Retries: we will not retry ACH pulls for v1.

#### **Event Logging**

We must store all [approval evidence](#bookmark=id.cwot2towzlw4) in our event database. We do not need to store this in a document with any e-signature of the customer. We simply must be able to re-create the sequence of events (inputs and outputs of the customer interaction and decisions) for any ROIs or regulatory audits in the future. *Why?* Compliance & auditability.

To evidence the approval we must store (for at least 6 years) the following for each approved payment: 

* Date of approval,

* UTC and local timestamp, 

* User ID, 

* Payment Installment ID,

* Loan ID,

* Approval payment link,

* Due date, 

* Exact amount, 

* IP address / mobile device / session IDs, and 

* Any 2FA (if available) 

These should also be reflected for the customer in their Funding dashboard.

#### **Notifications**

We want to provide customers with email \+ mobile push notifications with as close to “one-click” approval flows as possible. See the [Figma](https://www.figma.com/design/AHASadZysQxMC00aXA2spH/Novo-Funding?node-id=38547-50946&t=sdNzn4zErqSDtICA-0) for cadence and notif details. *Why?* Drive awareness and minimize missed payments.

For SMS notifications \-\> our [Terms of Use](https://www.novo.co/legal/terms-of-use-agreement) obtain prior express consent for account-related calls/texts when a customer provides a phone number. For non-marketing messages (like monthly payment-approval nudges), TCPA requires PEC (not “written” marketing consent) so we can send without an extra opt-in toggle. Guardrails to stay within PEC:

* Content: strictly transactional (e.g., “Do you approve this month’s Novo Funding transfer? Reply YES to confirm your next payment.”), no upsell or marketing.

* Timing: send only 8am–9pm local time; respect the customer's time zone.

* Responses: 

  * If a customer responds with **YES** or **Y** or any caps-agnostic variation of that (“yes”, “yEs”, “y”, etc), send a confirmation message over SMS and email.

  * Honor the opt-out **STOP** responses automatically (but do not include stop messaging in SMS copy); maintain a real-time suppression list. If a user says STOPs, use email/in-app only thereafter.

  * Any other SMS based responses should encourage contacting Novo support for more assistance.

  * If no responses over SMS are provided, then maintain that installment’s approval state as “Pending”.

This keeps us compliant using existing consent, avoids a new capture flow, and aligns with TCPA/CTIA expectations for transactional notices.

#### **Early Actions**

We can allow customers to approve the next upcoming transfer up to 30 days before the due date. Each approval is a one-time, month-specific authorization ([OTA](#bookmark=id.7d3l1tb3dgw2)), not a standing or batch authorization.

*Important Note: We cannot pre-approve a block or series of future debits (quarterly, semi-annual, etc.). That will look to OCCC like recurring auto-pay by another name and raises our enforcement risk.*

#### **Confirmations** 

We should display confirmations of transactions in the Novo dashboard (on web and mobile apps) and over email. All “receipts” should confirm the authorization and payment were successful. *Why?* Transparency, builds trust.

#### **Auto-Debit Error Handling**

We should duplicate the UX that currently applies to auto-debit cases where the bank balance is not enough to cover the monthly payment. *Why?* Reduce delinquency cases.

Currently, when there are insufficient funds in a customer’s account: 

* Novo will first attempt to pull the funds from the user's account they selected when making their initial MCA draw (default is their ‘available Novo balance’)

  * *If all funds are secured → the process is complete*

* If there is a remaining amount to collect, Novo then attempts to collect from any linked external accounts (in order from highest balance accounts to lowest)

  * *If all funds are secured → the process is complete*

* If there is a remaining amount to collect after all accounts are zeroed, Novo will continue to try to collect whatever they can from all accounts and flag the transfer as delinquent and begin the collections process:

  * The users funding will be locked until transfer is completed.

  * Within the Novo application, a large banner (see figma) will appear on the users Funding dashboard detailing the users lack of funds, the minimum amount due, and CTA to add funds to regain funding access.

  * Email notifications will also be sent to encourage the user to add funds to their account in order for the auto-dept to be collected.

  * Late fees will also be applied to delinquent accounts.

* US patterns we should replicate for non-auto-debit businesses in Texas:

  * All transfer approval flows will be unique to this user group and therefore will require new elements.

  * Assuming the user has approved the transfer but their account lacks funds, we’ll use the existing collections flow and elements, including: collection attempts across all accounts, collections notifications and emails, late fees, and funding suspension.

    

**Pause Payments Handling:-**

1. **System Priority**

   * The **Pause Payment** feature will have the **highest priority** across the system at all times.

2. **Data Initialization**

   * At launch, a **script** will add all **pending approvals** for **Texas (TX) customers** into the system.

3. **Resuming Payments**

   * All existing **Pause Payments** for TX customers will be **lifted (i.e., payments resumed)** as part of the initial setup.   
   * The ones applied manually by Admin should not be lifted. We have to convey this information to Account servicing so that in case they have applied for some and now need to be lifted. [Shikhar Varshney](mailto:shikhar@novo.co)to find out how many are these currently.

4. **Payment Authorization Logic**

   * The system can **collect payments** only when there is a **valid customer authorization** associated with the approval sent.

   * If an **agent applies a Pause Payment**, even when a valid approval exists, the system will **not initiate payment collection**.

5. **Business Rule Enforcement**

   * **Pause Payments** will **always override** any other payment-related operations or approvals, maintaining **topmost priority** in execution logic.

6. **Comms and behaviour if Pause Payment is applied:-**   
   * We will not create approvals for Texas customers who have Pause Payments applied manually by Admin.  
   * We will simply not create approvals for these TX based customers who have pause payments applied at the time of loan creation, cycle change or any other triggers.  
   * We will not send approval comms of any kind when pause payments are applied.  
   * We will resume sending comms and immediately create a new approval when we resume Payments.

**Release Strategy:-** [https://docs.google.com/document/d/1ueCAuVY3OtXSNxF9HbVGHwLpXwr0xdg3XSrmpHwGOnE/edit?tab=t.0](https://docs.google.com/document/d/1ueCAuVY3OtXSNxF9HbVGHwLpXwr0xdg3XSrmpHwGOnE/edit?tab=t.0)

## **Open Questions**

| [Rubina Singh](mailto:rubina@novo.co) & [Anthony Gutierrez](mailto:anthony.gutierrez@novo.co) | What final wording should we use across in-app, dashboard, email, and notifications? From [Shikhar Varshney](mailto:shikhar@novo.co): “I still have concerns over the New re-approval emails copy. It does not feel too customer centric and these emails will surely create confusion. We \[also\] do not have emails for Address change, Cancellation and Returns” |
| :---- | :---- |
| [Chelsye Toliver](mailto:chelsye@novo.co) | Instead of initiating same-day ACH files at 10:30am ET with Settlement at 1:00pm ET, we can submit next-day files with the settlement date as the due date. It increases collection success rate by presenting debits early in banking day, and it avoids situations where customers spend money between balance check and payment.  |

## 

