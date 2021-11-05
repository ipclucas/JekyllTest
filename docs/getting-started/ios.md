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
- Infinite Peripherals payment device
- Infinite Peripherals developer key for your app bundle ID
- Payment related credentials: username/email, password, service name and tenant key
</div>

***If you are missing any of these items, please contact Infinite Peripherals**

---

## Project Setup
Before we jump into the code we need to make sure your Xcode project is properly configured to use the QuantumPay frameworks. Follow the steps below to get you set up.

### Adding the QuantumPay SDKs

1. Open Xcode and create a new folder to put the frameworks in. If you have a place for frameworks already you can skip this.

2. Drag the QuantumPay frameworks into your folder.

<p align="center">
  <img src="https://www.infineadev.com/lucas/qpay/ios-1.png" style='border:1px solid #000000' />
</p>

3. Make sure when prompted you enable "Copy items if needed".

<p align="center">
  <img src="https://www.infineadev.com/lucas/qpay/ios-2.png" style='border:1px solid #000000' />
</p>

4. Go to your project's General tab and scroll down to the **Frameworks, Libraries and Embedded Content** section. For each framework we just added set the **Embed** field to "Embed & Sign".

<p align="center">
  <img src="https://www.infineadev.com/lucas/qpay/ios-3.png" style='border:1px solid #000000' />
</p>

### Installing ObjectBox

