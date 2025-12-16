# Інструкція з розгортання та перевірки DaemonSet і CronJob

Цей документ пояснює, як розгорнути надані маніфести `daemonset.yml` та `cronjob.yml` у Kubernetes-кластері й як перевірити їхню роботу. Команди наведено для Windows `cmd.exe` (у PowerShell або на Unix-подібних системах потрібно трохи змінити синтаксис шляхів).

Вимоги

- Наявність налаштованого `kubectl`, спрямованого на ваш кластер.
- Кластер має вміти резолвити сервіс `todoapp-service` у неймспейсі `todoapp` (ClusterIP або інший тип, доступний у кластері).

Файли, що використовуються

- `.infrastructure/daemonset.yml` — DaemonSet, який запускає `busyboxplus:curl` на кожному вузлі й опитує ClusterIP-сервіс кожні ~5 секунд.
- `.infrastructure/cronjob.yml` — CronJob, який виконується кожні 4 хвилини і робить запит до `/api/health` сервісу.

Крок 1 — Створити неймспейс `mateapp`

Запустіть:

```cmd
kubectl create namespace mateapp
```

Якщо неймспейс уже існує, отримаєте повідомлення про помилку — це нормально.

Крок 2 — Розгорнути DaemonSet та CronJob

З кореня репозиторію застосуйте маніфести:

```cmd
kubectl apply -f .\.infrastructure\daemonset.yml
kubectl apply -f .\.infrastructure\cronjob.yml
```

Крок 3 — Перевірка DaemonSet

Переглянути DaemonSet у неймспейсі `mateapp`:

```cmd
kubectl -n mateapp get daemonset
```

Переглянути Pod-и, створені DaemonSet (міксуйте по лейблу `app=todoapp`):

```cmd
kubectl -n mateapp get pods -l app=todoapp -o wide
```

Подивитись логи одного з Pod-ів (замініть <pod-name> на реальне ім'я):

```cmd
kubectl -n mateapp logs <pod-name>
```

Оскільки контейнер виконує нескінченний цикл, логи показуватимуть результати `curl` кожні ~5 секунд (якщо відповідь сервісу містить тіло). Для стрімінгу логів у реальному часі:

```cmd
kubectl -n mateapp logs -f <pod-name>
```

Якщо Pod-и DaemonSet не запускаються, опишіть Pod для діагностики подій:

```cmd
kubectl -n mateapp describe pod <pod-name>
```

Крок 4 — Перевірка CronJob

Список CronJob-ів у неймспейсі `mateapp`:

```cmd
kubectl -n mateapp get cronjob
```

Переглянути Job-и, створені CronJob-ом:

```cmd
kubectl -n mateapp get jobs --sort-by=.metadata.creationTimestamp
```

Щоб знайти Pod, який створив Job, можна відфільтрувати Pod-и за віком або за префіксом імені Job-а. Наприклад, покажіть усі Pod-и та знайдіть потрібний:

```cmd
kubectl -n mateapp get pods
```

Потім подивіться логи конкретного Pod-а Job-а:

```cmd
kubectl -n mateapp logs <job-pod>
```

Зверніть увагу: CronJob налаштовано на виконання кожні 4 хвилини, тож зачекайте кілька хвилин після застосування, щоб з'явилися перші Job-и.

Крок 5 — Перевірка параметрів історії CronJob

Щоб переглянути налаштування `successfulJobsHistoryLimit` та `failedJobsHistoryLimit`, опишіть CronJob:

```cmd
kubectl -n mateapp describe cronjob cronjob-busybox-curl
```

У виводі знайдіть відповідні поля. Також перевірте список Job-ів, чи відповідає кількість збережених виконаних/збитих Job-ів встановленим лімітам.

Поради з налагодження

- Якщо `curl` видає помилки DNS (не знаходить домен), перевірте правильність імені сервісу — у маніфестах використано `todoapp-service.todoapp.svc.cluster.local`.
- Якщо Pod-и мають помилки під час завантаження образу (`ImagePullBackOff`), переконайтесь, що образ `ikulyk404/busyboxplus:curl` доступний з вашого кластера.
- Якщо CronJob не створює Job-ів, перевірте часовий пояс кластера та поле `schedule` у маніфесті.

Очікувані логи

- Pod-и DaemonSet: періодичні записи від `curl` приблизно кожні 5 секунд. Якщо сервіс повертає HTTP 200 з тілом — воно буде у логах; якщо тіло порожнє, `curl -s` нічого не виведе, але код повернення може бути 0.
- Pod-и Job-ів (CronJob): одноразовий виклик `/api/health` кожного запуску; у логах має бути результат цього виклику.

Примітки

- Маніфести використовують неймспейс `mateapp` та DNS сервісу `todoapp-service.todoapp.svc.cluster.local`. За потреби змініть ці значення під ваші імена в кластері.
- У CronJob встановлено `concurrencyPolicy: Allow`, тобто нові Job-и можуть запускатися, навіть якщо попередні ще працюють.

Якщо потрібно — можу:

- Підправити DNS-ім'я сервісу у маніфестах під ваш кластер (якщо вкажете правильне ім'я).
- Додати додаткові лейбли або аннотації для зручнішої фільтрації Pod-ів/Job-ів.
- Показати приклад очікуваного виводу логів (наприклад, HTTP 200 / JSON), якщо надасте приклад відповіді від `/api/health`.
