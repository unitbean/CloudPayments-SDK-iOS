## CloudPayments SDK for iOS 

CloudPayments SDK позволяет интегрировать прием платежей в мобильные приложение для платформы iOS.

### Требования
Для работы CloudPayments SDK необходим iOS версии 11.0 и выше.

### Подключение
Для подключения SDK мы рекомендуем использовать CocoaPods. Добавьте в файл Podfile зависимости:

```
pod 'Cloudpayments', :git =>  "https://github.com/cloudpayments/CloudPayments-SDK-iOS", :branch => "master"
pod 'CloudpaymentsNetworking', :git =>  "https://github.com/cloudpayments/CloudPayments-SDK-iOS", :branch => "master"

```

**Для новых чипов М1 и выше.**
Для корректной установки CocoaPods для новых чипов, необходимо использовать команды ниже:

- `sudo arch -x86_64 gem install ffi` - Установка расширения динамических библиотек Ruby.

- `arch -x86_64 pod install` - Установка зависимостей.


### Структура проекта:

* **demo** - Пример реализации приложения с использованием SDK
* **sdk** - Исходный код SDK


### Возможности CloudPayments SDK:

Вы можете использовать SDK одним из трех способов: 
* использовать стандартную платежную форму Cloudpayments
* реализовать свою платежную форму с использованием функций CloudpaymentsApi без вашего сервера
* реализовать свою платежную форму, сформировать криптограмму и отправить ее на свой сервер

## Инициализация CloudPaymentsSDK

В `AppDelegate.swift` вашего проекта добавьте нотификацию `CloudtipsSDK` о событиях жизенного цикла приложения:

```swift
func application(_ application: UIApplication, continue userActivity: NSUserActivity, restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void) -> Bool {
    CloudPaymentsSDK.instance.applicationDidReceiveUserActivity(userActivity)
    return true
}

func application(_ app: UIApplication, open url: URL, options: [UIApplication.OpenURLOptionsKey : Any] = [:]) -> Bool {
    CloudPaymentsSDK.instance.applicationDidReceiveOpen(url, sourceApplication: options[.sourceApplication] as? String)
    return true
}
    
func applicationWillEnterForeground(_ application: UIApplication) {
    CloudPaymentsSDK.instance.applicationWillEnterForeground()
}
    
func applicationDidBecomeActive(_ application: UIApplication) {
    CloudPaymentsSDK.instance.applicationDidBecomeActive()
}
```

### Использование встроенной платежной формы от CloudPayments:

1. Cоздайте объект PaymentDataPayer и проинициализируйте его, затем создайте объект PaymentData передайте в него объект PaymentDataPayer, сумму платежа, валюту и дополнительные данные. Если хотите иметь возможность оплаты с помощью Apple Pay, передайте также Apple pay merchant id.

```
// Доп. поле, куда передается информация о плательщике. Используйте следующие параметры: FirstName, LastName, MiddleName, Birth, Street, Address, City, Country, Phone, Postcode
let payer = PaymentDataPayer(firstName: "Test", lastName: "Testov", middleName: "Testovich", birth: "1955-02-22", address: "home 6", street: "Testovaya", city: "Moscow", country: "RU", phone: "89991234567", postcode: "12345")
    
// Указывайте дополнительные данные если это необходимо
let jsonData: [String: Any] = ["age":27, "name":"Ivan", "phone":"+79998881122"] // Любые другие данные, которые будут связаны с транзакцией, в том числе инструкции для создания подписки или формирования онлайн-чека должны обёртываться в объект cloudpayments. Мы зарезервировали названия следующих параметров и отображаем их содержимое в реестре операций, выгружаемом в Личном Кабинете: name, firstName, middleName, lastName, nick, phone, address, comment, birthDate.

let paymentData = PaymentData() 
    .setAmount(String(totalAmount)) // Cумма платежа в валюте, максимальное количество не нулевых знаков после запятой: 2
    .setCurrency(.ruble) // Валюта
    .setApplePayMerchantId(Constants.applePayMerchantID) // Apple pay merchant id (Необходимо получить у Apple)
    .setYandexPayMerchantId(Constants.yandexPayMerchantID) // Yandex pay merchant id (Необходимо получить у Yandex)
    .setDescription("Корзина цветов") // Описание оплаты в свободной форме
    .setAccountId("111") // Обязательный идентификатор пользователя для создания подписки и получения токена
    .setIpAddress("98.21.123.32") // IP-адрес плательщика
    .setInvoiceId("123") // Номер счета или заказа
    .setEmail("test@cp.ru") // E-mail плательщика, на который будет отправлена квитанция об оплате
    .setPayer(payer)
    .setJsonData(jsonData)                    
```

