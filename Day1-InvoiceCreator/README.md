# Automated Invoice Generation & Database Update Workflow

This README provides both **users** and **creators** with clear guidance on how to use and build the automated invoice system shown in the workflow image.

---

## Table of Contents
1. [Overview](#overview)
2. [For End Users](#for-end-users)
3. [For Creators / Developers](#for-creators--developers)
4. [Workflow Steps Explained](#workflow-steps-explained)
5. [Google Sheet & Template Links](#google-sheet--template-links)
6. [Code Snippets](#code-snippets)
7. [Future Improvements](#future-improvements)

---

## Overview
This system automates **invoice creation, placeholder replacement, database logging, and email distribution** using **n8n** and **Google Sheets**.

- **Inputs:** Customer & product data (via form/webhook)
- **Outputs:**
  - Invoices generated in Google Sheets
  - Placeholders replaced with dynamic data
  - Database (Google Sheet) updated with transaction
  - Copies saved to Google Drive and optionally emailed

---

## For End Users

### How to Use:
1. Fill in the customer + product form (provided by your organization).
2. Submit — this triggers the automation.
3. Within seconds:
   - An invoice is generated (split into multiple files if >10 products)
   - You receive the invoice via email or shared Google Drive link.
4. All data is securely logged into the master invoice sheet.

---

## For Creators / Developers

### Prerequisites
- [n8n](https://n8n.io) account/self-hosted instance
- Google Sheets + Google Drive API enabled
- Webhook endpoint in n8n for form data

### Required Files & Links
- **Invoice Database:** [Google Sheet Link](https://docs.google.com/spreadsheets/d/1I3Kq-T9-Oc9SPtClhazH6ypjl_mUj7i3v0rcdZNroG8/edit?usp=sharing)
- **Invoice Template:** [Template Sheet Link](https://docs.google.com/spreadsheets/d/1pk7N3ea5970RouQgEaMNndpofQ_ZY7FBSk9C1bH7PZk/edit?usp=sharing)
- **Workflow Diagram:** ![Workflow Screenshot](./a6ff501c-8519-4a95-ba54-c91495a49190.png)
-  **Custumer Detail:** ![Google Sheet Link](https://docs.google.com/spreadsheets/d/1q-lMpQMHDDjF5bhrV2slt7x84J0Leu8W73aEyjO-BjM/edit?usp=sharing)
- 

### Setup Steps
1. Import provided n8n workflow JSON (coming soon).
2. Connect Google Sheets and Google Drive credentials in n8n.
3. Update Sheet IDs and Template IDs in workflow nodes.
4. Deploy webhook endpoint and connect your HTML form.
5. Test end-to-end automation with sample data.

---

## Workflow Steps Explained

### 1. **Get Last Invoice Number**
- Reads existing invoice log to generate next sequential invoice number.

### 2. **Chunk Products into Groups of 10**
- Splits product array into manageable parts (Google Sheet template supports 10 lines per invoice).

### 3. **Loop Over Chunks**
- For each chunk:
  - Assign new invoice number
  - Copy invoice template file
  - Replace placeholders with customer and product data

### 4. **Batch Update Template Placeholders**
- Uses Google Sheets API `batchUpdate` to populate template cells.

### 5. **Log Data to Master Sheet**
- Appends structured data for reporting and tracking.

### 6. **Send Invoice via Email (Optional)**
- Downloads populated invoice and sends as PDF attachment.

---

## Google Sheet & Template Links
- **Invoice Database:** [Open here](https://docs.google.com/spreadsheets/d/1I3Kq-T9-Oc9SPtClhazH6ypjl_mUj7i3v0rcdZNroG8/edit?usp=sharing)
- **Invoice Template:** [Open here](https://docs.google.com/spreadsheets/d/1pk7N3ea5970RouQgEaMNndpofQ_ZY7FBSk9C1bH7PZk/edit?usp=sharing)

---

## Code Snippets

### Generate base number
```javascript
const rows = items;
let lastInvoiceNum = "INV-000";

if (rows.length > 0) {
  const lastRow = rows[rows.length - 1];
  lastInvoiceNum = lastRow.json["Invoice Number"] || "INV-000";
}

const lastNum = parseInt(lastInvoiceNum.replace("INV-", "")) || 0;

return [
  {
    json: {
      startInvoiceIndex: lastNum + 1
    }
  }
];
```

### Chunk Products
```javascript
const data = $('Webhook1').first().json;
const products = data.body.products || [];
const chunkSize = 10;

function chunkArray(arr, size) {
  return Array.from({ length: Math.ceil(arr.length / size) }, (_, i) =>
    arr.slice(i * size, i * size + size)
  );
}

const chunks = chunkArray(products, chunkSize);

return chunks.map((chunk, i) => ({
  json: {
    customer: data.body.customer,
    grandTotal: data.body.grandTotal,
    products: chunk,
    invoiceChunkIndex: i
  }
}));
```

### Add Invoice Number1
```javascript
const baseIndex = $items("Generate Base Invoice Number1")[0].json.startInvoiceIndex;
const thisChunkIndex = $json.invoiceChunkIndex;

if (thisChunkIndex === undefined || isNaN(thisChunkIndex)) {
  // Prevent invalid chunk
  return [];
}

const finalInvoiceNo = `INV-${String(baseIndex + thisChunkIndex).padStart(3, "0")}`;

return [{
  json: {
    ...$json,
    invoiceNumber: finalInvoiceNo
  }
}];
```

### Return Count
```javascript
const count = $('Chunk Products1').first().json.products.length;
const result = count.toString() + ' products';

return [
  {
    json: {
      result
    }
  }
];
```

### Return Essential Data
```javascript
const products = $input.first().json.products;
// Calculate totals
const totals = products.reduce((acc, product) => {
  acc.totalPrice += product.total || 0;
  acc.totalQuantity += product.quantity || 0;
  return acc;
}, { totalPrice: 0, totalQuantity: 0 });

// Add current date (formatted as YYYY-MM-DD)
const today = new Date().toISOString().split('T')[0];

return [{
  json: {
    totalPrice: totals.totalPrice,
    totalQuantity: totals.totalQuantity,
    date: today
  }
}];
```

### Create cell update
```javascript
const customer =$('Add Invoice Number1').first().json.customer;
const products = $('Add Invoice Number1').first().json.products;
const invoiceNumber =$('Add Invoice Number1').first().json.invoiceNumber;
const issueDate = new Date().toISOString().split("T")[0];
const dueDate = new Date(Date.now() + 7 * 86400000).toISOString().split("T")[0];
const grandTotal =$('Webhook1').first().json.body.grandTotal;

const customerCells = [
  { range: 'Getinvoice.co!C8', values: [[customer.name || '']] },
  { range: 'Getinvoice.co!C9', values: [[customer.email || '']] },
  { range: 'Getinvoice.co!C10', values: [[customer.address || '']] },
  { range: 'Getinvoice.co!F3', values: [[issueDate]] },
  { range: 'Getinvoice.co!F6', values: [[dueDate]] },
  { range: 'Getinvoice.co!F4', values: [[invoiceNumber]] },
  { range: 'Getinvoice.co!F25', values: [[grandTotal]] }
];

const productStartRow = 14;
const productCells = [];
products.forEach((product, i) => {
  const row = productStartRow + i;
  productCells.push(
    { range: `Getinvoice.co!A${row}`, values: [[`${i + 1}`]] },
    { range: `Getinvoice.co!B${row}`, values: [[product.name]] },
    { range: `Getinvoice.co!D${row}`, values: [[product.quantity]] },
    { range: `Getinvoice.co!E${row}`, values: [[product.price]] },
    { range: `Getinvoice.co!F${row}`, values: [[product.total]] }
  );
});

return [{
  json: {
    valueInputOption: 'USER_ENTERED',
    data: [...customerCells, ...productCells],
    sheetId:  $('Copy file1').first().json.id // Pass from previous node if needed
  }
}];
```

### Batch Update
```javascript
{
  "Content-Type": "application/json"
}
```

---

## Future Improvements
- Automatic payment status updates via webhook from payment gateways.
- Adding digital signature & QR codes to invoices.
- Integration with CRM systems (HubSpot, Zoho, etc.).
- Multi-language invoice templates.

---

> **Note:** Full HTML form and n8n JSON export will be added in subsequent commits.
