# Loan-transfer comparison captures

Captured on 2026-10-24 from fork source
`854161212bf25666524ec62d20cd70939b3dc965`, based on upstream
`ab457b47370ae69964a835acea96083edd4b13d9` (`v0.7.5-alpha.9`).

These are browser captures of Sure's server-rendered dashboard category list
with synthetic records. Both use the same source revision; the before state
reproduces stale data that the upstream baseline can leave behind. The after
state applies the fork's real repair migration. They do not demonstrate the
current upstream head or exercise the complete interactive dashboard.

The category list is extracted without changing its HTML from
[`pages/dashboard/_outflows_donut.html.erb`](../../../app/views/pages/dashboard/_outflows_donut.html.erb).
The surrounding title, date, total, and source label are capture annotations.
The total comes from the same report query. The JavaScript donut chart is outside
the capture. Tailwind CSS comes from the repository's built `tailwind.css`.

## Reproduction

Use a fresh, isolated Rails **test** database with the repository schema loaded.
Do not use production data or provider connections. The
[migration regression](../../../test/migrations/repair_loan_payment_on_transfer_inflows_migration_test.rb)
shows the import and pairing setup used here; unlike that test, this fixture
has no unmatched repayment.

1. Freeze the clock at `2026-10-24 12:00:00 UTC`. Create a new USD family, a
   `Depository` account named `Demo checking` with balance 5,000, and a `Loan`
   account named `Demo loan` with balance 300,000.
2. Use `Account::ProviderImportAdapter#import_transaction` to import a checking
   outflow of `500` and loan inflow of `-500`, both dated `2026-10-24`, currency
   `USD`, source `simplefin`, and name `Synthetic repayment`. Give them distinct
   external IDs. Call `family.auto_match_transfers!`, reload the transaction
   associations, and verify they share a transfer.
3. Categorize only the bank-side transaction as `Loan Payments`, color
   `#6366F1`, icon `landmark`. Set the loan inflow's kind to `loan_payment` with
   `update_columns` to reproduce the stale state. Do not change either amount
   or the transfer link.
4. Build and render each report through the existing application code:

   ```ruby
   period = Period.current_month
   totals = IncomeStatement.new(family, accounts: family.accounts)
     .net_category_totals(period: period)
   data = PagesController.new.send(:build_outflows_donut_data, totals)
   fragment = ApplicationController.render(
     partial: "pages/dashboard/outflows_donut",
     locals: { outflows_data: data, period: period }
   )
   list = Nokogiri::HTML.fragment(fragment)
     .css("#outflows-donut-section > div > div > div").last.to_html
   ```

   Place `list` in a static HTML wrapper with the built Tailwind CSS inlined.
   Use the classes shown in the captures: body `bg-container p-6 text-primary`,
   main `space-y-4`, title `text-xl font-medium`, caption
   `text-sm text-secondary`, total `text-sm font-medium`, and source label
   `text-xs text-secondary`. The total annotation formats
   `Money.new(data[:total], "USD")`.
5. Capture the before state. Require
   `db/migrate/20260926171500_repair_loan_payment_on_transfer_inflows.rb`, run
   `RepairLoanPaymentOnTransferInflows.new.up` on this disposable database,
   then rebuild the report with a **new** `IncomeStatement` and render the after
   state. Verify total outflows are 1,000 before and 500 after, and that only
   the before state has an Uncategorized row.
6. Open each static HTML file in a fresh `agent-browser` session with file
   access enabled. Use the same light theme, 760 × 420 viewport, and PNG format.
   Inspect the rendered text before capture. These images used
   `agent-browser` 0.33.2 and Chrome for Testing 151.0.7922.77 with no browser
   console errors.

This process needs no Rails server or live provider requests. It illustrates
the accounting result; the regression tests remain the evidence for subsequent
imports and warm-cache behavior.