1. In order to use the QuantumPay frameworks you will need to add ObjectBox to your project. Visit [ObjectBox](https://swift.objectbox.io) and follow the instructions to install ObjectBox to your project.

2. If your project is not using CocoaPods yet, it may be more helpful to start here [ObjectBox - Detailed Instructions](https://swift.objectbox.io/install#detailed-instructions)

### Turn off Bitcode support
Go to your project's Build Settings tab and set **Enable Bitcode** to "No". The easiest way to find this setting is to use the search field in the top right.

<p align="center">
  <img src="https://www.infineadev.com/lucas/qpay/ios-4.png" style='border:1px solid #000000' />
</p>

### Add MFi protocols to Info.plist
Go to your project's **Info.plist** file and add a new entry for "Supported external accessory protocols" using the following values. Note: in Xcode 13.0+ this has been moved to the "Info" tab in your project's settings.

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

### Add Privacy entries to Info.plist

Also in your project's **Info.plist** file we need to add the four (4) privacy tags listed below. You can enter any string value you want or copy what we have below. Note: in Xcode 13.0+ this has been moved to the "Info" tab in your project's settings.

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
At this point, your Xcode project should be configured and ready to use the QuantumPay libraries. This next section will take you through some initial setup all the way to processing a payment.

### Initialize the SDKs
First, initialize the SDKs with valid keys provided by Infinite Peripherals. This is a required step before you are able to use other functions from the SDKs. If you do not have these keys yet, please contact Infinite Peripherals.

```swift
// Create tenant
let tenant = Tenant(hostKey: "Host key", tenantKey: "Tenant key")

// Initialize QuantumPay
InfinitePeripherals.initialize(developerKey: "Developer key", tenant: tenant)
```

### Create Payment Device
Now initialize a payment device that matches the hardware you are using. The current supported payment devices are: QPC150, QPC250, QPP400, QPP450, QPR250, QPR300. Note that this step is different for payment devices that are connected with Bluetooth LE.
- Initialize QPC150, QPC250 (Lightning connector)

```swift
let paymentDevice = QPC250()
```

- Initialize QPP400, QPP450, QPR250, QPR300 (Bluetooth LE) by supplying its serial number so the `PaymentEngine` can search for and connect to it. On first connection, the app will prompt you to pair the device. Be sure to press "OK" when the pop-up is shown. To complete the pairing, if using a QPR device, press the small button on top of the device opposite the power button. If using a QPP device, press the green check mark button on the bottom right of the keypad.

```swift
// The device serial number is found on the label on the device.
let paymentDevice = QPR250(serial: "2320900026") 
```

### Create Payment Engine
The payment engine is the main object that you will interact with to send transactions and receive callbacks.

```swift
do {
    try PaymentEngine.builder()
        // Set server environment to either test or production
        .server(server: ServerEnvironment.test)
        // Credentials used to register hardware devices
        .registrationCredentials(username: "username", password: "password")
        // Using the device we created earlier, set the peripheral and capabilities
        // Capabilities are the card input methods (e.g.: Mag stripe, contactless, chip). If you don't want the full array of capabilities provided by the device object, you can pass an array of select input methods only.
        .addPeripheral(peripheral: paymentDevice, capabilities: paymentDevice.availableCapabilities!, autoConnect: false)
        // The Point of Sale ID of the device/app. This posID should be unique for each instance of the app. This is used by the database to access saved transactions.
        .posID(posID: "posId")
        // Amount of time after transaction has started to wait for card to be presented
        .transactionTimeout(timeoutInSeconds: 30)
        // StoreAndForwardMode for submitting transactions to server
        .storeAndForward(mode: .whenOffline, autoUploadInterval: 60)
        // If this build function is successful, the PaymentEngine will be passed in the completion block.
        .build(handler: { (engine) in
            // Save the created engine object
            self.pEngine = engine
        })
}
catch {
    print("Error creating payment engine: \(error.localizedDescription)")
}
```

### Setup Handlers
Once the `PaymentEngine` is created, you can use it's handlers to track the operation. The `PaymentEngine` handlers will get called throughout the payment process and will return you the current state of the transaction. You can set these handlers in the completion block of the previous step.

`ConnectionStateHandler` will get called when the connection state of the payment device changes between connecting, connected, and disconnected. It is important to make sure your device is connected before attempting to start a transaction.
```swift
self.pEngine!.setConnectionStateHandler(handler: { (peripheral, connectionState) in
    // Handle connection state
})
```

`TransactionResultHandler` will get called after the transaction is processed. The `TransactionResult` status will disclose whether the transaction was approved or declined.
```swift
self.pEngine!.setTransactionResultHandler(handler: { (transactionResult) in
    // Handle the transaction result
    // transactionResult.status provides the transaction result
    // transactionResult.receipt provides the online receipt when transactionResult.state == .approved
})
```

`TransactionStateHandler` will get called when the transaction state changes. The `TransactionState` represents a unique state in the workflow of capturing a transaction. These include "waitingForCard, waitingForPin, onlineAuthorization, etc.
```swift
self.pEngine!.setTransactionStateHandler(handler: { (peripheral, transaction, transactionState) in
    // Handle transaction state
})
```

`PeripheralStateHandler` will get called when the state of the peripheral changes during the transaction process. The `PeripheralState` represents the current state of the peripheral as reported by the peripheral device itself. These include "idle", "ready", "contactCardInserted" etc.
```swift
self.pEngine!.setPeripheralStateHandler(handler: { (peripheral, state) in
    // Handle peripheral state
})
```

`PeripheralMessageHandler` will get called when there is new message about the transaction throughout the process. The peripheral message tells you when to present the card, if the card read is successful or failed, etc. This usually indicates something that should be displayed in the user interface.
```swift
self.pEngine!.setPeripheralMessageHandler(handler: { (peripheral, message) in
    // Handle peripheral message
})
```

### Connect to Payment Device
Now that your payment engine is configured and your handlers are set up, lets connect to the payment device. Please make sure the device is attached and turned on. We need to connect to the payment device prior to starting a transaction. The connection state will be returned to the `ConnectionStateHandler` that we set up previously. If you didn't set autoConnect when creating the payment engine, you will need to call `connect()` before starting a transaction.

```swift
self.pEngine!.connect()
```

### Create an Invoice
Time to create an invoice. This invoice object holds information about a purchase order and the items in the order.

```swift
let invoice = try self.pEngine!
                    // Build invoice with a reference number. This can be anything.
                    .buildInvoice(reference: invoiceNum)
                    // Set company name
                    .companyName(companyName: "ACME SUPPLIES INC.")
                    // Set purchase order reference. This can be anything to identify the order.
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
                    // Calculate totals on the invoice
                    .calculateTotals()
                    // Builds invoice instance with the provided values
                    .build()
```

### Create a Transaction
The transaction object holds information about the invoice, the total amount for the transaction and the type of the transaction (e.g.: sale, auth, refund, etc.)

```swift
let transaction = try self.pEngine!.buildTransaction(invoice: invoice)
                        // The transaction is of type Sale
                        .sale()
                        // The total amount of for all invoices
                        .amount(1.00, currency: .USD)
                        // A unique reference to the transaction. This cannot be reused.
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
Now that everything is ready we can start the transaction and take payment. Watch the handler messages and status updates to track the transaction throughout the process.

```swift
try self.pEngine!.startTransaction(transaction: txn) { (transactionResult, transactionResponse) in
    // Handle the transaction result and response
    // transactionResult.status discloses the state of the transaction after being processed. See `TransactionResultStatus` for more info.
    // transactionResponse discloses the server's response for the submitted transaction. If an error occurs, the object will contain the
    // error's information. See `TransactionResponse` for more info.
    // ....
}
```

### Transaction Receipt
Once the transaction is completed and approved, the receipt is sent to the `TransactionResultHandler` callback.

```swift
// The url for customer receipt
transactionResult.receipt?.customerReceiptUrl

// The url for merchant receipt
transactionResult.receipt?.merchantReceiptUrl
```

### Disconnect Payment Device
Now that the transaction is complete you are free to disconnect the payment device if you wish. Please note that this should not be called before or during the transaction process.

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