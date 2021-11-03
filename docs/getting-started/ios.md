---
layout: default
title: iOS
nav_order: 1
parent: Getting Started
has_children: false
---

# Getting Started with iOS
{: .fs-9 .no_toc }

Learn how to set up your iOS app to process payment transactions using QuantumPay.
{: .fs-5 .fw-300 }

---

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## Requirements

<div class="code-example" markdown="1">
- SDKs
    - QuantumPayClient.xcframework
    - QuantumPayMobile.xcframework
    - QuantumPayPeripheral.xcframework
    - QuantumSDK.xcframework
    - ObjectBox **(CocoaPods)**
- Xcode 11+ / iOS 13+ / Swift 5.0+
- Infinite Peripherals' payment device
- Infinite Peripherals developer key for your app bundle ID
- Payment related credentials: username/email, password, service name and tenant key
</div>

***If you are missing any of these items, please contact Infinite Peripherals**

---

## Project Setup
Before we jump into the code we need to make sure your Xcode project is properly configured to use the QuantumPay frameworks. Follow the steps below to get you set up.

### Adding the QuantumPay SDKs

1. Open Xcode and create a new folder to put the frameworks in. If you have a place for frameworks already you can skip this.

2. Drag the QuantumPay frameworks into your folder

<p align="center">
  <img src="https://www.infineadev.com/lucas/qpay/ios-1.png" style='border:1px solid #000000' />
</p>

3. Make sure when prompted you enable "Copy items if needed"

<p align="center">
  <img src="https://www.infineadev.com/lucas/qpay/ios-2.png" style='border:1px solid #000000' />
</p>

4. Go to your project's General Settings and scroll down to the **Frameworks, Libraries and Embedded Content** section. For each framework we just added set the **Embed** field to "Embed & Sign".

<p align="center">
  <img src="https://www.infineadev.com/lucas/qpay/ios-3.png" style='border:1px solid #000000' />
</p>

### Installing ObjectBox

