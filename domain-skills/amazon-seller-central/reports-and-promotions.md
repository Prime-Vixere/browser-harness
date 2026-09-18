# Amazon Seller Central: reports, promotions, Subscribe & Save, Ads reports

Private JSON endpoints behind the Seller Central UI. All are same-origin `fetch` calls from a
`sellercentral.amazon.com` tab (cookies carry auth). Much faster than scraping tables.

## Account switching
- Switcher page: `/account-switcher/default/merchantMarketplace`. Accounts are `button.full-page-account-switcher-account-details`;
  click the account name, then its marketplace button ("United States"), then `kat-button[data-test=confirm-selection]`
  (click the button inside its shadowRoot).
- The selection is cookie-wide: it changes the account for every Seller Central tab in the profile.

## Promotions (Promotion Central)
- List: `GET /promotion-central/api/v1/promotions?pageSize=50&pageNumber=1&sortColumn=promotionStatus%2Cschedule&sortOrder=desc%2Cdesc&store=ATVPDKIKX0DER[&status=RUNNING]`
  - **Requires header `region: NA`**, or it returns `total: 0` with no error.
  - Each item: `published` (namespace Coupon / BuyXGetY / PriceDiscount / Deal, schedule, benefit, `productSelectionId`),
    `performance` (sales, units, budget used, clips), `uiStatus` (RUNNING, CANCELED, ENDED, SUPPRESSED, BUDGET_EXHAUSTED).
- Dashboard rows render inside shadow DOM; walk `shadowRoot`s to find `kat-link` hrefs.
- Coupon detail: `GET /coupons/api/couponPromotion?promotionId=<id>&clientId=LegacyCouponsUI`
  and ASINs: `GET /coupons/api/couponPromotion/products?productSelectionId=<id>&pageSize=999&clientId=LegacyCouponsUI`
  (includes per-ASIN `isSuppressed` + `suppressionDetails` with the required price).
- Buy X Get Y / volume discount detail is server-rendered at `/promotions/view?promotionId=<id>&region=NA`
  (tiers, tracking ID). Change history (editor + timestamps) is embedded JSON in the
  `[data-a-modal*="Change History"]` attribute (`inlineContent`), no click needed.
- In the All Orders report `promotion-ids`, coupons appear as `PLM-<couponPromotionId>`, and BXGY promos as their
  **Tracking ID** (e.g. `Percentage Off 2026/09/15 14-1-31-265`), not the promotion ID.

## Business Reports
- GraphQL: `POST /business-reports/api`, operation `reportDataQuery`,
  `variables.input = {legacyReportId, startDate, endDate, asins: []}`.
  - `102:DetailSalesTrafficByChildItem` gives child-ASIN rows for the whole date range (aggregated). For a daily
    series, loop one day at a time (about 1s per call).
  - `102:SalesTrafficTimeSeries` returns one row per day (epoch-seconds dates), incl. Units per Order Item.
- Percent fields come back as strings like `"19.08"` (percent, not fraction).

## Report Central (All Orders, FBA Promotions, etc.)
- Configs: `GET /reportcentral/api/v1/getReportConfigurations`. All Orders = FRP 2400 (order date), 30-day max range.
  FBA Promotions = FRP 2601.
- Request: `POST /reportcentral/api/v1/submitDownloadReport?reportFileFormat=TSV&&reportStartDate=YYYY%2FMM%2FDD&reportEndDate=YYYY%2FMM%2FDD&xdaysBeforeUntilToday=-1&startDateTimeOffset=0&endDateTimeOffset=0&specialDateOptions=&reportFRPId=2400&language=&disableTimezone=true`
  - Params go in the **query string** on a POST. Header `anti-csrftoken-a2z` from `meta[name='anti-csrftoken-a2z']` is required (403 "Unsafe cross site request" without it).
  - Without `xdaysBeforeUntilToday=-1` the dates are silently ignored and you get today only.
- Status: `GET /reportcentral/api/v1/getDownloadHistoryRecords?reportId=2400&reportId=2402&isCountrySpecific=false` (InQueue, InProgress, Done). Occasionally returns an empty body; retry.
- Download: `GET /reportcentral/api/v1/downloadFile?referenceId=<id>&fileFormat=TSV` redirects to a same-origin
  `/listing/downloadfile?...` URL, so `fetch(...).then(r=>r.text())` works directly.

## Subscribe & Save
- Pages: `/sns/dashboard` (QuickSight embed), `/sns/performance` (downloadable offer-level report).
  The dashboard page also contains destructive buttons (Remove all products, Opt out). Do not click around blindly.
- Performance report is a Data Kiosk query. Submit: `POST /sns/sp/api/v1/reports` with
  `{reportType:"OFFER_METRICS_REPORT_V2", requestPreferences:{gqlQuery, merchant:<sellerId>, marketplace:<marketplaceId>}}`.
  Copy `gqlQuery` from `POST /sns/sp/api/v1/reports/history` (`{reportType:"OFFER_METRICS_REPORT_V2",pageSize:50,processingStatuses:[]}`),
  change the dates, and set `marketplaceIds:["{MARKETPLACE_IDS}"]` (literal placeholder; a real ID is rejected).
  - Only one query runs at a time; a second concurrent submit returns a generic 400. Submit serially.
  - `aggregationFrequency:WEEK` is rejected; request separate 7-day windows instead.
  - Poll via the history endpoint (`processingStatus` DONE + `dataDocumentId`); `GET /sns/sp/api/v1/reports?referenceId=` errors.
  - Download URL: `GET /sns/sp/api/v1/report/download?documentId=<dataDocumentId>&reportType=OFFER_METRICS_REPORT_V2` returns an S3 URL.
    The file is gzipped JSONL with a UTF-8 BOM (older UI-generated ones are CSV).

## Amazon Ads console reports (advertising.amazon.com)
- Account entity IDs appear in links on `/am/overview` (filter box `#amzn-adam:accounts-overview:quickFilter:input`; type with `Input.insertText`).
- Report list: `/reports?entityId=<ENTITY...>`. Create form: `/reports/new?entityId=...`.
  Hand-built payloads to `PUT /reports/api/subscriptions` fail ("Invalid createSubscription request"); the UI actually calls
  `PUT /reports/api/subscriptions/custom?entityId=...` with a large column list. Easiest path is driving the form:
  category/type are `button[data-takt-id='storm-ui-dropdown-trigger-button']` + `[role=option]`, time unit is a `label` "Daily",
  calendar day buttons have `aria-label` like `Wednesday July 1 2026`, plus `go to previous month` / `go to next month`.
  DOM `.click()` works in a background tab; coordinate clicks on the popover did not.
- Download links on `/reports/history/<subscriptionId>?entityId=...` (`a[href*=download-report]`) redirect cross-origin to S3,
  so `fetch` fails on CORS. Use `Browser.setDownloadBehavior` with a `downloadPath` and click a created `<a>`.
