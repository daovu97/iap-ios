# In-App Purchase 

## I/ Overview
Offer extra content and features — including digital goods, subscriptions, and premium content — directly within your app through in‑app purchases on all Apple platforms. You can even promote and offer in-app purchases directly on the App Store.

<p align="center" width="100%">
    <img width="60%" src="https://developer.apple.com/in-app-purchase/images/en-lockup-hero-large_2x.png">
</p>


You need to create an App ID. This will link together your app to your in-app purchaseable products. Login to the ```Apple Developer Center```, then select ```Certificates, IDs & Profiles```.
## II/ Creating In-App Purchase Products
There are four types of in-app purchases and you can offer multiple types within your app.

### 1. Consumable
    Provide different types of consumables, such as lives or gems used to further progress in a game, boosts in a dating app to increase profile visibility, or digital tips for creators within a social media app. Consumable in‑app purchases are depleted as they’re used and can be purchased again. They’re frequently offered in apps and games that use the freemium business model.
### 2. Non-consumable
    Provide non-consumable, premium features that are purchased once and don’t expire. Examples include additional filters in a photo app, extra brushes in an illustration app, or cosmetic items in a game. Non-consumable in-app purchases can offer Family Sharing.
### 3. Auto‑renewable subscriptions
    Provide ongoing access to content, services, or premium features in your app. People are charged on a recurring basis until they decide to cancel. Common use cases include access to media or libraries of content (such as video, music, or articles), software as a service (such as cloud storage, productivity, or graphics and design), education, and more. Auto-renewable subscriptions can offer Family Sharing.
### 4. Non-renewing subscriptions
    Provide access to services or content for a limited duration, such as a season pass to in-game content. This type of subscription doesn’t renew automatically, so people need to purchase a new subscription once it concludes if they want to retain access.

<p align="center" width="100%">
    <img width="60%" src="https://koenig-media.raywenderlich.com/uploads/2018/07/S4-In-app.png">
</p>

<p align="center" width="100%">
    <img width="60%" src="https://koenig-media.raywenderlich.com/uploads/2016/03/IAP-Type.png">
</p>


In this article, I will talk about ```Auto‑renewable subscriptions```.
## III/ Implement ```Auto‑renewable subscriptions``` to iOS project.
### 1. Project Configuration
Select ```Project``` under ```Targets``` in Xcode. Select the ```General``` tab, switch your Team to your correct team, and enter the bundle ID you used earlier.
<p align="center" width="100%">
    <img width="100%" src="https://koenig-media.raywenderlich.com/uploads/2016/03/RazeFaces_ProjectConfiguration.png">
</p>
Next select the Capabilities tab. Scroll down to In-App Purchase and toggle the switch to ON.

    Note: If IAP does not show up in the list, make sure that, in the Accounts section of Xcode preferences, you are logged in with the Apple ID you used to create the app ID.
<p align="center" width="100%">
    <img width="100%" src="https://koenig-media.raywenderlich.com/uploads/2016/03/RazeFaces_Capabilities.png">
</p>

### 2. Implement code
#### Using StoreKit 
```swift
import StoreKit
```

#### Request all product for buy
```swift
private let productIdentifiers: Set<ProductIdentifier>
private var purchasedProductIdentifiers: Set<ProductIdentifier> = []
private var productsRequest: SKProductsRequest?
private var productsRequestCompletionHandler: ProductsRequestCompletionHandler?
public func productsRequest(_ request: SKProductsRequest, didReceive response: SKProductsResponse) {
    print("Loaded list of products...")
    let products = response.products
    productsRequestCompletionHandler?(true, products)
    clearRequestAndHandler()

    for p in products {
      print("Found product: \(p.productIdentifier) \(p.localizedTitle) \(p.price.floatValue)")
    }
  }
  
  public func request(_ request: SKRequest, didFailWithError error: Error) {
    print("Failed to load list of products.")
    print("Error: \(error.localizedDescription)")
    productsRequestCompletionHandler?(false, nil)
    clearRequestAndHandler()
  }

  private func clearRequestAndHandler() {
    productsRequest = nil
    productsRequestCompletionHandler = nil
  }
}
```
This extension is used to get a list of products, their titles, descriptions and prices from Apple’s servers by implementing the two methods required by the SKProductsRequestDelegate protocol.

#### Making Purchases 
```swift
public func buyProduct(_ product: SKProduct) {
  print("Buying \(product.productIdentifier)...")
  let payment = SKPayment(product: product)
  SKPaymentQueue.default().add(payment)
}
```

Response delegate