1. In order to use the QuantumPay frameworks you will need to add ObjectBox to your project. Visit [ObjectBox](https://swift.objectbox.io) and follow the instructions to install ObjectBox for your project.

2. If your project is not using CocoaPods yet, it may be more helpful to start here [ObjectBox - Detailed Instructions](https://swift.objectbox.io/install#detailed-instructions)

### Turn off Bitcode support

Go to your project's Build Settings and set **Enable Bitcode** to "No". The easiest way to find this setting is to search using the bar in the top right.

<p align="center">
  <img src="https://www.infineadev.com/lucas/qpay/ios-4.png" style='border:1px solid #000000' />
</p>

### Add MFi protocols

Go to your project's **Info.plist** file and add a new entry for "Supported external accessory protocols" using the following values.

```
com.datecs.pengine
com.datecs.linea.pro.msr
com.datecs.linea.pro.bar
com.datecs.printer.escpos
com.datecs.pinpad
```

<p align="center">
  <img src="https://www.infineadev.com/lucas/qpay/ios-5.png" style='border:1px solid #000000' />
</p>

### Add Privacy entries into Info.plist

Also in your project's **Info.plist** file we need to add the four (4) privacy tags listed below. You can enter any string value you want or copy what we have below.

```
"Privacy - Bluetooth Always Usage Description" 
"Privacy - Bluetooth Peripheral Usage Description"
"Privacy - Location When In Use Usage Description" 
"Privacy - Location Usage Description"
```
<p align="center">
  <img src="https://www.infineadev.com/lucas/qpay/ios-6.png" style='border:1px solid #000000' />
</p>

---

## Processing a Payment
At this point, your Xcode project should be configured and ready to use the QuantumPay libraries. This next section will take you through some initial setup and all the way to processing a payment.


### Initialize the SDKs
First, initialize the SDKs with valid keys provided by Infinite Peripherals. This is a required step before you are able to use other functions from the SDKs.
```swift
// Create tenant
let tenant = Tenant(hostKey: "Host key", tenantKey: "Tenant key")

// Initialize QuantumPay
InfinitePeripherals.initialize(developerKey: "Developer key", tenant: tenant)
```

### Create Payment Device
Now initialize a payment device that matches the hardware you are using. The current supported payment devices: QPC150, QPC250, QPP400, QPP450, QPR250, QPR300. Note that this step is different for payment devices that are connected with Bluetooth LE.
- Initialize QPC150, QPC250 (Lightning connector)

```swift
let paymentDevice = QPC250()
```

- Initialize QPP400, QPP450, QPR250, QPR300 (Bluetooth LE) by supplying its serial number so the `PaymentEngine` can search for it and connect. On first connection the app will prompt you to pair the device. Be sure to press "OK" when the pop-up is shown. To complete the pairing, if using a QPR device, press the small button on top of the device opposite the power button. If using a QPP device, press the green check mark button on the bottom right of the keypad.

```swift
// The device serial number is found on the label on the device.
let paymentDevice = QPR250(serial: "2320900026") 
```

### Create Payment Engine
The payment engine is the main object that you will interact with to send transactions and receive callbacks.

```swift
do {
    try PaymentEngine
        // The PaymentEngineBuilder object.
        .builder()
        // Set the server to send transactions to. Either test or production.
        .server(server: ServerEnvironment.test)
        // The credential used to register hardware devices
        .registrationCredentials(username: "username", password: "password")
        // Set the paymentDevice that we created earlier to PaymentEngine
        // The capabilities are the card input methods (e.g.: Mag stripe, contactless, chip). If you don't want the full array of capabilities provided by the device object, you can put an array of selected input methods only.
        .addPeripheral(peripheral: self.paymentDevice, capabilities: self.paymentDevice.availableCapabilities!, autoConnect: false)
        // The point of sale ID of the device/app. This posID should be unique for each instance of the app. The database also rely on this posID to access a saved database
        .posID(posID: "A unique ID")
        // The timeout when waiting for a card to be presented to the payment device when a transaction has started.
        .transactionTimeout(timeoutInSeconds: 30)
        // StoreAndForwardMode for handling card transactions.
        .storeAndForward(mode: .whenOffline, autoUploadInterval: 60)
        // Build the PaymentEngine. If successful, the PaymentEngine will be passed in the completion block.
        .build(handler: { (engine) in
            /// Save the engine object for operation
            self.pEngine = engine
        })
}
catch {
    print("Error creating payment engine: \(error.localizedDescription)")
}
```

### Setup Handlers
Once the `PaymentEngine` is created, you can use it to set handlers for the operation. The `PaymentEngine` handlers will get called through out the payment process and return back various states of the transaction. You can also set these handlers in the completion block of Step #3.
- `ConnectionStateHandler` will get called when the connection state of the payment device changes between connecting, connected, and disconnected. Please make sure that the connection state is connected before starting a transaction.
```swift
self.pEngine!.setConnectionStateHandler(handler: { (peripheral, connectionState) in
    // Handle connection state
})
```

- `TransactionResultHandler` will get called after the transaction processing. The result status will disclose whether the transaction is declined, or approved.
```swift
self.pEngine!.setTransactionResultHandler(handler: { (transactionResult) in
    // Handle the transaction result
    // TransactionResult.status provides the transaction result
    // TransactionResult.receipt provides the online receipt when TransactionResult.state == .approved
})
```

- `TransactionStateHandler` will get called when the transaction state changes. The TransactionState represents a unique state in the workflow of capturing a transaction.
```swift
self.pEngine!.setTransactionStateHandler(handler: { (peripheral, transaction, transactionState) in
    // Handle the transaction states
})
```

- `PeripheralStateHandler` will get called when peripheral state of the transaction changes. The peripheral state represents the current state of the peripheral as reported by the peripheral device itself.
```swift
self.pEngine!.setPeripheralStateHandler(handler: { (peripheral, state) in
    // Handle peripheral state
})
```

- `PeripheralMessageHandler` will get called when there is new message about the transaction through out the process. The peripheral message tells you when to present the card, if the care read is ok or fails, etc.
```swift
self.pEngine!.setPeripheralMessageHandler(handler: { (peripheral, message) in
    // Handle the peripheral message
})
```

### Connect to Payment Device
Please make sure the device is attached and turned on. We need to connect to the payment device prior to start the transaction using the payment device. The connection state will be returned to `ConnectionStateHandler` that we already setup at Step #4. If you did't set AutoConnect when adding the peripheral in Step #3, you need to call `connect()` before starting a transaction: 
```swift
self.pEngine!.connect()
```

### Create an Invoice
The invoice holds information about a purchase order, and the items in the order.
```swift
let invoice = try self.pEngine!
                    // Build invoice with a reference number. This can be anything.
                    .buildInvoice(reference: invoiceNum)
                    // Set company name
                    .companyName(companyName: "ACME SUPPLIES INC.")
                    // Set purchase order reference. This can be anything to identify an order
                    .purchaseOrderReference(reference: "P01234")
                    // A way to add item to the invoice
                    .addItem(productCode: "SKU1", description: "Discount Voucher for Return Visit", unitPrice: 0)
                    // Another way to add item to the invoice
                    .addItem { (itemBuilder) -> InvoiceItemBuilder in
                        return itemBuilder
                            .productCode("SKU2")
                            .productDescription("In Store Item")
                            .saleCode(SaleCode.S)
                            .unitPrice(1.00)
                            .quantity(1)
                            .unitOfMeasureCode(.Each)
                            .calculateTotals()
                    }
                    // Calculates the totals on the invoice
                    .calculateTotals()
                    // Builds the Invoice instance with the provided values.
                    .build()
```

### Create a Transaction
The transaction holds information about the invoice, the total amount for that transaction, and the type of the transaction (e.g.: sale, auth, refund...)
```swift
let transaction = try self.pEngine!.buildTransaction(invoice: invoice)
                        // The transaction is of type Sale
                        .sale()
                        // The total amount of all the invoices
                        .amount(1.00, currency: .USD)
                        // A unique reference to the transaction, and cannot be reused.
                        .reference("A reference to this transaction")
                        // Date and time of the transaction
                        .dateTime(Date())
                        // The service code generated by Infinite Peripherals
                        .service("service")
                        // Some information about the transaction that you want to add
                        .metaData(["orderNumber" : invoiceNum, "delivered" : "true"])
                        // Build the transaction
                        .build()
```

### Start Transaction
When we have everything ready, we can now start the transaction and take payment. Watch the handlers' messages, and statuses to see the current process.
```swift
try self.pEngine!.startTransaction(transaction: txn) { (transactionResult, transactionResponse) in
        // Handle the transaction result and response
        // transactionResult.status disclose the state of the transaction after processed. See `TransactionResultStatus` for more info.
        // transactionResponse disclose response of server for the submitted transaction. If error is found, the object will contain
        // errors information. See `TransactionResponse` for more info.
        // ....
    }
```

### Transaction Receipt
Once the transaction is completed and approved, you can retrieve the receipt from the `TransactionResultHandler` callback. 
```swift
// The url for customer receipt
transactionResult.receipt?.customerReceiptUrl

// The url for merchant receipt
transactionResult.receipt?.merchantReceiptUrl
```

### Disconnect Payment Device
If needed, you can disconnect the payment device at anytime. The device need to be connected at all time before, and during a transaction process.
```swift
self.pEngine!.disconnect()
```

---

## Scanning a Barcode

Some of our payment devices also support barcode scanning (e.g., QPC150, QPC250). In order to receive the barcode data, you will need to set your class to conform to the protocol `IPCDTDeviceDelegate` and add the class instance as a delegate that will receive the data when scanned. Note that the scan button will not work if it is not properly connected to the application.

```swift
// Add class instance as delegate
self.paymentDevice.device.addDelegate(self)
```

Once a barcode is scanned, it will be sent to this delegate method:

```swift
func barcodeData(_ barcode: String!, type: Int32) {
    // Handle the barcode
}
```