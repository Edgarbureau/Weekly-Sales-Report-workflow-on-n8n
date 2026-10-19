# Weekly-Sales-Report-workflow-on-n8n
# Automated Weekly Sales Report (n8n + Supabase + AI)

An n8n workflow that replaces a manual weekly reporting routine. Every Monday it pulls order data from a database, checks the data quality, saves the clean rows, logs the bad ones for review, writes a short AI summary, and emails a formatted report. If anything breaks, a separate workflow sends an alert.

All data in this project is fake and was created for demonstration.

![Workflow canvas](images/01-workflow-canvas.png)

## The problem

In many finance and operations teams, the weekly report is built by hand: export orders from the ERP into a spreadsheet, clean it, calculate totals, write a summary, and email it. It is slow, repetitive and error-prone. A single bad row can distort the totals, and a failed export can go unnoticed.

## The solution

A scheduled workflow that does all of it automatically and handles problems openly:

- Bad data is separated and logged with a reason, never silently dropped.
- Report figures are calculated by code, so they are exact.
- The AI only writes the narrative around those figures and is told not to invent numbers.
- If the workflow fails, a separate alert workflow tells someone straight away.

**Tools:** n8n Cloud, Supabase (PostgreSQL), Google Sheets, Gmail, OpenAI (GPT-5 mini).

## How it works

The main workflow has 10 nodes. After validation it splits: flagged rows go to the Exceptions sheet, and clean rows continue on to the report.

1. **Monday 8am trigger** starts the run on a schedule, so nobody has to remember it.
2. **Get last 14 days orders** queries Supabase with SQL and labels each row as this week or last week. The database does the filtering, and the label allows a week-over-week comparison without a second query.
3. **Validate rows** is one Code node that checks every row against five rules and adds `issues` and `is_valid` fields. It runs on all rows at once because duplicate detection needs to see earlier rows.
4. **Is row valid?** routes each row by reading the result of the previous step, so the rules live in one place only.
5. **Log flagged rows** appends each bad row, with its reason, to a Google Sheet that a non-technical person can open and act on.
6. **Save clean rows** upserts the valid rows into a `clean_orders` table, matched on `order_id`. Running the workflow twice does not create duplicates.
7. **Calculate report figures** reduces the clean rows to one summary: revenue this week and last week, percent change, top customers and the number of flagged rows. Code does the maths because language models can get arithmetic wrong.
8. **Build AI prompt** assembles the prompt text so it can be inspected before it is sent.
9. **Write AI summary** asks the model for a 5-line plain-English summary using only the supplied figures. A small model is enough for this, which keeps cost and delay low.
10. **Email report** sends the formatted HTML report. The figures in the email come from step 7, not from the AI, so they stay exact even if the wording changes.

The error workflow has 2 more nodes: an **Error Trigger** that starts when the main workflow fails, and a **Gmail** node that sends the alert. One central error handler is simpler than adding error branches to every node.

![Query node](images/06-sql-query-node.png)

## Data validation

The demo database holds 20 orders, 5 of them deliberately broken. Each row is checked for:

- Quantity greater than zero
- Unit price greater than zero
- A customer ID present
- An order date that is not in the future
- A unique invoice number (the first copy stays valid and later copies are flagged)

**Result:** 20 rows in, 15 clean, 5 flagged, each with the reason it failed.

![Validate rows output](images/03-validate-rows-output.png)

![Exceptions log sheet](images/04-exceptions-sheet.png)

![Clean orders table in Supabase](images/05-clean-orders-table.png)

## The report email

Kept minimal on purpose: one accent colour, grey labels, a short AI summary and a small table of key figures.

![Weekly report email](images/07-report-email.png)

## Failure alerts

If the main workflow fails, the alert email names the workflow, the node that failed, the error message, and links to the failed run.

![Error workflow](images/02-error-workflow.png)

![Failure alert email](images/08-failure-alert-email.png)

## Successful run

![Successful execution](images/09-successful-execution.png)

## Setup

1. **Database:** create a free Supabase project and, in the SQL Editor, create two tables. `orders` holds the source data (`order_id`, `invoice_no`, `customer_id`, `customer_name`, `region`, `quantity`, `unit_price`, `order_date`, `status`) and should be filled with sample rows, including a few deliberately bad ones. `clean_orders` has the same columns plus `week`, `issues` and `is_valid`, with `order_id` as the primary key.
2. **Connection:** in n8n, create a Postgres credential using the Supabase **Session pooler** details (host ending in `pooler.supabase.com`, port `5432`, user `postgres.<project-ref>`, SSL set to Require). The pooler is used because n8n Cloud connects over IPv4 and Supabase's direct address is IPv6 only.
3. **Sheet:** create a Google Sheet named `Exceptions log` with these headers in row 1: `order_id, invoice_no, customer_id, customer_name, region, quantity, unit_price, order_date, status, week, issues`.
4. **Build the workflows:** build the main workflow in n8n by following the node list and screenshots above, and build the alert workflow separately (an Error Trigger followed by a Gmail node). Publish the alert workflow, then connect the Postgres, Google Sheets, Gmail and OpenAI credentials.
5. **Link the alert:** in the main workflow's Settings, set Error Workflow to the alert workflow, and set your own email address in both Gmail nodes.
6. **Test:** run the main workflow manually. To test the alert, point the query at a column that does not exist and publish with a 1-minute schedule, because manual runs never trigger the error workflow. Restore the query and the weekly schedule afterwards.

## Design decisions

- **Simple over clever:** one Code node holds all the validation rules, which keeps the canvas short and the logic easy to read.
- **Code calculates, AI writes:** this avoids arithmetic errors in the report.
- **Bad data is kept, not deleted:** the Exceptions sheet gives a reviewer the reason for every rejected row.
- **Failures are loud:** a dedicated alert workflow means a broken report never goes unnoticed.
- **Row Level Security** is enabled on both tables, so they are not exposed through Supabase's public API. The workflow connects with the database login and is unaffected.

## Known limitations

- The Exceptions sheet uses Append, so re-running adds the same flagged rows again. A production version would clear the sheet each run or deduplicate on `order_id`.
- The demo uses fake data in Supabase to stand in for an ERP. A real deployment would connect to the company's own database or API.
- SSL certificate checking may need to be relaxed for the pooler connection in a demo. A real client setup should use the proper certificate.

## Next steps

- Connect Power BI or Looker Studio to the `clean_orders` table.
- Add a chat-based AI agent that answers questions such as "sales by region last week?" from the database.

## Author

**[Oko Edgar]**
[www.linkedin.com/in/edgar-oko-3861b9421] | [Edgaroko90@gmail.com]

Available to work during US business hours.