```swift
// MARK: - SKPaymentTransactionObserver
 
extension IAPHelper: SKPaymentTransactionObserver {
 
  public func paymentQueue(_ queue: SKPaymentQueue, 
                           updatedTransactions transactions: [SKPaymentTransaction]) {
    for transaction in transactions {
      switch transaction.transactionState {
      case .purchased:
        complete(transaction: transaction)
        break
      case .failed:
        fail(transaction: transaction)
        break
      case .restored:
        restore(transaction: transaction)
        break
      case .deferred:
        break
      case .purchasing:
        break
      }
    }
  }
 
  private func complete(transaction: SKPaymentTransaction) {
    print("complete...")
    deliverPurchaseNotificationFor(identifier: transaction.payment.productIdentifier)
    SKPaymentQueue.default().finishTransaction(transaction)
  }
 
  private func restore(transaction: SKPaymentTransaction) {
    guard let productIdentifier = transaction.original?.payment.productIdentifier else { return }
 
    print("restore... \(productIdentifier)")
    deliverPurchaseNotificationFor(identifier: productIdentifier)
    SKPaymentQueue.default().finishTransaction(transaction)
  }
 
  private func fail(transaction: SKPaymentTransaction) {
    print("fail...")
    if let transactionError = transaction.error as NSError?,
      let localizedDescription = transaction.error?.localizedDescription,
        transactionError.code != SKError.paymentCancelled.rawValue {
        print("Transaction Error: \(localizedDescription)")
      }

    SKPaymentQueue.default().finishTransaction(transaction)
  }
 
  private func deliverPurchaseNotificationFor(identifier: String?) {
    guard let identifier = identifier else { return }
 
    purchasedProductIdentifiers.insert(identifier)
    UserDefaults.standard.set(true, forKey: identifier)
    NotificationCenter.default.post(name: .IAPHelperPurchaseNotification, object: identifier)
  }
}
```

Verify With server
```Swift
func verifyReceipt(completion: ((Bool) -> Void)? = nil) {
        guard !isVerifying else { return }
        guard let _ = UserSessionManager.shared.getUserId(),
              let receiptData = getLocalReceipt(),
              let udid = UIDevice.current.identifierForVendor?.uuidString else { completion?(false); return }
        
        self.isVerifying = true
        let request = VerifyRecepitRequest(receipt_data: receiptData, udid: udid)
        APIService.verifyReceipt(request)
            .subscribe(onNext: {[weak self] in completion?($0.isSuccess) ;  self?.isVerifying = false },
                       onError: {[weak self] err in
                print("verifyReceipt \(err.localizedDescription)")
                completion?(false)
                self?.isVerifying = false
                DispatchQueue.main.async {
                    let isToLogin = err.codeData == 403
                    self?.showAlert(message: isToLogin ? R.string.localizable.login_again_title() : err.messageData,
                                    buttonTitle: isToLogin ? R.string.localizable.to_login_title() : "OK",
                                    completion: {
                        if isToLogin {
                            UserSessionManager.shared.logoutHandler()
                        }
                    })
                    self?.removeAll()
                }
            }).disposed(by: self.bag)
    }
```
Request Local Reciept
```swift
private func getLocalReceipt() -> String? {
        if let appStoreReceiptURL = Bundle.main.appStoreReceiptURL,
            FileManager.default.fileExists(atPath: appStoreReceiptURL.path) {

            do {
                let receiptData = try Data(contentsOf: appStoreReceiptURL, options: .alwaysMapped)
                return receiptData.base64EncodedString()
            } catch { print("Couldn't read receipt data with error: " + error.localizedDescription)
                return nil
            }
        } else {
            return nil
        }
    }
```

### 3. Making a Sandbox Purchase (Testing)
Build and run the app — but to test out purchases, you’ll have to run it on a device. The sandbox tester created earlier can be used to perform the purchase without getting charged.
Go to your iPhone and make sure you’re logged out of your normal App Store account. To do this, go to the ```Settings``` app and tap ```iTunes & App Store```.

<p align="center" width="100%">
    <img width="40%" src="https://koenig-media.raywenderlich.com/uploads/2016/03/RazeFaces-Settings.jpg">
</p>

<p align="center" width="100%">
    <img width="50%" src="https://koenig-media.raywenderlich.com/uploads/2018/05/RazeFaces_SandboxCombined2.png">
</p>

## IV/ Common cases
### 1. Purchase again with other account
Subscription package will follow google account and productId so when buying again will not pay anymore -> return status as purchased (try to check again)
-> check with the server with ```Purchase.purchaseToken```, the server checks that already in the db matches the userid, it will notify the purchase failed
     
### 2. I'm making a purchase but I'm losing my life while I'm checking out store (the payment won't be completed, so I'll receive the error event ```onPurchasesUpdated```)

### 3. I'm buying, while verifying the server is down
In this case, the purchase status will be saved if the purchase on google is successful, when restarting the app next time will check that flag and query history and then verify with the server.