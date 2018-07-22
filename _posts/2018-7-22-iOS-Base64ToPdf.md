---
layout: post
title: Отображение Pdf из base64 в iOS!
---
Довольно часто заказчики просят реализовать в приложении возможность показа PDF документа из base64, который присылает сервер.

Давайте разберемся, как это реализовать.

![](https://image.flaticon.com/icons/png/128/35/35653.png)

Для тестирования вы можете взять любой pdf файл и конвертировать с помощью [любого](http://base64converter.com/) онлайн конвертера в base64.
 
iOS предоставляет вам возможность отобразить ваш PDF в WebView.
Для этого вам необходимо получить объект Data из вашей base64-строки:
``` swift 4
extension String {
    func base64ToData() -> Data? {
        return Data(base64Encoded: self)
    }
}
```
Далее вы можете отобразить ваши бинарные данные в WebView, указав необходимый вам [mimeType](http://www.iana.org/assignments/media-types/media-types.xhtml)
```  swift 4
  enum Constants {
        static let defaultUrl = "https://www.default.com"
        static let mimeTypePDF = "application/pdf"
        static let defaultNamePDFFile = "pdfFile.pdf"
    }
    
    static func makeWebView(base64String: String, frame: CGRect) -> WKWebView? {
        let webView = WKWebView(frame: frame)
        
        guard let data = base64String.base64ToData(),
            let url = URL(string: Constants.defaultUrl)
            else { return nil }
        
        webView.load(data, mimeType: Constants.mimeTypePDF, characterEncodingName: "", baseURL: url)
        return webView
    }
 ```
 
 Так же помимо просмотра pdf часто есть необходимость в возможности поделиться этим pdf файлом, через различные каналы, имеющиеся на телефоне, а так же возможность отправки по электронной почте.
 Тут нам на помощь приходит **[UIActivityViewController](https://developer.apple.com/documentation/uikit/uiactivityviewcontroller)**.
 Перед использование данного контроллера вы должны локально сохранить ваш файл и получить на него ссылку, а после передать в контроллер.
 Сохранить файл можно следующим образом: 
 
 ``` swift 4
    static func saveBase64StringToPDF(_ base64String: String) -> URL? {
        guard
            var documentsURL = (FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)).last,
            let convertedData = Data(base64Encoded: base64String)
            else {
                //handle error when getting documents URL
                return nil
        }
        
        //name your file however you prefer
        documentsURL.appendPathComponent(Constants.defaultNamePDFFile)
        
        do {
            try convertedData.write(to: documentsURL)
        } catch {
            //handle write error here
            return nil
        }
        
        return documentsURL
    }
  ```
  Далее при инициализации UIActivityViewController передаем данный url, далее мы можем указать, какие каналы передачи мы хотим [убрать](https://developer.apple.com/documentation/uikit/uiactivityviewcontroller/1622009-excludedactivitytypes?language=objc).
  ``` swift 4
  static func pdfActivityController(url: URL) -> UIActivityViewController {
        let objectsToShare = [url]
        let activityController =
            UIActivityViewController(activityItems: objectsToShare,
                                     applicationActivities: nil)
        
       
        
        activityController.excludedActivityTypes = Constants.excludedPdfActivities
        return activityController
    }
 ```
 После проделанных операций мы можем показать экран 
 ``` swift 4
  DispatchQueue.main.async {
                 self.present(activityController, animated: true, completion: nil)
            }
 ```          
 Проект полностью можно посмотреть на [GitHub](https://github.com/ELezov/iOS-PDF)
 
 **Eugene Lezov** on [GitHub](https://github.com/ELezov)
           
       
 
 
