# Shopify-ZohoInventory-Integration
A deluge script in Zoho Inventory that creates the Package and Shipment (marked as delivered), creates the Invoice (marked as sent), and creates the Payment for the Invoice upon Sales Order Creation.

## Core Idea
This code helps with the integration between Shopify and Zoho Inventory. When Shopify orders come into Zoho Inventory, they come through as "Confirmed" Sales Orders. Unfortunately, Zoho doesn't also mark them as "Paid" (even though they're paid for through Shopify). It also doesn't create a Package, Shipment and Invoice record for the order once it's fulfilled from Shopify. This deluge script auto-creates the records above upon Sales Order creation to help with record keeping in Zoho Inventory.

## Configuration
This API call uses the old version of the API authenticaion (V1) which requires users to generate an Auth Token at the link below. The token will then be inserted in the script.

Auth Token Link Generator: https://accounts.zoho.com/apiauthtoken/create?SCOPE=ZohoInventory%2Finventoryapi

## Tutorial

### Define the Required Variables
These variables are required for the function execution. The auth token is inserted here.
```
salesorderID = salesorder.get("salesorder_id");
salesorderdate = salesorder.get("date").toDate();
organizationID = organization.get("organization_id");
authtoken = "21fce7c632ec79807ecee9713c33a594";
//Replace authtoken with token generated from customer's account
```

### Create the Package & Shipment Record
This part of the script creates the Package & Shipment Record based on the Sales Order, and marks the Shipment as Delivered.

```
//Create Package & Shipment Record
mapper = Map();
customerID = salesorder.get("customer_id").toString();
//Uncomment the below two lines if you need the package running number to match the SO number
//package_number = salesorder.get("salesorder_number").replaceFirst("SO","PA");
//mapper.put("package_number",package_number);
mapper.put("customer_id",customerID);
mapper.put("date",salesorderdate);
lineItems = salesorder.get("line_items").toList();
newLineItems = List();
for each  lineItem in lineItems
{
if(lineItem.get("name").isNull())
{
continue;
}
lineItemMap = Map();
solineitemID = lineItem.get("line_item_id");
lineItemMap.put("so_line_item_id",solineitemID);
quantity = lineItem.get("quantity");
lineItemMap.put("quantity",quantity);
newLineItems.add(lineItemMap);
}
mapper.put("line_items",newLineItems);
jsonString = Map();
jsonString.put("JSONString",mapper);
//Create Package
response = invokeurl
[
url :"https://inventory.zoho.com/api/v1/packages?authtoken=" + authtoken + "&salesorder_id=" + salesorderID + "&organization_id=" + organizationID
type :POST
parameters:jsonString
];
info response.toMap().get("message");
pack = response.get("package");
packid = pack.get("package_id");
ship = Map();
ship.put("date",salesorderdate);
ship.put("delivery_method","Manual");
jsons = Map();
jsons.put("JSONString",ship);
//Create Shipment and Mark as Delivered
resp = invokeurl
[
url :"https://inventory.zoho.com/api/v1/shipmentorders?package_ids=" + packid + "&salesorder_id=" + salesorderID + "&authtoken=" + authtoken + "&organization_id=" + organizationID + "&is_delivered=true"
type :POST
parameters:jsons
];
info resp.toMap().get("message");
```

### Create the Invoice Record
This script creates the Invoice and marks it as Sent based on the Sales Order.
```
//Create Invoice Record
bson = Map();
customerID = salesorder.get("customer_id").toString();
salesoderdate = salesorder.get("date").toDate();
//Uncomment the below two lines if you need the invoice running number to match the SO number
//invoice_number = salesorder.get("salesorder_number").replaceFirst("SO","INV");
//bson.put("invoice_number",invoice_number);
bson.put("customer_id",customerID);
bson.put("date",salesorder.get("date"));
lineItems = salesorder.get("line_items").toList();
newLineItems = List();
for each  lineItem in lineItems
{
lineItemMap = Map();
solineitemID = lineItem.get("line_item_id");
lineItemMap.put("salesorder_item_id",solineitemID);
id = lineItem.get("item_id");
lineItemMap.put("item_id",id);
des = lineItem.get("description");
lineItemMap.put("description",des);
wh = lineItem.get("warehouse_id");
lineItemMap.put("warehouse_id",wh);
quantity = lineItem.get("quantity");
lineItemMap.put("quantity",quantity);
r = lineItem.get("rate");
lineItemMap.put("rate",r);
d = lineItem.get("discount");
lineItemMap.put("discount",d);
newLineItems.add(lineItemMap);
}
bson.put("line_items",newLineItems);
response = zoho.inventory.createRecord("Invoices",organizationID,bson);
info response.toMap().get("message");
inv = response.get("invoice");
//Marking Invoice as Sent
invoiceID = inv.get("invoice_id");
invoicedate = inv.get("date").toDate();
response2 = invokeurl
[
url :"https://inventory.zoho.com/api/v1/invoices/" + invoiceID + "/status/sent?authtoken=" + authtoken + "&organization_id=" + organizationID
type :POST
];
info response2.toMap().get("message");
invoiceID = response.get("invoice").get("invoice_id");
invoicedate = response.get("invoice").get("date");
amount = response.get("invoice").get("total");
customer_id = response.get("invoice").get("customer_id");
invoice_number = response.get("invoice").get("invoice_number");
```

### Create the Payment for the Invoice
Since all Sales Orders are technically already paid for and managed in Shopify before coming into Zoho Inventory, it would be a good idea to mark the invoices as "PAID" instead of leaving them "OVERDUE BY X DAYS" if left unattended. This script marks the Invoice as PAID by creating a payment for it.

```
//Create Payment for the Invoice
paramap = Map();
paramap.put("customer_id",customer_id);
paramap.put("payment_mode","Cash"); //Change this to the relevant payment method
paramap.put("amount",amount);
paramap.put("date",invoicedate);
paramap.put("reference_number",invoice_number);
paramap.put("description","Payment has been added to " + invoice_number);
invoicelist = List();
invoicemap = Map();
invoicemap.put("invoice_id",invoiceID);
invoicemap.put("amount_applied",amount);
invoicemap.put("tax_amount_withheld",0);
invoicelist.add(invoicemap);
paramap.put("invoices",invoicelist);
paramap.put("exchange_rate",1);
paramap.put("bank_charges",0);
paramap.put("tax_account_id","");
response3 = invokeurl
[
url :"https://inventory.zoho.com/api/v1/customerpayments?authtoken=" + authtoken + "&organization_id=" + organizationID
type :POST
parameters:paramap + ""
];
info response3;
```
