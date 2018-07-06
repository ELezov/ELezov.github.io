---
layout: post
title: Биометрическая аутентификация в iOS!
---

![][LOGO]
##### Биометрическая аутентификация в iOS представлена (на момент написания) двумя видами:
* аутентификация по отпечатку пальца (**Touch ID**)
* аутентификация по форме лица (**Face ID**)

В iOS 8 появилась возможность использования технологий **Touch ID**. Впервые данная функциональность была встроена в iPhone 5s. 
**Face ID** - это 3d сканер формы лица. На данный момент имеется лишь на iPhone X и заменяет собой **Touch ID**.

Работа с **Touch ID**/**Face ID** осуществляется через **LocalAuthentication.framework**.

### Порядок действий
1. ###### Необходимо создать context проверки подлинности:

    ```let context = LAContext()```
2. ###### Необходимо проверить, можем ли мы использовать биометрическую аутентификацию 
    ```context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &authError)```
3. ###### Если использование биометрической аутентификации возможно, то пробуем авторизоваться

``` swift
    let reason = /* Здесь указывается причина для авторизации*/
    context.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics,
                                       localizedReason: reason) { success, evaluateError in
    	/* handle error and do smth */
    }
```
    
### Исключения и ошибки на которые стоит обратить внимание

##### 0. Отслеживание изменения отпечатков пальца (удаление / добавление)
Это недокументированное исключение, отслеживать событие необходимо вручную.
Я воспользовался примером со [stackoverflow][SOFA] и написал такую функцию:
``` swift
// Проверяем, были ли изменены биометрические данные
func biometricDateIsValid() -> Bool {
    let context = LAContext()
    context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil)
    var result: Bool = true
    // Получаем сохраненный биометрические данные
    let oldDomainState = UserDefaultsHelper.biometricDate
    // Получаем текущие биометрические данные
    guard let domainState = context.evaluatedPolicyDomainState
        else { return result }
    // Сохраняем новые текущие биометрические данные в UserDefaults
    UserDefaultsHelper.biometricDate = domainState
    result = (domainState == oldDomainState || oldDomainState == nil)
    return result
  }
```
После проверки на возможность использования биометрической аутентификации мы получаем текущий **PolicyDomainState** из context и сравниваем с тем,который был раньше. ( При каждой успешной аутентификации мы сохраняем текущий в UserDefaults. )

    В тех случая, когда биометрическая аутентификация невозможна, мы получаем ошибку. 
    Среди них есть виды ошибок, которые рекомендуется обрабатывать.

##### 1. Биометрическая аутентификация недоступна на устройстве.
##### 2. Аутентификация закончилась неуспешно, так как превышено количество попыток входа.
##### 3. Аутентификация закончилась неуспешно, так как на телефоне отсутствуют биометрические данные.
Более подробно о типах ошибок можно прочитать [тут][ERRORCODE].
``` swift
let errorCode = error._code
if #available(iOS 11.0, *) {
    if [LAError.biometryLockout.rawValue,
        LAError.biometryNotEnrolled.rawValue].contains(where: { $0 == errorCode }) {
        /* показываем alert с возможностью перехода в настройки телефона */
    } else {
        /* Уведомляем пользователя об ошибке и невозможности аутентификации*/
    }
} else {
    if [LAError.touchIDLockout.rawValue,
        LAError.touchIDNotEnrolled.rawValue].contains(where: { $0 == errorCode }) {
        /* показываем alert с возможностью перехода в настройки телефона */
    } else {
        /* Уведомляем пользователя об ошибке и невозможности аутентификации*/
    }
}
```
Ошибка **(1)** обычно обрабатывается по общим правилам, а ошибки **(2)**  и **(3)** немного иначе.

Ошибка **(2)** говорит нам о том, что превышено количество попыток ввода для аутентификации и о заблокированном состоянии технологии биометрической аутентификации. Для продолжения работы мы даем возможность пользователя попасть в настройки телефона и разблокировать механизм биометрической аутентификации.

В случае ошибки **(3)** необходимо пользователю также дать возможность перейти в настройки телефона и добавить биометрическую информацию на телефон.

Для перехода в настройки телефона из нашего приложения необходимо следующее:
``` swift
func showDeviceSettings() {
    guard let settingURL =   URL(string : "App-Prefs:") else { return }    
    if UIApplication.shared.canOpenURL(settingURL){
        if #available(iOS 10.0, *) {
            UIApplication.shared.open(settingURL, options: [:], completionHandler: nil)
        } else {
            UIApplication.shared.openURL(settingURL)
        }
    }
}
```

Интересная особенность заключается в том, что в **iOS 10** вы могли перейти сразу на экран настройки **Touch ID** с помощью следующего URL:
``` swift 
URL(string : "App-Prefs:root=TOUCHID_PASSCODE") 
```
В **iOS 11** вы можете пройти только для главного экрана настроек.

##### 4. Экран авторизации и Face ID.

Часто в приложении на экране авторизации сразу включена биометрическая аутентификация. В случае c **Face ID**, в отличие от **Touch ID**, от пользователя не требуется почти никакой реакции (просто смотреть в телефон). Поэтому авторизация с **Face ID** происходит моментально. 

Но есть случаи, когда мы хотим разлогиниться и попадаем на экран авторизации,  на котором с помощью **Face ID** происходит чуть ли не мгновенная авторизация. В итоге получаем замкнутый цикл. ***Не приятно, не так ли?*** Поэтому этот случай требует `обязательного` отслеживания.

#### Технология “быстрый вход”

Данная функциональность реализуется всего лишь с помощью добавления одного параметра при авторизации.
``` swift
context.touchIDAuthenticationAllowableReuseDuration = 30
```
Этот параметр работает `только` с  **Touch ID** *(Проверено и на Face ID)*.
Данный параметр определяет продолжительность валидности последней сессий биометрической аутентификации. В данном случае имеется в виду время с последней разблокировки телефона *(не приложения)* с помощью биометрических данных.

Если вы уложились в это время, то аутентификация пройдет мгновенно.
По умолчанию данный параметр равен **0**.

`!` Есть кейс, когда пользователь, находясь в приложении, блокировал девайс, а потом после разблокировки телефона с помощью биометрических данных хотел разлогиниться. Тут можно получить похожую с Face ID ошибку. Этот случай тоже `необходимо` отслеживать.

##### Проект полностью вы можете посмотреть [тут][PROJECT]

[**Eugene Lezov, iOS Developer**][PROFILE_GITHUB]

[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)

   [LOGO]: <https://www.intego.com/mac-security-blog/wp-content/uploads/2017/10/Touch-ID-vs-Face-ID.png>
   [SOFA]: <https://stackoverflow.com/questions/25669172/ios8-touchid-detection-if-fingerprint-was-added>
   [ERRORCODE]: <https://developer.apple.com/documentation/localauthentication/laerror/code>
   [PROFILE_GITHUB]: <https://github.com/ELezov>
   [PROJECT]: <https://github.com/ELezov/iOS-BiometricLocalAuth/tree/develop>
