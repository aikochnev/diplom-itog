# Итоговый проект курса  "DevOps-инженер с нуля" - `Кочнев Андрей`

Учебный проект по развёртыванию контейнеризированного приложения в Yandex Cloud с использованием:

    Terraform;

    Ansible;

    Docker;

    Yandex Container Registry;

    GitHub Actions;

    SSH-деплой на виртуальные машины.

- Инфраструктура на Terraform: https://github.com/aikochnev/diplom-terraform

- Приложение на основе Docker: https://github.com/aikochnev/diplom-docker

- Дополнение на Ansible: https://github.com/aikochnev/diplom-ansible

Проект демонстрирует полный цикл доставки приложения:
```
Изменение исходного кода
↓
Git push в GitHub
↓
GitHub Actions
↓
Сборка Docker image
↓
Push image в Yandex Container Registry
↓
SSH-подключение к VM
↓
docker pull
↓
Запуск контейнера
```
---

## Архитектура проекта

В Yandex Cloud создаются:
```
Cloud Folder
└── VPC Network
    ├── Subnet web-b
    │   └── VM web-b
    └── Subnet web-d
        └── VM web-d

Container Registry
└── Repository my-app
```

Приложение разворачивается на двух виртуальных машинах: web-b и web-d. Обе VM получают Docker image из Yandex Container Registry и запускают одинаковый контейнер приложения.

Terraform отвечает за создание и управление инфраструктурой: VPC, подсетями, виртуальными машинами, Container Registry и правами сервисного аккаунта VM. Ansible отвечает за bootstrap CI-инфраструктуры и синхронизацию GitHub переменных. Такой подход разделяет инфраструктуру, служебную настройку CI и непосредственно application deployment.

Структура каталогов:
```
diplom/
├── ansible/
│   ├── ansible.cfg
│   ├── bootstrap.yml
│   ├── group_vars/
│   │   └── all.yml
│   ├── inventory/
│   │   └── prod.yml
│   └── README.md
├── docker/
│   ├── .github/
│   │   └── workflows/
│   │       └── ci-cd.yml
│   ├── compose.yaml
│   ├── Dockerfile
│   ├── index.html
│   └── README.md
├── keys/
│   └── diplom-ci-key.json
└── terraform/
    ├── backend.tf
    ├── cloud-init.yml
    ├── deploy-infrastructure.sh
    ├── iam.tf
    ├── main.tf
    ├── outputs.tf
    ├── personal.auto.tfvars
    ├── providers.tf
    ├── security.tf
    ├── variables.tf
    └── README.md
```
### Компоненты проекта

Terraform

Terraform создаёт:

    VPC-сеть;

    подсети;

    security group;

    две виртуальные машины;

    Yandex Container Registry;

    Container Repository;

    service account diplom-vm;

    роль container-registry.images.puller для diplom-vm.

Сервисный аккаунт diplom-vm используется виртуальными машинами для получения Docker image из Container Registry.

Terraform также использует удалённый S3-compatible backend в Yandex Object Storage. Это позволяет хранить Terraform state централизованно и использовать его после повторного запуска с локальной машины.

Ansible запускается локально на управляющей Ubuntu-машине: localhost

Он не заменяет Docker-деплой и не подключается к VM для каждого запуска приложения.

Ansible playbook выполняет bootstrap-операции:

    проверяет наличие yc;

    проверяет наличие terraform;

    проверяет наличие gh;

    проверяет наличие jq;

    проверяет существование service account diplom-ci;

    создаёт diplom-ci, если он отсутствует;

    проверяет роль container-registry.images.pusher;

    получает значения Terraform outputs;

    получает текущие GitHub Variables;

    обновляет только изменившиеся значения:

        YC_REGISTRY_ID;

        WEB_B_IP;

        WEB_D_IP.

Playbook является повторяемым: уже существующий diplom-ci повторно не создаётся, а уже назначенная IAM role повторно не добавляется. Ansible ориентирован на desired state и idempotency: повторный запуск должен приводить к тому же состоянию без лишних изменений.

Приложение упаковывается в Docker image на основе Dockerfile.

Основные файлы:
```
docker/
├── Dockerfile
├── compose.yaml
└── index.html
```
Dockerfile описывает образ приложения.

compose.yaml используется для локального запуска и проверки контейнера.

Пример локального запуска:

<img src = "img/dip-01.png" width = 100%>

После запуска приложение доступно по адресу:

http://localhost

GitHub Actions

GitHub Actions запускается после push в ветку main.

Workflow выполняет следующие этапы:

    получает исходный код;

    настраивает Docker Buildx;

    выполняет вход в Yandex Container Registry;

    собирает Docker image;

    публикует image в Container Registry;

    подключается к web-b по SSH;

    подключается к web-d по SSH;

    скачивает новый image;

    перезапускает контейнер.

Использование GitHub Actions для сборки и публикации Docker image соответствует стандартному CI/CD-подходу: commit запускает workflow, workflow собирает image и отправляет его в registry.

Настройка GitHub Secrets

В GitHub Repository должны быть настроены Secrets:

YC_CI_KEY_JSON
DEPLOY_SSH_KEY

YC_CI_KEY_JSON содержит authorized key сервисного аккаунта: diplom-ci

Этот аккаунт используется GitHub Actions для публикации Docker image в Yandex Container Registry.

diplom-ci создаётся и проверяется через Ansible, но authorized key не нужно пересоздавать при каждом запуске. Это позволяет сохранить стабильное значение YC_CI_KEY_JSON.
DEPLOY_SSH_KEY

DEPLOY_SSH_KEY содержит приватный SSH-ключ для подключения GitHub Actions к виртуальным машинам:

