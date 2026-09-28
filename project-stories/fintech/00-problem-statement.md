# Problem Statement

## In two sentences

A growing fintech has transaction, customer, and accounting data spread across many systems. Finance, risk, and product each get different numbers, reports arrive late, and audits are painful.

## The company

A mid-size fintech offering payments and lending to consumers and small businesses. It grew fast by adding systems as it launched each product, so its data grew in pieces with no shared design.

## Where the data lives today

| Source | System | What it holds | How data can be pulled | Size / rhythm (confirm) | Example | Explanation |
|---|---|---|---|---|---|---|
| Payments | PostgreSQL (a database) on AWS RDS — the company's own payments app | Every transaction, refund, and chargeback (a disputed card payment) | CDC (Change Data Capture) from the database log | ~5 million transactions a day, changing constantly | A customer pays $50, later gets a $10 refund, or disputes a charge — each becomes a new entry the moment it happens. | A database is a large organized digital record, like a spreadsheet a computer can search instantly. This one is built and owned by the company to run its payments app. CDC means watching the database's internal log and getting told the instant something changes, instead of repeatedly asking it for updates. |
| Lending | Mambu (a cloud lending and core-banking platform) | Loans, repayment schedules, balances | Mambu API (Application Programming Interface), pulled every hour | ~500,000 active loans | A customer borrows $5,000; Mambu tracks their repayment schedule and how much they still owe. | Mambu is off-the-shelf software the company bought rather than built, used to run the loan side of the business. An API is a set of doors a piece of software opens so other programs can ask it for data, instead of a person clicking through a screen. |
| Accounting | NetSuite (an ERP — Enterprise Resource Planning — system, used here as the accounting software) | General ledger (the master list of all money in and out), invoices, journal entries | NetSuite API and scheduled saved-search exports, once a day | ~2 million ledger lines a month | A $100 sale becomes one line in the general ledger, pulled from NetSuite once a day. | An ERP is software that runs a company's core back-office operations; here it's the official book of record for accounting. |
| Settlement | Card processor and partner-bank files (for example Adyen), delivered over SFTP (a secure way of transferring files) | What was actually settled and paid out, after fees | Daily CSV files over SFTP | ~50 files a day | Adyen sends a daily file saying "yesterday we processed 10,000 card payments, and after fees, $980,000 was deposited into your bank account." | When a customer pays by card, a processor like Adyen moves the money between banks and card networks, then reports back what actually landed in the company's account. It is not an escrow (money deliberately held until a condition is met) — the delay is just how long it normally takes money to move through the banking system. |
| Customers | Salesforce (a CRM — Customer Relationship Management — tool) | Customer and business-account details | Salesforce API, once a day | ~1 million customers | Salesforce holds "Jane's Bakery LLC — signed up March 2025, contact jane@bakery.com, business type: retail." | A CRM is software that holds a profile for each customer or business and tracks the company's interactions with them. |
| Identity checks | Persona (a KYC — Know Your Customer — identity-verification service) | Results of customer identity and sanctions checks | Persona API and webhooks (automatic notifications sent the moment something happens) | ~10,000 checks a day | A new customer uploads a driver's license; Persona checks it is real and checks the person is not on a government sanctions list before they can open an account. | KYC is the one-time check at signup, confirming someone is really who they say they are. AML (Anti-Money Laundering) is the related legal requirement to keep watching transactions afterward for suspicious patterns — Persona handles the KYC part; ongoing AML transaction monitoring happens later, once data reaches the platform. |

The systems do not agree on basic things. Each has its own customer ID, its own timestamps and time zones, and its own version of a transaction's status.

*Example:* in Salesforce, a person is customer `CUST-4821`. In Mambu, the same person's loan is filed under customer `12345` — nothing links the two automatically. The payments database logs a transaction at `14:02 UTC`; NetSuite logs the same event as `9:02 AM Eastern` — same moment, different numbers on the clock. The payments system calls a transaction "pending" while Adyen's file calls the same thing "in process" — same real state, different words, so a program comparing them sees no match.

