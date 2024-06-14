# Инструкция для настройки и тестирования Альфа-Линк - 1С:Директбанк через API



# Содержание
[1. Скачивание архива со всеми необходимыми файлами](#1)

[2. Стандартный алгоритм взаимодействия](#2)

[3. Данные тестовой среды](#3)

[4. Описание заголовков](#4)

[5. Правила формирования пакетов](#5)

[6. Формирование документа в пакете на примере запроса выписки](#6)

[7. Рекомендации по отправке документов зарплатного проекта](#7)

[8. Подписание документов](#8)

[9. Инструкция по работе с тестовым проектом Postman](#9)

[10. Справочная информация и поддержка](#10)



## 1. Скачивание архива со всеми необходимыми файлами<a name="1"></a>

По [ссылке](https://github.com/Host-to-Host-Instructions/alfalink-1c-directbank-via-api/archive/refs/heads/master.zip) можно скачать архив с необходимыми для тестирования файлами.

**Архив содержит:**

- Данную инструкцию по подключению (**README.md и в формате pdf**)
- Файл с тестовым проектом для Postman (**1C-DirectBank. API - Тестирование.postman_collection.json**)
- Файл сертификата для тестовой организации **directBank_cert_psw_123456.pfx** для подписания платежей
- Файл с корневыми сертификатами УЦ Альфа-Банка **cacerts.p7b**
- Список отзыва сертификатов **certificate_revocation_list.crl** Обновляется **каждый** месяц 14-го числа.
- Папка с примерами документов в формате xml (**examples**)

## 2. Стандартный алгоритм взаимодействия<a name="2"></a>

**Пакет, отправляемый пользователем содержит документ (запрос выписки/платёж/зарплатную ведомость/заявку на открытие ЛС), зашифрованный в base64, в поле Packet.Document.Data.**

**Пакет, получаемый пользователем от банка содержит сформированную выписку или статус отправленного пакета, зашифрованный в base64, в поле Packet.Document.Data.**

1. **POST /API/v1/directbank/Logon** - создание сессии для отправки и получения пакетов. В ответ возвращается xml документ с идентификатором сессии в поле SID. Время жизни сессии - 600 сек. Идентификатор сессии необходим при любых отправке/получении документов.
2. **POST /API/v1/directbank/SendPack** - отправка пакета документов. Внутри пакета может быть **запрос выписки/платёжное поручение/ведомость в банк/заявку на открытие ЛС**.  В хедере SID указывается идентификатор сессии. XML пакета помещается в body запроса. 
3. **GET /API/v1/directbank/GetPackList** - получить список пакетов с ответами от банка. В ответ возвращается список идентификаторов непрочитанных пользователем пакетов с ответами от банка. После отправки документа первым придёт пакет с результатом отправки («Принят»/Сообщение об ошибке).
4. **GET /API/v1/directbank/GetPack** - получить пакет по его идентификатору. Внутри пакета может содержаться **статус запроса** или **сформированная выписка**. Аргумент запроса id - идентификатор пакета из списка, полученного в пункте 3. 

| №    | Метод | URL                            | Описание                                         | Что отправляем?                                              | Что получаем?                                                |
| ---- | ----- | ------------------------------ | ------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1    | POST  | /API/v1/directbank/Logon       | создание сессии для отправки и получения пакетов | Логин/пароль                                                 | Идентификатор сессии                                         |
| 2    | POST  | /API/v1/directbank/SendPack    | отправка пакета документов                       | Идентификатор сессии<br />Пакет с запросом выписки/платежом/ведомостью/заявкой | Идентификатор запроса (дальше не используется)/Ошибка (если запрос/пакет некорректный) |
| 3    | GET   | /API/v1/directbank/GetPackList | получить список пакетов с ответами от банка      | Идентификатор сессии                                         | Список идентификаторов пакетов, которые банк уже сформировал в ответ пользователю (если получены не все пакеты - повторть запрос) |
| 4    | GET   | /API/v1/directbank/GetPack     | получить пакет по его идентификатору             | Идентификатор сессии<br />Идентификатор пакета (для каждого пакета из п. 3 - свой запрос) | Пакет с ответом от банка. В нём может быть статус запроса или сформированная выписка (в поле Packet.Document.Data) |

Например, для получения выписки необходимо:

1. Создать сессию (Logon)
2. Отправить пакет с **запросом выписки** (SendPack)
3. Получить список пакетов от банка (должны прийти 2 пакета - со статусом запроса и со сформированной выпиской) (GetPackList)
4. Получить содержимое пакета со статусом запроса (в случае успеха статус - **Принят**) (GetPack)
5. Если статус успешный - получить содержимое пакета со сформированной выпиской (GetPack)

**Если в статусе запроса - ошибка, то второго пакета не будет!**


## 3. Данные тестовой среды<a name="3"></a>

**Данные для авторизации:**

| Параметр | Значение |
| ------------- |-------------|
| Сервер | https://alfa-link-int.alfabank.ru/API/v1/directbank |
| Логин | directBank |
| Пароль | 123456 |

**Реквизиты компании:**

| Параметр | Значение |
| ------------- |-------------|
| Краткое наименование | ООО "Тест Директ Банк" |
| Полное наименование | Общество с ограниченной ответственностью "Тест Директ Банк" |
| ИНН | 0329156629 |
| КПП | 043360081 |
| Номер счета | 40702810701300009144 |
| Банк | АО "АЛЬФА-БАНК" |
| Корсчет банка | 30101810200000000593 |
| БИК банка | 044525593 |

**Реквизиты компании получателя (для тестирования платежей):**

| Параметр | Значение |
| ------------- |-------------|
| Краткое наименование | ООО "Тест Директ банк Получатель" |
| Полное наименование | Общество с ограниченной ответственностью "Тест Директ банк Получатель" |
| ИНН | 0329156636 |
| КПП | 770101999 |
| Номер счета | 40702810300000019796 |
| Банк | АО "АЛЬФА-БАНК" |
| Корсчет банка | 30101810200000000593 |
| БИК банка | 044525593 |

**Реквизиты для тестирования зарплатного проекта (зарплатная ведомость и заявка на открытие лицевых счетов):**

Для тестирования отправки документов по зарплатному проекту необходимо использовать специального пользователя:

**Логин**: azon_no_sign

**Пароль**: 123456

При отправке необходимо создать новую сессию с данными авторизации данного пользователя и использовать её при отправке документов зарплатного проекта.

Также, следует обратить внимание, что **Header: customerid=40702810300000003333**

| Параметр                            | Значение                                        |
| ----------------------------------- | ----------------------------------------------- |
| Краткое наименование                | ООО "МАНИ"                                      |
| Полное наименование                 | Общество с ограниченной ответственностью "МАНИ" |
| ИНН                                 | 0551666943                                      |
| Номер счета                         | 40702810300000003333                            |
| Банк                                | АО "АЛЬФА-БАНК"                                 |
| Корсчет банка                       | 30101810200000000593                            |
| БИК банка                           | 044525593                                       |
| Номер зарплатного договора          | 00CXDP                                          |
| ФИО сотрудника                      | Семенов Семен Семенович                         |
| Отделение банка                     | 0000                                            |
| Лицевой счёт сотрудника             | 40817810804040000004                           |
| Серия паспорта                      | 99 89                                           |
| Номер паспорта                      | 898989                                          |
| Дата выдачи паспорта                | 2017-07-14                                      |
| Кем выдан паспорт                   | Уфмс                                            |
| Код подразделения                   | 090-909                                         |
| Дата рождения                       | 1970-07-23                                      |
| Место рождения (полное/сокращённое) | Москва/Москва                                   |
| Эмбоссированный текст               | Semenov / Semen / MR                            |

## 4. Описание заголовков <a name="4"></a>

| Заголовок |	Значение | Описание |	Комментарий |
| --------- | -------- | -------- | ----------- |
| Content-Type |	application/xml |	MIME тип передаваемого контента | Все пакеты в формате xlm |
| customerid | 40702810601300003716 | Id клиента | В качестве идентификатора клиента используется один из его счетов |
| apiversion | 2.2.2 | Версия формата данных | Версия формата данных, в которых осуществляется обмен с банком (текущая рабочая версия - 2.2.2) |
| Authorization | | Данные для авторизации | Используется логин и пароль для авторизации |
| SID	| | Идентификатор сессии | Не используется в методе Logon |


## 5. Правила формирования пакетов <a name="5"></a>

Формат пакетов, используемых при обмене соответствует схемам даных 1С.

**Пакет, отправляемый пользователем содержит документ (запрос выписки/платёж/зарплатную ведомость/заявку на открытие ЛС), зашифрованный в base64, в поле Packet.Document.Data.**

**Пакет, который пользователь получает в ответ, содержит статус отправленного документа, либо сформированную выписку, зашифрованные в base64, в поле Packet.Document.Data.**

Ссылка на xsd-схемы: https://github.com/1C-Company/DirectBank/blob/master/doc/xsd-scheme/readme.md

Описание полей пакета:

| Параметр  |	Значение | Описание |
| --------- | -------- | -------- |
| Packet@id	| Новый id	| Уникальный идентификатор транспортного пакета |
| Packet@CreationDate	| dateTime	| Дата и время создания транспортного контейнера (системное время) |
| Packet@formatVersion	| 2.2.2	| Версия формата данных, в которых осуществляется обмен с банком (текущая рабочая версия - 2.2.2) |
| Packet@UserAgent	| Наименование системы	| Наименование системы, в рамках которых формируется документ |
| Packet.Document@id	| Новый id	| Уникальный идентификатор электронного документа |
| Packet.Document@dockind	| Тип документа	| Основные типы: 10 - платёжное поручение, 14 - запрос выписки, 21 - зарплатная ведомость, 19 - заявка на открытие ЛС |
| Packet.Document@formatVersion	| 2.2.2	| Версия формата данных, в которых осуществляется обмен с банком (текущая рабочая версия - 2.2.2) |
| Packet.Sender.Customer@id	| Id клиента	| В качестве идентификатора клиента используется один из его счетов. |
| Packet.Sender.Customer@Name	| Наименование организации	| Указывается полное наименование организации |
| Packet.Sender.Customer@inn	| ИНН организации	| |
| Packet.Recipent.Bank@bic	| БИК банка	| |
| Packet.Recipent.Bank@Name	| Наименование банка	| |
| Packet.Document.Data	| Данные электронного документа	| Документ кодируется в base64 |



## 6. Формирование документа в пакете на примере запроса выписки<a name="6"></a>

Любой документ для отправки кодируется в строку формата base64 и помещается в поле Data пакета.

Схемы документов см. тут: https://github.com/1C-Company/DirectBank/blob/master/doc/xsd-scheme/readme.md

**Запрос выписки - документ, отправляемый пользователем в банк, который содержит информацию о том, по какому счёту и за какой период необходимо сформировать выписку.**

**Запрос выписки, закодированный в base64, размещается в поле Packet.Document.Data отправляемого пакета.**

Для того, чтобы получить готовую выписку, необходимо после отправки запроса выписки(SendPack) получить список пакетов с ответами (GetPackList). Если выписка успешно сформирована, то будет получено два пакета: 

1. Со статусом запроса выписки
2. Со сформированной выпиской

Каждый пакет можно получить по его ID с помощью запроса GetPack.

Описания полей запроса выписки:

| Параметр  |	Значение | Описание |
| --------- | -------- | -------- |
| StatementRequest@id	| Новый id	| Уникальный идентификатор электронного документа (должен совпадать с Packet.Document@id) |
| StatementRequest@CreationDate	| dateTime	| Дата и время создания выписки (системное время) |
| StatementRequest.Data.DateFrom	| dateTime	| Дата выписки «с» (включая указанный день) |
| StatementRequest.Data.DateTo	| dateTime	| Дата выписки «по» (включая указанный день) |
| StatementRequest@formatVersion	| 2.2.2	| Версия формата данных, в которых осуществляется обмен с банком (текущая рабочая версия - 2.2.2) |
| StatementRequest@UserAgent	| Наименование системы	| Наименование системы, в рамках которых формируется документ |
| StatementRequest.Sender@id	| Id клиента	| В качестве идентификатора клиента используется один из его счетов (должно совпадать с packet.Sender.Customer@id) |
| StatementRequest.Sender@Name	| Наименование организации	| Должно совпадать с Packet.Sender.Customer@Name |
| StatementRequest.Sender@inn	| ИНН организации	| Должно совпадать с Packet.Sender.Customer@inn |
| StatementRequest.Sender@kpp	| КПП организации| 	|
| StatementRequest.Recipent@bic	| БИК банка	| Должно совпадать с Packet.Sender.Recipent.Bank.bic |
| StatementRequest.Recipent@Name	| Наименование банка	| Должно совпадать с Packet.Sender.Recipent.Bank.Name |
| StatementRequest.Data.StatementType	| 0	| Тип выписки: 0 - Окончательная выписка; 1 - Промежуточная выписка; 2 - Текущий остаток на счете |
| StatementRequest.Data.Account	| Номер счета	| Номер счета, по которому запрашивается выписка (При тестировании необходимо использовать счёт, указанный в проекте Postman) |
| StatementRequest.Data.Bank@bic	| БИК банка	| Должно совпадать с Packet.Recipent.Bank@bic |
| StatementRequest.Data.Bank@name	| Наименование банка	| Должно совпадать с Packet.Recipent.Bank@Name |



## 7. Рекомендации по отправке документов зарплатного проекта<a name="7"></a>

Документы зарплатного проекта отправляются аналогично платёжному поручению - сначала отправляется документ (SendPack), затем необходимо получать статусы документа (GetPackList -> GetPack).

**Правила заполнения полей зарплатной ведомости**

| Обяз-сть | Поле в документе                       | Описание                                                     |
| -------- | -------------------------------------- | ------------------------------------------------------------ |
| R        | СчетаПК@**ДатаФормирования**           | Формат - YYYY-MM-DD                                          |
| R        | СчетаПК@**НаименованиеОрганизации**    | Краткое наименование организации                             |
| R        | СчетаПК@**ИНН**                        | ИНН организации                                              |
| R        | СчетаПК@**РасчетныйСчетОрганизации**   | Расчетный Счет зарплатного проекта                           |
| R        | СчетаПК@**ИдПервичногоДокумента**      | 1С идентификатор документа - генерируется на стороне пользователя (должен быть уникальным) |
| R        | СчетаПК@**НомерДоговора**              | Поле "Номер" в блоке "Сведения о договоре" для выбранного в ведомости зарплатного проекта |
| R        | СчетаПК@**БИК**                        | БИК банка, указанного в зарплатном проекте                   |
| R        | СчетаПК@**НомерРеестра**               | Номер реестра - макс длина 11 цифр, генерируется на стороне пользователя |
| R        | СчетаПК@**ДатаРеестра**                | Дата формирования реестра (Формат - YYYY-MM-DD)              |
| R        | Сотрудник@**Нпп**                      | Номер строки в ведомости в банк - генерируется на стороне пользователя |
| R        | Сотрудник.**Фамилия**                  | Фамилия сотрудника                                           |
| R        | Сотрудник.**Имя**                      | Имя сотрудника                                               |
| O        | Сотрудник.**Отчество**                 | Отчество сотрудника                                          |
| R        | Сотрудник.**ОтделениеБанка**           | Отделение банка для выбранного в ведомости зарплатного проекта<br />Список кодов отделений можно найти на странице https://alfabank.ru/sme/salaryproject/<br/>Раздел "Полезно знать", вкладка "Управление зарплатным проектом", подраздел "Открытие счетов и выпуск зарплатных карт для сотрудников резидентов РФ", п.5 "Справочник отделений"<br/>Скачать XLS файл, в нём таблица "Отделение доставки карты", столбец - "Цифровой код"<br/>**XLS файл периодически обновляется, рекомендуется проверять обновления раз в полгода** |
| R        | Сотрудник.**ЛицевойСчет**              | Номер лицевого счета сотрудника                              |
| R        | Сотрудник.**Сумма**                    | Сумма к выплате сотруднику                                   |
| R        | Сотрудник.**КодВалюты**                | Код валюты выплаты (Сейчас ведомости формируются только в рублях, код по умолчанию - 643) |
| R        | **ВидЗачисления**                      | На данный момент реализована только заработная плата, поэтому указывается константа - 01 |
| R        | КонтрольныеСуммы.**КоличествоЗаписей** | Количество строк в ведомости                                 |
| R        | КонтрольныеСуммы.**СуммаИтого**        | Общая сумма к выплате                                        |

**Правила заполнения полей заявки на открытие лицевых счетов**

- Для поля **Место рождения** необходимо указать как минимум один из тегов Город/Район/Регион/Населённый пункт (полное название и сокращённое)
- Поле **Эмбоссированный текст** содержит имя сотрудника на зарплатной карте. Разделяется на три поля, третье может отсутствовать. Примеры заполнения: TATIANA M/IVANOVA или TANIA/IVANOVA/MRS

| Обяз-сть | Поле в документе                                             | Описание                                                     |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| R        | СчетаПК@**ДатаФормирования**                                 | Формат - YYYY-MM-DD                                          |
| R        | СчетаПК@**НомерРеестра**                                     | Номер реестра - макс длина 11 цифр, генерируется на стороне пользователя |
| R        | СчетаПК@**ДатаРеестра**                                      | Дата реестра (Формат - YYYY-MM-DD)                           |
| R        | СчетаПК@**НомерДоговора**                                    | Номер ЗП договора                                            |
| R        | СчетаПК@**БИК**                                              | БИК банка, указанного в зарплатном проекте                   |
| R        | СчетаПК@**НаименованиеОрганизации**                          | Наименование организации                                     |
| R        | СчетаПК@**ИНН**                                              | ИНН организации                                              |
| R        | СчетаПК@**РасчетныйСчетОрганизации**                         | Расчетный Счет зарплатного проекта                           |
| R        | СчетаПК@**ИдПервичногоДокумента**                            | 1С идентификатор документа - генерируется на стороне пользователя (должен быть уникальным) |
| R        | Сотрудник@**НПП**                                            | НПП сотрудника. Идентификатор строки заявки на открытие ЛС для одного сотрудника - генерируется на стороне пользователя |
| R        | Сотрудник.**Фамилия**                                        | Фамилия сотрудника                                           |
| R        | Сотрудник.**Имя**                                            | Имя сотрудника                                               |
| О        | Сотрудник.**Отчество**                                       | Отчество сотрудника                                          |
| R        | Сотрудник.**ОтделениеБанка**                                 | Отделение банка<br />Список кодов отделений можно найти на странице https://alfabank.ru/sme/salaryproject/<br/>Раздел "Полезно знать", вкладка "Управление зарплатным проектом", подраздел "Открытие счетов и выпуск зарплатных карт для сотрудников резидентов РФ", п.5 "Справочник отделений"<br/>Скачать XLS файл, в нём таблица "Отделение доставки карты", столбец - "Цифровой код"<br/>**XLS файл периодически обновляется, рекомендуется проверять обновления раз в полгода** |
| R        | Сотрудник.УдостоверениеЛичности.**ВидДокумента**             | Вид документа                                                |
| R        | Сотрудник.УдостоверениеЛичности.**Серия**                    | Серия паспорта                                               |
| R        | Сотрудник.УдостоверениеЛичности.**Номер**                    | Номер паспорта                                               |
| R        | Сотрудник.УдостоверениеЛичности.**ДатаВыдачи**               | Дата выдачи паспорта                                         |
| R        | Сотрудник.УдостоверениеЛичности.**КемВыдан**                 | Кем выдан паспорт                                            |
| R        | Сотрудник.УдостоверениеЛичности.**КодПодразделения**         | Код подразделения                                            |
| R        | Сотрудник.УдостоверениеЛичности.**КодВидаДокумента**         | Код вида документа                                           |
| R        | Сотрудник.**ДатаРождения**                                   | Дата рождения сотрудника                                     |
| R        | Сотрудник.**Пол**                                            | Пол, необходимо F переводить в "Женский", М - переводить в "Мужской" |
| O        | Сотрудник.**Должность**                                      | Должность                                                    |
| O        | Сотрудник.МестоРождения.Регион.**РегионНазвание**            | Регион места рождения (полное)                               |
| O        | Сотрудник.МестоРождения.Регион.**РегионСокращение**          | Регион места рождения (сокращение)                           |
| O        | Сотрудник.МестоРождения.Район.**РайонНазвание**              | Район места рождения (полное)                                |
| O        | Сотрудник.МестоРождения.Район.**РайонСокращение**            | Район места рождения (сокращение)                            |
| O        | Сотрудник.МестоРождения.Город.**ГородНазвание**              | Город места рождения (полное)                                |
| O        | Сотрудник.МестоРождения.Город.**ГородСокращение**            | Город места рождения (сокращение)                            |
| O        | Сотрудник.МестоРождения.НаселенныйПункт.**НаселенныйПунктНазвание** | Населённый пункт места рождения (полное)                     |
| O        | Сотрудник.МестоРождения.НаселенныйПункт.**НаселенныйПунктСокращение** | Населённый пункт места рождения (сокращение)                 |
| R        | Сотрудник.ЭмбоссированныйТекст@**Поле1**                     | Эмбосированный текст Поле 1                                  |
| R        | Сотрудник.ЭмбоссированныйТекст@**Поле2**                     | Эмбосированный текст Поле 2                                  |
| О        | Сотрудник.ЭмбоссированныйТекст@**Поле3**                     | Эмбосированный текст Поле 3                                  |
| R        | Сотрудник.**КодВалюты**                                      | Код валюты счета                                             |
| R        | Сотрудник.**Резидент**                                       | Резидент (true/false)                                        |
| R        | Сотрудник.**Гражданство**                                    | Гражданство                                                  |
| R        | Сотрудник.**МобильныйТелефон**                               | Мобильный телефон                                            |
| O        | Сотрудник.**ТабельныйНомер**                                 | Табельный номер сотрудника                                   |
| O        | Сотрудник.**ДатаОформления**                                 | Дата оформления                                              |
| O        | Сотрудник.**СуммаЗаработнойПлаты**                           | Сумма заработной платы                                       |
| O        | Сотрудник.**ДатаВыплаты**                                    | Дата выплаты                                                 |
| O        | Сотрудник.**КонтактныйМобильныйТелефон**                     | Контактный номер мобильного телефона                         |
| R        | КонтрольныеСуммы.**КоличествоЗаписей**                       | Количество строк в ведомости                                 |

## 8. Подписание документов<a name="8"></a>

В зависимости от настроек обмена с банком документы могут иметь различные маршруты подписания (подробнее https://github.com/1C-Company/DirectBank/blob/master/doc/common-section/type-tables.md#edo-Settings_Data_CryptoParameters_CustomerSignature_GroupSignatures)

На тестовом стенде по умолчанию устанавливаются следующие маршруты (в случае необходимости их можно изменить.):
- Платежное поручение – единоличная подпись/2 подписи;
- Запрос выписки – без подписи;
- Просмотр выписки – без подписи;
- Зарплатная ведомость - без подписи;
- Заявка на открытие лицевых счетов - без подписи.

Электронная подпись формируется в формате PKCS #7 Detached. Подпись хранится в отдельных тегах схемы 1С (теги описаны в схеме XML-схема описания общих типов: 1C-Bank_Exch-Common.xsd). 

Для автоматического подписания документов необходимо установить и использовать следующие инструменты (**только, если нужно отправлять платежи!**):
- КриптоПро CSP с серверной лицензией (доступен пробный период 90 дней), 
- КриптоПро JavaCSP с серверной лицензией (доступен пробный период 90 дней), 
- Jdk. 

https://www.cryptopro.ru/products/csp/jcsp https://www.cryptopro.ru/products/csp/jcp/downloads 

Пример подписанного файла в payment_directbank.xml (в примерах) 

В качестве примера кода для подписи можно использовать файл PKCS7Example.java (в примерах, это официальный пример из пакета КриптоПро JavaCSP). 

Больше примеров можно найти в архиве samples-sources.jar самого пакета КриптоПро JavaCSP. 

## 9. Инструкция по работе с тестовым проектом Postman<a name="9"></a>

1. Создать сессию через POST-запрос «Создать сессию». Для этого

   1.1. На вкладке «Auth» ввести логин/пароль клиента, который был выдан при подключении к тестовому стенду Альфа-Линк (по умолчанию можно тестировать на пользователе, который сохранен в проекте Postman)

   1.2. На вкладке «Headers» в поле «CustomerId» ввести номер счета клиента, с которым был настроен тестовый обмен с банком

   1.3. Нажать «Send» и скопировать ответ «SID»

2. Отправить пакет с документом (запрос выписки/платёжное поручение/ведомость в банк/заявку на открытие ЛС) через POST-запрос «Отправить пакет с Запросом выписки». Для этого

   2.1. На вкладке «Auth» ввести логин/пароль клиента, который был выдан при подключении к тестовому стенду Альфа-Линк Линк (по умолчанию можно тестировать на пользователе, который сохранен в проекте Postman)

   2.2. На вкладке «Headers» в параметр «CustomerId» ввести номер счета клиента, в параметр «SID» = значение из 1.1

   2.3. На вкладке «Body» внести исходных xml пакета с документом (Платеж, Запрос выписки, Ведомость в банк), созданных по схемам 1C

   2.4. Нажать «Send»

3. Получить список пакетов с ответами от банка, которые не были получены клиентом через Get-запрос «Получить список пакетов с ответами от банка»

   3.1. На вкладке «Auth» ввести логин/пароль клиента, который был выдан при подключении к тестовому стенду Альфа-Линк

   3.2. На вкладке «Headers» в параметр «CustomerId» ввести номер счета клиента, в параметр «SID» = значение из 1.1

   3.3. Нажать «Send» и скопировать значения идентификаторов пакетов

4. Получить значение пакета из списка через Get-запрос «Получить значение пакета с ответом»

   4.1. На вкладке «Auth» ввести логин/пароль клиента, который был выдан при подключении к тестовому стенду Альфа-Линк

   4.2. На вкладке «Headers» в параметр «CustomerId» ввести номер счета клиента, в параметр «SID» = значение из 1.1

   4.3. На вкладке «Params» ввести параметр из 3.3 (выбранный идентификатор пакета)

   4.4. Нажать «Send» 



## 10. Справочная информация и поддержка<a name="10"></a>

В случае возникновения вопросов/проблем вы можете:

- Найти решение своего вопроса на [странице вопросов к документации](https://github.com/Host-to-Host-Instructions/alfalink-1c-directbank-via-api/issues)
- Создать тикет на [странице документации](https://github.com/Host-to-Host-Instructions/alfalink-1c-directbank-via-api/issues). Для этого нужно нажать на кнопку New issue, указать в заголовке краткое описание проблемы, а потом описать проблему полностью, при необходимости, приложив скриншоты.
- Отправить свой вопрос по почте на адрес:
  - [EBoldina@alfabank.ru](mailto:EBoldina@alfabank.ru) (Екатерина Болдина)
  - в копию (обязательно):
    - [AKopyltsova@alfabank.ru](mailto:AKopyltsova@alfabank.ru) (Анна Копыльцова)
    - [alfa-link@alfabank.ru](mailto:alfa-link@alfabank.ru)
