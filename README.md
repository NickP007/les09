## Тиждень 9

### Задача 1

#### Попередні вимоги:

- встановлені утиліти **terraform**, **flux**, **sops** та **age**
- доступ до облікового запису GitHub та згенерований персональний токен з дозолами на читання/запис/видалення репозиторію

#### Підготовка до розгортання інфраструктури

1.1 Використаємо існуючий персональний токен з дозолами на читання/запис/видалення репозиторію до облікового запису GitHub, за необхідності згенеруємо його.

1.1.1 Генерація персонального токену:

- перейдемо до облікового запису GitHub
- перейдемо до сторінки генерації токенів, натиснувши меню Settings -> Developer settings -> Personal access tokens -> Tokens (classic) -> Generate new token
- вкажемо назву токену в полі *Note*, та виберемо пункти *Full control of private repositories*, *Delete repositories*
- натиснемо Generate Token, зкопіюємо отриманий токен та збережемо в надійному місці

1.2 Отримаємо інфраструктурний код для розгортання за допомогою **terraform**:

  ```bash
  git clone --branch tf_kind_auth --single-branch https://github.com/NickP007/les07 tf_kind
  cd tf_kind
  ```

#### Розгортання інфраструктури

2.1 Відредагуємо файл vars.auto.tfvars, вказавши необхідні значення у відповідності до конфігурації

2.2 Імпортуємо персональний токен GitHub, отриманий в п.1.1:

  ```bash
  export TF_VAR_GITHUB_OWNER=${YOUR_GITHUB_ACCOUNT}
  export TF_VAR_GITHUB_TOKEN=${YOUR_GITHUB_ACCESS_TOKEN}
  ```

2.3 Розгорнемо Kubernetes Cluster з встановленою системою **flux** за допомогою **terraform**:

  ```bash
  terraform init
  terraform apply
  ```

2.4 Зробимо клонування репозиторію з налаштуваннями кластера GitOps Flux:

> Назва репозиторію міститься у файлі vars.auto.tfvars, значення змінної ${FLUX_GITHUB_REPO}. За замовчуванням 'flux-gitops-kind'

  ```bash
  git clone https://github.com/${TF_VAR_GITHUB_OWNER}/${FLUX_GITHUB_REPO}
  cd ${FLUX_GITHUB_REPO}
  ```

2.5 Пропатчимо **flux** для роботи з **sops** та ключами, згенерованими за допомоги **age**:

  ```bash
  git remote add flux_sops https://github.com/NickP007/les07
  git fetch flux_sops flux_sops_age
  git merge --allow-unrelated-histories --strategy-option theirs -m "Merge branch 'flux_sops_age' into main" flux_sops/flux_sops_age
  git push
  ```

#### Розгортання кластера демонстраційного застосунку з моніторінговим стеком

В якості демонстраційного застосунку використаємо наступну модель:

- "Сторонній" веб-сервер та базу данних розгорнемо із застосунку Grafana TNS (The New Stack observability app) без використання 'loadgen'
- Навантаження будемо генерувати за допомогою telegrm-боту. Для цього вже існуючому telegrm-боту 'kbot' (<https://github.com/NickP007/kbot>) додамо команду /get, що буде відсилати два запити (GET і PUSH) з рандомною затримкою на веб-сервер TNS з імплементацією traceID
- Метрики та трейси будемо відсилати на OpenTelemetry Collector за допомогою пакунку 'go.opentelemetry.io/otel' (<https://github.com/open-telemetry/opentelemetry-go>), що додамо до коду telegram-боту. Адреса серверу метрик буде визначена змінною оточення 'METRICS_HOST', адреса серверу трейсів - 'TRACES_HOST'.
- З огляду на те, що застосунок Grafana TNS для збіру трейсів використовує бібліотеку 'opentracing' (<https://github.com/opentracing/opentracing-go>), в код telegram-боту додамо пакунок 'go.opentelemetry.io/otel/bridge/opentracing' для сумісності обробки наскрізних трейсів.

3.1 Додамо файли декларацій демонстраційного застосунку та моніторінгового стеку до репозитроію GitOps Flux:

  ```bash
  git remote add cl_mon https://github.com/NickP007/les09
  git fetch cl_mon main
  git merge --allow-unrelated-histories -m "Merge branch 'cl_mon' into main" cl_mon/main
  ```

3.2 Згенеруємо ключ для шифрування секрету за допомогою **age**:

  ```bash
  mkdir -p $HOME/.config/sops/age
  age-keygen -o $HOME/.config/sops/age/keys.txt
  kubectl -n flux-system create secret generic sops-age --from-file=keys.agekey=$HOME/.config/sops/age/keys.txt
  ```

3.3 Згенеруємо маніфест секрету та зашифруємо його за допомогою створеного ключа п.3.2:

  ```bash
  read -s TELE_TOKEN
  kubectl -n demo create secret generic secret --from-literal=token=$TELE_TOKEN --dry-run=client -o yaml > clusters/demo/secret.yaml
  sops -e -i -age $(grep 'public key' $HOME/.config/sops/age/keys.txt | cut -d" " -f4) --encrypted-regex '^(token)$' clusters/demo/secret.yaml
  ```

3.4 Додамо файл секрету до репозиторію. Зробимо коміт та відправимо налаштування кластеру на сервер Github:

  ```bash
  git add clusters/demo/secret.yaml
  git commit -s -m "Add secret to 'demo' cluster"
  git push origin main
  ```

Через де-який час за допомогою системи **flux** буде розгорнуто демонстраційний застосунок з моніторінговим стеком у локальному dev-середовищі.

#### Перевірка роботи демонстраційного застосунку та взаємодії з моніторінговим стеком

4.1 Робота демонстраційного застосунку та взаємодії з моніторінговим стеком наведена у відео 4.1

![Відео 4.1.](docs/img/Monitoring_CC.gif)
