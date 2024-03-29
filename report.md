# Отчёт о выполнении задачи "Дрон-инспектор"

- [Отчёт о выполнении задачи "Дрон-инспектор"](#отчёт-о-выполнении-задачи-дрон-инспектор)
  - [Постановка задачи](#постановка-задачи)
  - [Известные ограничения и вводные условия](#известные-ограничения-и-вводные-условия)
    - [Цели и Предположения Безопасности (ЦПБ)](#цели-и-предположения-безопасности-цпб)
  - [Описание системы](#описание-системы)
    - [Компоненты](#компоненты)
    

## Постановка задачи

Компания создаёт дрон для мониторинга трубопроводов, тянущихся на сотни километров в том числе по горам, лесам и пустыням. Поэтому нужна функция автономного выполнения задачи. 
При этом даже в горах летает гражданская авиация (те же вертолёты экстренной медицинской помощи), очень бы не хотелось, чтобы, например, дрон компании врезался в вертолёт с пациентами. Поэтому тяжёлые автономные дроны должны летать в контролируемом воздушном пространстве, координацию полётов в котором осуществляет система организации воздушного движения (ОрВД).

В рамках хакатона было предложено: 
- доработать предложенную архитектуру (см. далее) бортового обеспечения дронов-инспекторов с учётом целей безопасности
- декомпозировать систему и отделить критический для целей безопасности код
- в бортовом ПО нужно внедрить компонент "монитор безопасности" и реализовать контроль взаимодействия всех подсистем дрона
- доработать функциональный прототип 
- создать автоматизированные тесты, демонстрирующие работу механизмов защиты


## Известные ограничения и вводные условия
1. По условиям организаторов должна использоваться микросервисная архитектура и шина обмена сообщениями для реализации асинхронной работы сервисов.
2. Между собой сервисы ПЛК общаются через шину сообщений (message bus), а всё "снаружи" принимают в виде REST запросов.
3. Физическая надёжность аппаратуры (ПЛК и всего остального производства) и периметра не учитывается.
4. Графический интерфейс для взаимодействия с пользователем не требуется, достаточно примеров REST запросов.
5. Персонал, обслуживающий систему, благонадёжен.


### Цели и Предположения Безопасности (ЦПБ)
Цели безопасности:

1. Выполняются только аутентичные задания на мониторинг 
2. Выполняются только авторизованные системой ОрВД задания 
3. Все манёвры выполняются согласно ограничениям в полётном задании (высота, полётная зона/эшелон) 
4. Только авторизованные получатели имеют доступ к сохранённым данным фото-видео фиксации 
5. В случае критического отказа дрон снижается со скоростью не более 1 м/с 
6. Для запроса авторизации вылета к системе ОрВД используется только аутентичный идентификатор дрона 
7. Только авторизованные получатели имеют доступ к оперативной информации 


Предположения безопасности:

1. Аутентичная система ОрВД благонадёжна
2. Аутентичные сотрудники благонадёжны и обладают необходимой квалификацией
3. Только авторизованные сотрудники управляют системами
4. Аутентичное полётное задание составлено так, что на всём маршруте дрон может совершить аварийную посадку без причинения неприемлемого ущерба заказчику и третьим лицам


### Описание системы

Изначально, в общем виде контекст работы выглядел следующим образом:

![Контекст](docs/images/drone-inspector_general.png)

В ходе решения она была детализирована до следующей:

![Решение](docs/images/drone-inspector_result.png)


### Компоненты

| Название | Назначение | Комментарий |
|----|----|----|
|*Служба планирования полетов (СПП)* | Имитатор сервиса распределения задач по дронам. Позволяет согласовывать полетное задание с ОРВД, отправлять  дронов на задание, задавать режимы полёта. Получает данные от дрона, в том числе телеметрию. | Доверенный. Согласно условиям задания. |
|*Система организации воздуного движения (ОРВД)* | Имитатор центральной государственной службы управления движением дронов в общем воздушном пространстве. Подтверждает полетное задание,получает телеметрическую информацию о местоположении каждого дрона, отправляет экстренные команды дронам. | Доверенный. Согласно условиям задания. |
|*1. Имитатор Блока приема данных* | Приемная антенна и приемник выполнят функции по приему и оцифровке сигналов от ОРВД и СПП. Далее информация передается на CCU для дешифровки и обработки команд. | Недоверенный. Для передачи данных используются радиочастоты, что является уязвивым элементом системы. |
|*2. Имитатор Блока передачи данных.* | Передатчик и передающая антенна выполняют задачи по передаче закодированного CCU сигнала. | Недоверенный. Для передачи данных используются радиочастоты, что является уязвивым элементом системы.|
|*3. Имитатор Центрального блока управления (CCU)* | Осуществляет полный контроль над БПЛА. Выполняет шифрование и дешифрование данных. | Доверенный. Защита основана на системе выделенного электропитания и отдельного защищенного модуля памяти. Функционирование основано на дополнительном шифровании. |
|*4. Имитатор Акумулятора для CCU и Memory* | Выделенный защищенный источник электропитания для CCU, защищенной памяти для CCU и других блоков "зеленой зоны". | Доверенный, т.к. отделен от взаимодействия с блоками "вне зеленой зоны". |
|*5. Имитатор Аккумулятора для электронных систем БПЛА* | Специальный элемент защиты электронники БПЛА в качестве отдельно выделенного для этих целей аккумулятора. | Доверенный. Защита основана на отсутствии взаимодействия с силовыми аккумуляторами и дополнительной системе защиты от внешних воздействий. |
|*6. Имитатор Служебной памяти только для CCU (Memory)* | Специальный элемент защиты CCU в качестве отдельно выделенного блока внешней памяти. | Доверенный. Защита основана на отсутсвии внешних взаимодействий с другими блоками БПЛА. Питание обеспечивается защищенным Аккумулятором Блока 4. |
|*7. Имитатор Системы электропитания электроники БПЛА* | Дополнительный элемент защиты от внешних воздействий на электронные платы (блоки) по сети электропитания. | Доверенный. Защита обеспечивается системой гальванических развязок в шине передачи данных БПЛА. |
|*8. Имитатор Монитора безопасности (МБ)* | Блок мониторинга работоспособности всех систем БПЛА и реагирования на нештатные ситуации. Дублирует работу CCU в части безопасности БПЛА. В случае поломки CCU обеспечивает через шину передачи данных и Блок 22 аварийную поссадку и формирование сигнала для пеленга с координатами и кодом ошибки. | Доверенный. Защита обеспечивается выделенным источником электропитания и дополнительным внутренним шифрованием, аналогично CCU. |
|*9. Имитатор Системы навигации и контроля аутентичности полета (СНП)* | Функционирование СНП обеспечивается выделенным блоком памяти с полетным заданием блока 10, а также данными из блока 18 о текущем местоположении БПЛА. Результаты расчетов передаются в CCU и далее по шине в блок 22. | Доверенный. Защита основана на отдельном электропитании, отсутствии подключения к шине передачи данных (аналог Memory), шифрованием аналогично CCU, а также переводом в режим "блокировка на запись" после записи полетного задания в СПП. В полете изменить данные невозможно. |
|*10. Имитатор Служебной памяти для хранения полетного задания (СППЛ)* | Функции аналогичны блоку 6 "Memory". Информация хранится для функционирования блока 9 СНП. Копия полетного задания также хранится в Memory. | Доверенный. Защита основана на отдельном электропитании, отсутствии подключения к шине передачи данных (аналог Memory). Блокировки функции "запись". |
|*11. Имитатор Системы предотвращения столкновений (СПС)* | Функционирование СПС основано на контроле местополжения БПЛА в заданном коридоре и в заданных координатах, а также сканированием и контролем окружающего воздушного пространства. В случае возможных угроз информация передается в CCU и в блок 8 "МБ". |Доверенный. Защита основана на отдельном электропитании, предположении об отсутствии целенаправленных внешних помех, данные от радиолокаторов (лидаров) получаются напрямую по шине передачи данных. |
|*12. Имитатор Шины передачи данных* | Обеспечивает передачу данных между системами БПЛА по проводам и по оптическим линиям связи. | Недоверенный. Отсутствует система кодирования передачи данных между электронным компонентами БПЛА. Частично защита предусмотрена наличием гальванических развязок. |
|*13. Имитатор Системы хранения основных данных о полете* | Осуществляет запись/перезапись данных на SSD по командам из CCU. | Недоверенный. Взаимодействует с другими недостоверными блоками, не имеет шифрования для защиты от внешнего воздействия. |
|*14. Имитатор Процессинга предобработки и анализа видео* | Полученные из блока 15 видеопотоки обрабатываются обученной моделью ИИ. Итоговые данные передаются в CCU. При необходимости видеоданные частично фиксируются в памяти блока 13. | Недоверенный. Зашита не предусмотрена из-за сложности реализации при многопотоковом видеоряде. |
|*15. Имитатор Системы агрегации и анализа потокового видео без предобработки* | Блок 15 получает видеопотоки непосредственно с видеокамер. Производит частичную грубую фильтрацию с целью дальнейшей передачи данных в блок 14 для полноценной их обработки ИИ. | Недоверенный. Зашита не предусмотрена из-за сложности реализации при многопотоковом видеоряде. |
|*16. Имитатор Контроллера режимов работы видеокамер* | Выполняет функции контроля работоспособности видеокамер, управления режимами видеосъемки. Управляется CCU, а также по командам CCU выполняет целевые запросы от блоков 8 "МБ", 9 "СНП" и 11 "СПС". | Недоверенный. Так как сегмент "видео" не защищен, то контроллер отдельно защищать нецелесообразно. |
|*17. Имитатор Видекамер* | Формируются видеопотоки в соответствии с командами из блока 16. | Недоверенный. Так как блок 16 является недоверенным, то видепотоки от видеокамер также считаются недоверенными. |
|*18. Имитатор Комплексирование для задач навигации/телеметрии* | Решаются задачи минимизации стороннего вмешательства в систему позиционирования БПЛА. В случае значительного расхождения в данных (особенно от спутников) выдается предупреждающий сисгнал в CCU и в блок 8 "МБ". | Повышающий доверие. За счет обработки данных от трех источников и обратной связи от блока 9. |
|*19. Имитатор Инерциальной навигационной системы* | Автономный модуль, то есть не зависящий от внешних источников информации, для определения положения и ориентации БПЛА в пространстве. | Недоверенный. Возможны ошибочные данные. |
|*20. Имитатор Системы определения координат по треангуляции* | Дополнителльная система навигации как при отсутствии спутниковой связи, так и при ее наличии для подтверждения получаемых данных. | Недоверенный. Не гарантируется постоянная работоспособность. |
|*21. Имитатор Системы спутниковой связи и определения координат GPS и Глонасс* | Основной модуль определения координат в безлюдных местностях, где нет сотовой связи и затруднено использование триангуляционного модуля 20. | Недоверенный. Возможны внешние воздействия с целью передачи на БПЛА фальсифицированных координат. |
|*22. Имитатор Системы управления авионикой и выполнения плавной аварийной посадки* | Блок получает от CCU команды по требуемым перемещениям БПЛА, рассчитывает параметры движения и управляет двигжителями и авионикой. В случае получения команды "аварийная посадка" от CCU или от блока 8 выволняется прописанная последовательность действий для приземления БПЛА. | Недоверенный. В случае выявления угроз воздействия на авионику БПЛА функционирование блока 22 блокируется и управление переходит к защищенному блоку 8. |
|*23. Имитатор Контроллера управления режимами работы ротором/роторами* | Выдает команды авионике для изменения режимов работы и позиционирования БПЛА. В случае неисправности или угрозы воздействия функции дублируются блоком 8.| Недоверенный. Так как блок 22 недоверенный.|
|*24. Имитатор Ротора/роторов* | Отрабатываются команды блока 23 "Контроллер управления". | Недоверенный. Режимы работы могут быть искажены блоком 23. |
|*25. Имитатор Контроллера управления расходом энергии/топлива* | Принеобходимости управляет интегрированными силовыми установками. Сообщает в CCU по шине о текущем состоянии энергопитания. При критических значениях формирует сигнал опасности. Информация об опсаности передается в CCU и в блок 8 "Мониторинг безопасности". | Недоверенный, т.к. возможно вмещательство извне по шине обмена данными и через блок 22. |
|*26. Имитатор Датчиков контроля заряда/разряда силовых аккумуляторов* | Передают данные в блок 25 "Контроллер управления режимами". | Доверенный. Не подвержен внешнему воздействию. |
|*27. Имитатор Силовых аккумуляторов* | Силовые аккумуляторы обеспечивают электроэнергией только роторы. | - |
|*28. Имитатор Контроллера управления сканированием наружного пространства* | Управляет работой локационных систем БПЛА. Основное назначение - передача информации по шине данных в блок 11 с целью предотвращения столкновения. | Недоверенный. Защита при потоковом получении данных от локаторов затруднена. |
|*29. Имитатор Радиолокаторов* | Радиолокационный комплекс БПЛА выполняет задачи сканирования окружающего воздушного пространства. | Недоверенный. Работа может быть искажена блоком 28 "Контроллер управления радиолокаторами". |
|*30. Имитатор Датчиков расхода топлива для гибридных БПЛА* | Передают данные в блок 25 "Контроллер управления расходом топлива". | Доверенный. Не подвержен внешнему воздействию. |
|*31. Имитатор Топливные баки БПЛА* | При использовании гибридного БПЛА. | - |
