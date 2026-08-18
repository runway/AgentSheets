## Domain heuristics for table blocks

Use these rules when the request is unclear or the inspection tools return many choices.

### Accounting (QuickBooks, Xero, NetSuite)

- **Variables:** payment amount, expense amount, total amount, invoice amount
- **Row dimensions:** GL account, vendor, department, expense category
- **Column:** system Date, not payment date

### HRIS / Payroll (Gusto, BambooHR, Rippling, ADP)

- **Variables:** salary amount, total compensation, headcount, hours worked
- **Row dimensions:** employee, department, location, role, employment type
- **Column:** system Date, not pay period or effective date

### CRM / Sales (Salesforce, HubSpot, Close)

- **Variables:** contract value, revenue, deal amount, pipeline value, win rate
- **Row dimensions:** customer/account, sales rep, product, region, stage, lead source
- **Column:** system Date, not close date or contract date

### Generic / Unknown

- Use internal `VARIABLE` properties as variables. Use `STRING` and `ENUM` properties as row
  dimensions.
