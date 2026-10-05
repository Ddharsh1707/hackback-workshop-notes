# accountill notes

Solo lab. Scope: server/ only; client/build/ and node_modules/ ignored.

## Claims

- The auth middleware is defined but never used: no route file or index.js imports it, so every API route can be called without logging in.
  Evidence: server/middleware/auth.js:7 (defined); server/routes/invoices.js:1-2, server/routes/clients.js:1-2, server/routes/profile.js:1-2, server/routes/userRoutes.js:1-2 (imports only express and controllers) [Confirmed]

- The server has 24 route handlers: 20 across 4 router files (invoices 6, clients 5, profile 5, users 4) plus 4 defined directly in index.js (/send-pdf, /create-pdf, /fetch-pdf, /). profile.js:7 is commented out and not counted.
  Evidence: server/index.js:32-35, server/index.js:53, server/index.js:87, server/index.js:97, server/index.js:102 [Confirmed]

- Invoices and clients link to their owner by a plain string array, not a database reference, and the invoice's client details are copied into the invoice instead of linked.
  Evidence: server/models/InvoiceModel.js:15, server/models/InvoiceModel.js:17, server/models/ClientModel.js:9 [Confirmed]

- The only unique constraints in the data model are on User.email and Profile.email; nothing stops duplicate invoice numbers.
  Evidence: server/models/userModel.js:5, server/models/ProfileModel.js:5, server/models/InvoiceModel.js:13 [Confirmed]

- Every generated PDF is written to the same file name, invoice.pdf, so two users generating PDFs at the same time can overwrite each other's invoice.
  Evidence: server/index.js:57, server/index.js:88, server/index.js:98 [Confirmed]

## Corrections

- The agent first said the server needs 6 environment variables. Re-reading the code shows 7: DB_URL, PORT, SECRET, SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS. Corrected claim:
  - The server reads 7 environment variables (PORT falls back to 5000).
    Evidence: server/index.js:39-43, server/index.js:106-107, server/middleware/auth.js:5 [Confirmed]