web-b
web-d

Публичный ключ должен быть добавлен в metadata VM или через cloud-init.
Настройка GitHub Variables

Ansible обновляет следующие Repository Variables:

YC_REGISTRY_ID
WEB_B_IP
WEB_D_IP

Пример актуальных значений после развёртывания:

YC_REGISTRY_ID = crpjuu6gmcfl07kln6pl
WEB_B_IP       = 111.88.152.203
WEB_D_IP       = 158.160.225.253

<img src = "img/dip-02.png" width = 100%>

Значения IP-адресов могут измениться после пересоздания виртуальных машин, поэтому они не должны быть жёстко зашиты в workflow.

# Запуск проекта

## Требования

Перед началом работы должны быть установлены:

- Git;
- Docker;
- Docker Compose;
- Terraform;
- Ansible;
- SSH-клиент.

## Клонирование репозиториев с кодом terraform, ansible и docker

Для начала необходимо скачать проекты из GitHub. Для этого склонируйте репозиторий на локальный компьютер или сервер:

git clone git@github.com:aikochnev/diplom-ansible
git clone git@github.com:aikochnev/diplom-docker
git clone git@github.com:aikochnev/diplom-terraform

## Авторизация в Yandex Cloud

Перед запуском Terraform необходимо настроить авторизацию в Yandex Cloud и доступ к backend.

Если переменные окружения уже настроены, этот шаг можно пропустить.

Для временной настройки переменных выполните:

export YC_TOKEN=$(yc iam create-token)
export YC_CLOUD_ID=$(yc config get cloud-id)
export YC_FOLDER_ID=$(yc config get folder-id)

Проверить значения можно командами:

echo "$YC_CLOUD_ID"
echo "$YC_FOLDER_ID"

## Настройка GitHub Secrets

Перед запуском GitHub Actions добавьте секретные переменные в репозиторий.

Откройте репозиторий созданый на основе git clone git@github.com:aikochnev/diplom-docker на GitHub и перейдите:

Settings → Secrets and variables → Actions → New repository secret

Создайте следующие Secrets:

Name	                        Значение
YC_CI_KEY_JSON	         JSON-ключ сервисного аккаунта Yandex Cloud
DEPLOY_SSH_KEY	         Приватный SSH-ключ для подключения к серверу (лучше создать новый)

В командной строке выполните:

yc iam key create \
  --service-account-id <SERVICE_ACCOUNT_ID> \
  --output-file yc-ci-key.json

<img src = "img/dip-05.png" width = 100%>

Скопировать и вставить содержимое файла yc-ci-key.json в secret.

Пример значения `DEPLOY_SSH_KEY`:

```
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

## Запуск Terraform

После настройки авторизации выполните:

cd ~/diplom/terraform
terraform init
terraform plan
terraform apply

После подтверждения Terraform создаст инфраструктуру.

<img src = "img/dip-03.png" width = 100%>

Список виртуальных машин, созданных terraform можн увидеть в соотвествующей вкладке Yandex Cloud.

<img src = "img/dip-04.png" width = 100%>

## Запуск Ansible bootstrap

После успешного terraform apply перейдите в каталог Ansible:

cd ~/diplom/ansible
ansible-playbook bootstrap.yml

<img src = "img/dip-02.png" width = 100%>

## CI/CD-деплой приложения

После настройки инфраструктуры обычный деплой выполняется через GitHub Actions.

Из каталога Docker-приложения:

cd ~/diplom/docker
nano index.html
git add .
git commit -m "Update application"
git push origin main

После успешного GitHub Actions deployment:

curl http://<WEB_B_IP>
curl http://<WEB_D_IP>

Или откройте в браузере:

http://<WEB_B_IP>
http://<WEB_D_IP>

Ожидаемый вывод:

<img src = "img/dip-06.png" width = 100%>

Проверка контейнера на VM:

docker ps
docker images

<img src = "img/dip-07.png" width = 100%>

Изменение версий приложения можно увидеть вот здесь:

Откройте репозиторий созданый на основе git clone git@github.com:aikochnev/diplom-docker на GitHub и перейдите на вкладку Actions

<img src = "img/dip-08.png" width = 100%>

<img src = "img/dip-09.png" width = 100%>

<img src = "img/dip-10.png" width = 100%>

<img src = "img/dip-11.png" width = 100%>

<img src = "img/dip-12.png" width = 100%>

# Работа после пересоздания инфраструктуры

Если необходимо полностью пересоздать инфраструктуру:

cd ~/diplom/terraform
terraform destroy
terraform apply

<img src = "img/dip-13.png" width = 100%>

После нового apply нужно обновить GitHub Variables:

cd ~/diplom/ansible
ansible-playbook bootstrap.yml

В результате Ansible получит новые IP-адреса и обновит:

WEB_B_IP
WEB_D_IP

<img src = "img/dip-14.png" width = 100%>

Проверка состояния проекта

cd ~/diplom/terraform
terraform plan

Ожидаемый результат при отсутствии изменений:

No changes. Your infrastructure matches the configuration.

<img src = "img/dip-15.png" width = 100%>

После проведения изменений инфраструктуры (Terraform и Ansible), актуальную версию деплоя можно увидеть вот здесь aikochnev/diplom-docker на GitHub и перейдите на вкладку Actions:

<img src = "img/dip-16.png" width = 100%>

<img src = "img/dip-17.png" width = 100%>

<img src = "img/dip-18.png" width = 100%>

<img src = "img/dip-19.png" width = 100%>

<img src = "img/dip-20.png" width = 100%>

<img src = "img/dip-21.png" width = 100%>