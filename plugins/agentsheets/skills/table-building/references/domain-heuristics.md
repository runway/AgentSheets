## Domain heuristics for table blocks

Use these heuristics to choose the right variables and dimensions when the user's request is ambiguous or when `inspect_variables`/`inspect_dimensions` return many candidates.

### Accounting (QuickBooks, Xero, NetSuite)

- **Variables:** payment amount, expense amount, total amount, invoice amount
- **Row dimensions:** GL account, vendor, department, expense category
- **Column:** system Date (prefer over payment date)

### HRIS / Payroll (Gusto, BambooHR, Rippling, ADP)

- **Variables:** salary amount, total compensation, headcount, hours worked
- **Row dimensions:** employee, department, location, role, employment type
- **Column:** system Date (prefer over pay period / effective date)

### CRM / Sales (Salesforce, HubSpot, Close)

- **Variables:** contract value, revenue, deal amount, pipeline value, win rate
- **Row dimensions:** customer/account, sales rep, product, region, stage, lead source
- **Column:** system Date (prefer over close date / contract date)

### Generic / Unknown

- Internal `VARIABLE`-type properties are variables; `STRING`/`ENUM` properties are row dimensions.