2. Создайте объект PaymentConfiguration, передайте в него объект PaymentData и ваш Public_id из [личного кабинета Cloudpayments](https://merchant.cloudpayments.ru/). Реализуйте протокол PaymentDelegate, чтобы узнать о завершении платежа

```
let configuration = PaymentConfiguration.init(
    publicId: Constants.merchantPublicId, // Ваш Public_id из личного кабинета
    paymentData: paymentData, 
    delegate: self,
    uiDelegate: self,
    scanner: nil,
    showEmailField: true, // Показывать поле ввода адреса для отправки квитанции при отображении формы ввода карточных данных (по умолчанию false)
    useDualMessagePayment: true, // Использовать двухстадийную схему проведения платежа, по умолчанию используется одностадийная схема
    disableApplePay: false, // Выключить Apple Pay, по умолчанию Apple Pay включен
    disableYandexPay: false, // Выключить Yandex Pay, по умолчанию Yandex Pay включен
    apiUrl: sApiUrl)
)
```
### Использование Yandex Pay:
Если вы планируете использовать Yandex Pay , вам необходимо зарегистрироваться в программе Yandex Pay, выполнить в своем приложении, авторизацию в Yandex Login SDK: https://yandex.ru/dev/mobileauthsdk/ и провести инициализацию в `AppDelegate.swift`

В `AppDelegate.swift` вашего проекта в методе `application(_:didFinishLaunchingWithOptions:)` осуществите инициализацию SDK:

Если в проекте используется YandexPay, то для настройки YandexLoginSDK используйте пункты 1-3 [инструкции](https://yandex.ru/dev/mobileauthsdk/doc/sdk/concepts/ios/2.0.0/sdk-ios-install.html).

Если в проекте не используется YandexPay, инициализацию проводить не нужно, также если вы не планируете использовать Yandex Pay , вам необходимо в объекте PaymentConfiguration при инициализации указать `disableYandexPay: true`

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    do {
        // Инициализируйте SDK 
        // Если в проекте используется YandexPay, то необходимо указать соответсвующие параметры:
        // yandexPayAppId - ваш appId, который вы получили при настройке YandexLoginSDK
        // sandboxMode - режим песочницы YandexPay
        let yaAppId = "..."
        try CloudPaymentsSDK.initialize(yandexPayAppId: yaAppId, yandexPaySandboxMode: false)
    } catch {
        fatalError("Unable to initialize CloudPaymentsSDK")
    }
        
    return true
}
```

3. Вызовите форму оплаты внутри своего контроллера

```
PaymentForm.present(with: configuration, from: self)
```

4. Сканер карт

Вы можете подключить любой сканер карт. Для этого нужно реализовать протокол PaymentCardScanner и передать объект, реализующий протокол, при создании PaymentConfiguration. Если протокол не будет реализован, то кнопка сканирования не будет показана

Пример со сканером [CardIO](https://github.com/card-io/card.io-iOS-SDK)

* Создайте контроллер со сканером и верните его в функции протокола PaymentCardScanner
```
extension CartViewController: PaymentCardScanner {
    func startScanner(completion: @escaping (String?, UInt?, UInt?, String?) -> Void) -> UIViewController? {
        self.scannerCompletion = completion
        
        let scanController = CardIOPaymentViewController.init(paymentDelegate: self)
        return scanController
    }
}
```
* После завершения сканирования вызовите замыкание и передайте данные карты
```
extension CartViewController: CardIOPaymentViewControllerDelegate {
    func userDidCancel(_ paymentViewController: CardIOPaymentViewController!) {
        paymentViewController.dismiss(animated: true, completion: nil)
    }
    
    func userDidProvide(_ cardInfo: CardIOCreditCardInfo!, in paymentViewController: CardIOPaymentViewController!) {
        self.scannerCompletion?(cardInfo.cardNumber, cardInfo.expiryMonth, cardInfo.expiryYear, cardInfo.cvv)
        paymentViewController.dismiss(animated: true, completion: nil)
    }
}
```


### Использование вашей платежная формы с использованием функций CloudPaymentsApi:

1. Создайте криптограмму карточных данных

```
// Обязательно проверяйте входящие данные карты (номер, срок действия и cvc код) на корректность, иначе функция создания криптограммы вернет nil.
let cardCryptogramPacket = Card.makeCardCryptogramPacket(with: cardNumber, expDate: expDate, cvv: cvv, merchantPublicID: Constants.merchantPulicId)
```

2. Выполните запрос на проведения платежа. Создайте объект CloudpaymentApi и вызовите функцию charge для одностадийного платежа или auth для двухстадийного. Укажите email, на который будет выслана квитанция об оплате.

```
let api = CloudpaymentsApi.init(publicId: Constants.merchantPulicId)
api.auth(cardCryptogramPacket: cardCryptogramPacket, cardHolderName: cardHolderName, email: nil, amount: String(total)) { [weak self] (response, error) in
    if let response = response {
        self?.checkTransactionResponse(transactionResponse: response, completion: completion)
    } else if let error = error {
        completion?(false, error.localizedDescription)
    }
}
```

3. Если необходимо, покажите 3DS форму для подтверждения платежа

```
let data = ThreeDsData.init(transactionId: transactionId, paReq: paReq, acsUrl: acsUrl)
let threeDsProcessor = ThreeDsProcessor()
threeDsProcessor.make3DSPayment(with: data, delegate: self)
```

4. Для получения формы 3DS и получения результатов прохождения 3DS аутентификации реализуйте протокол ThreeDsDelegate.

```
extension CheckoutViewController: ThreeDsDelegate {
    // Вы получаете объект WKWebView, который сами показываете нужным вам способом и в нужном вам месте
    func willPresentWebView(_ webView: WKWebView) {
        self.showThreeDsForm(webView)
    }

    // Для завершения оплаты в выполните метод post3ds CloudpaymentsApi. 
    // threeDsCallbackId - идентификатор, полученный в ответ на запрос на проведение платежа
    func onAuthotizationCompleted(with md: String, paRes: String) {
        hideThreeDs()
        post3ds(transactionId: md, paRes: paRes, threeDsCallbackId: threeDsCallbackId)
    }

    func onAuthorizationFailed(with html: String) {
        hideThreeDs()
        print("error: \(html)")
    }

}
```

#### Оплата с помощью Apple Pay с CloudPaymentsApi

[Об Apple Pay](https://developers.cloudpayments.ru/#apple-pay)

1. Создайте массив объектов PKPaymentSummaryItem используя информацию о товарах выбранных вашим клиентом
```
var paymentItems: [PKPaymentSummaryItem] = []
for product in CartManager.shared.products {
    let paymentItem = PKPaymentSummaryItem.init(label: product.name, amount: NSDecimalNumber(value: Int(product.price)!))
    paymentItems.append(paymentItem)
}
```
2. Укажите ваш Apple Pay ID и допустимые платежные системы
```
let applePayMerchantID = "merchant.com.YOURDOMAIN" // Ваш ID для Apple Pay
let paymentNetworks = [PKPaymentNetwork.visa, PKPaymentNetwork.masterCard] // Платежные системы для Apple Pay
```
3. Проверьте, доступны ли пользователю эти платежные системы
```
buttonApplePay.isHidden = !PKPaymentAuthorizationViewController.canMakePayments(usingNetworks: paymentNetworks) // Скройте кнопку Apple Pay если пользователю недоступны указанные вами платежные системы
```
4. Создайте и выполните запрос к Apple Pay
```
let request = PKPaymentRequest()
request.merchantIdentifier = applePayMerchantID
request.supportedNetworks = paymentNetworks
request.merchantCapabilities = PKMerchantCapability.capability3DS // Возможно использование 3DS
request.countryCode = "RU" // Код страны
request.currencyCode = "RUB" // Код валюты
request.paymentSummaryItems = paymentItems
let applePayController = PKPaymentAuthorizationViewController(paymentRequest: request)
applePayController?.delegate = self
self.present(applePayController!, animated: true, completion: nil)
```
5. Обрабатываем ответ от Apple Pay
```
extension CheckoutViewController: PKPaymentAuthorizationViewControllerDelegate {
    func paymentAuthorizationViewController(_ controller: PKPaymentAuthorizationViewController, didAuthorizePayment payment: PKPayment, completion: @escaping ((PKPaymentAuthorizationStatus) -> Void)) {
        completion(PKPaymentAuthorizationStatus.success)
        
        // Конвертируйте объект PKPayment в строку криптограммы
        guard let cryptogram = payment.convertToString() else {
            return
        }
               
        // Используйте методы API для выполнения оплаты по криптограмме
        // (charge (для одностадийного платежа) или auth (для двухстадийного))
        //charge(cardCryptogramPacket: cryptogram, cardHolderName: "")
        auth(cardCryptogramPacket: cryptogram, cardHolderName: "")
      
    }
    
    func paymentAuthorizationViewControllerDidFinish(_ controller: PKPaymentAuthorizationViewController) {
        controller.dismiss(animated: true, completion: nil)
    }
}
```

#### ВАЖНО:

При обработке успешного ответа от Apple Pay, обязательно выполните преобразование объекта PKPayment в криптограмму для передачи в платежное API CloudPayments

```
let cryptogram = payment.convertToString() 
```
После успешного преобразования криптограмму можно использовать для проведения оплаты.

### Другие функции

* Проверка карточного номера на корректность

```
Card.isCardNumberValid(cardNumber)

```

* Проверка срока действия карты

```
Card.isExpDateValid(expDate) // expDate в формате MM/yy

```

* Определение типа платежной системы

```
let cardType: CardType = Card.cardType(from: cardNumberString)
```

* Определение банка эмитента

```
CloudpaymentsApi.getBankInfo(cardNumber: cardNumber) { (info, error) in
    if let error = error {
        print("error: \(error.message)")
    } else {
        if let bankName = info?.bankName {
            print("BankName: \(bankName)")
        } else {
            print("BankName is empty")
        }
        
        if let logoUrl = info?.logoUrl {
            print("LogoUrl: \(logoUrl)")
        } else {
            print("LogoUrl is empty")
        }
    }
}
```

* Шифрование карточных данных и создание криптограммы для отправки на сервер

```
let cardCryptogramPacket = Card.makeCardCryptogramPacket(with: cardNumber, expDate: expDate, cvv: cvv, merchantPublicID: Constants.merchantPulicId)
```

* Шифрование cvv при оплате сохраненной картой и создание криптограммы для отправки на сервер

```
let cvvCryptogramPacket = Card.makeCardCryptogramPacket(with: cvv)
```

* Преобразование PKPayment в строку-криптограмму для отправки на сервер

```
let payment: PKPayment
let cryptogram = payment.convertToString()
```

* Отображение 3DS формы и получении результата 3DS аутентификации

```
let data = ThreeDsData.init(transactionId: transactionId, paReq: paReq, acsUrl: acsUrl)
let threeDsProcessor = ThreeDsProcessor()
threeDsProcessor.make3DSPayment(with: data, delegate: self)

public protocol ThreeDsDelegate: class {
    func willPresentWebView(_ webView: WKWebView)
    func onAuthotizationCompleted(with md: String, paRes: String)
    func onAuthorizationFailed(with html: String)
}
```

### Поддержка

По возникающим вопросам технического характера обращайтесь на support@cp.ru