## What is going wrong

1. **Different numbers for the same question.** Finance reads revenue from NetSuite after fees and refunds. Product counts it from the payments database as gross payment volume. Risk counts it from Mambu's loan balances. So "how much did we make last month?" has three answers, and nobody can explain the gaps.
   *Example:* a customer pays $100, later gets a $5 refund, and Adyen takes a $3 fee. Product reports "$100 of sales." Finance reports "$92 of revenue." Risk isn't tracking this transaction at all — it tracks a different number, loan balances. None of the three is wrong; nobody agreed on the definition.
2. **Slow, manual reporting.** Each month, analysts export NetSuite, Mambu, and the settlement files into Excel and match them by hand. The month-end close takes about 10 days (confirm).
   *Example:* closing March's books means one analyst spends most of two weeks copying numbers out of three different systems into a spreadsheet and lining them up row by row.
3. **Late data.** Transaction data reaches analysts a day or more after it happens, so problems such as unusual payment patterns are spotted late.
   *Example:* a stolen card is used to submit 50 loan applications from the same IP address in one hour. Because the data only arrives in tomorrow's batch, nobody notices until the fraudster is already gone.
4. **Hard audits.** Nobody can quickly show where a number came from or who changed it, which makes SOX (Sarbanes-Oxley Act — a US law requiring public companies to prove their financial numbers are accurate) audits and regulator questions stressful.
   *Example:* an auditor asks "prove this $2M revenue figure is correct and show nobody edited it without a record." The only answer anyone can give is a spreadsheet someone remembers building by hand.
5. **Sensitive data spread everywhere.** Card and personal data is copied into many places with inconsistent access rules, a PCI-DSS (Payment Card Industry Data Security Standard — the card industry's security rulebook) and privacy risk.
   *Example:* that same month-end spreadsheet has real card numbers pasted into it, sitting in someone's email — exactly what PCI-DSS says not to do.
6. **Fragile pipelines.** Each team built its own scripts. When one breaks, nobody notices until a report looks wrong.
   *Example:* the script that pulls NetSuite data was written by someone who has since left the company. It silently stops working one week, and nobody notices until a report looks wrong three weeks later.

## Who is affected

- **Finance:** cannot close the books on time or trust the figures.
- **Risk and compliance:** see fraud and exposure late; struggle to prove controls to auditors.
- **Product and leadership:** make decisions on numbers that disagree.
- **Analysts:** spend most of their time gathering and fixing data instead of analyzing it.

## The goal

Build one trusted data platform that becomes the single source of truth for the company, led by the senior data engineer who owns the architecture.

## What the platform must do

- Bring in data from every source above, both as changes happen (transactions) and on a schedule (files, ERP).
- Keep the raw data untouched, then clean and organize it in clear stages.
- Provide one agreed definition for each business metric, used by everyone.
- Check data quality automatically and stop bad data before it reaches reports.
- Control who can see what, protecting sensitive card and personal data.
- Record where every number came from.
- Serve finance, risk, and leadership through dashboards.

## Out of scope

Building the fraud-detection model itself and the customer-facing product. The platform supplies clean, timely data to them.

## How success is measured (illustrative, confirm)

- Month-end close reduced from ~10 days to ~3 days.
- Transaction data available to analysts within minutes instead of a day.
- One certified definition for each key metric (revenue, active customers, default rate).
- Automated quality checks on all critical tables, with failures caught before reports are affected.
- Audit questions answered from recorded lineage in hours, not weeks.

## Constraints

- **Compliance:** PCI-DSS for card data, SOX for financial reporting, KYC/AML for customer and transaction monitoring.
- **Accuracy over speed:** financial figures must reconcile exactly.
- **No downtime** for the live payments system; data must be copied without slowing it.
- **Cost awareness:** compute is paid for by use and needs to be controlled.

## My role

Lead/senior data engineer. I owned the architecture: choosing the tools and layers, defining the data standards, guiding the team building it, and working with finance, risk, and compliance to agree what "correct" means.
