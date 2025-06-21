# Лабораторная работа: Развёртывание веб-сервера в Yandex Cloud с помощью Terraform

##  Цель работы
Освоить принципы инфраструктуры как кода (IaC) с использованием Terraform и Yandex Cloud: создать виртуальную машину, развернуть на ней веб-сервер (Nginx) в Docker, используя автоматическое конфигурирование.

##  Используемые технологии

- **Terraform**
- **Yandex.Cloud (Compute, VPC)**
- **Docker**
- **Nginx**
- **Bash**

##  Структура проекта

terraform-yc-lab/
├── main.tf # Основные ресурсы
├── variables.tf # Описание переменных
├── terraform.tfvars # Значения переменных 
├── outputs.tf # Вывод IP-адреса
├── metadata.sh # Скрипт инициализации на ВМ
└── static/ 

## Теория 

Terraform - это инструмент инфраструктуры как кода (IaC, Infrastructure as Code) от компании HashiCorp, который позволяет автоматизировать создание, изменение и управление облачными и локальными ресурсами с помощью декларативных конфигурационных файлов.
Работает с разными провайдерами, я буду использовать Yandex Cloud. 

# 0. Скачать Terrafrom и yc CLI, ssh

Terrafrom is not available in your region, поэтому включаем впн, пару минут и оп:

![image](https://github.com/user-attachments/assets/2dc79a12-faff-4a01-8fec-b1eca246eb5f)

С YaCloud такой проблемы нет 

![image](https://github.com/user-attachments/assets/051ebe75-4886-4ea8-9ce4-1ecf025696ee)

Теперь создадим публичный ssh ключ `ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa` для доступа к VM. 

Просмотреть наши важные штуки:

```
# Получить токен
yc config get token

# Получить cloud_id
yc resource-manager cloud list

# Получить folder_id
yc resource-manager folder list
```

# 1. Создание конфигурационных файлов Terraform

Создаём основной файл `main.tf`, в котором описываем:

- Подключение провайдера Yandex Cloud
- Создание VPC и подсети
- Создание виртуальной машины (VM)
- Подключение NAT для публичного IP
- Применение скрипта cloud-init через `metadata.sh`

```
provider "yandex" {
  token     = var.yc_token
  cloud_id  = var.cloud_id
  folder_id = var.folder_id
  zone      = var.zone
}

resource "yandex_vpc_network" "default" {
  name = "terraform-net"
}

resource "yandex_vpc_subnet" "default" {
  name           = "terraform-subnet"
  zone           = var.zone
  network_id     = yandex_vpc_network.default.id
  v4_cidr_blocks = ["10.0.0.0/24"]
}

resource "yandex_compute_instance" "webserver" {
  name        = "terraform-vm"
  platform_id = "standard-v1"
  zone        = var.zone

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = var.image_id
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.default.id
    nat       = true
  }

  metadata = {
    ssh-keys        = "ubuntu:${file("~/.ssh/id_rsa.pub")}"
    user-data       = file("metadata.sh")
  }
}

```

В этом файле будет описано всё: облачная сеть, подсеть, ВМ и тд. Также указываем количество ресурсов, которые нам необходимы. 

После создаем `variables.tf` и `terraform.tfvars` для хранения переменных. Указываем наш токен, и id'ки, а также зону, в которой будет доступен наш сервис. 

![image](https://github.com/user-attachments/assets/62652f49-4fca-4290-9c4b-409924585950)

После подготавливаем скрипт metadata.sh, который будет запускать нашу ВМ:

```
#!/bin/bash
apt update
apt install -y docker.io
systemctl start docker
docker run -d -p 80:80 --name nginx nginx
```
В нем мы устанавливаем докер и уже в нем запускаем контейнер с nginx. Не забываем выдать права на исполнение файла. 

После создаем файл outputs.tf для вывода ip адреса:

```
output "external_ip" {
  value = yandex_compute_instance.webserver.network_interface[0].nat_ip_address
}
```
# 2. Инициализация и запуск

Для инициализации выполним `terraform init`, при этом стоит сказать, что так как в данный момент terraform registry заблокирован в рф, то придется локально установить и зарегестрировать провайдера. 

```
provider_installation {
  network_mirror {
    url = "https://terraform-mirror.yandexcloud.net/"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
```

После все заработает:

![image](https://github.com/user-attachments/assets/67326bea-8045-4fce-80df-5dfca8f9d4e9)

Теперь просматриваем план: `terraform plan -var-file="terraform.tfvars"`

![image](https://github.com/user-attachments/assets/3b87432d-c9c6-417b-b649-1a058973fee1)

И применяем его: `terraform apply -var-file="terraform.tfvars"`

После подтверждения принятия, Terraform создаст ресурсы. 

Переходим по ip и видим окно веб-сервера nginx!

![image](https://github.com/user-attachments/assets/23f1a53b-351d-40cc-85c6-d57ef9b0bd84)

В браузерной версии YCloud также видно, что все работает. 

![image](https://github.com/user-attachments/assets/3996fb72-1774-41be-a6f4-421c7e1d67f8)

После мне захотелось помимо Compute Cloud, VPC, также настроить Object Storage или же S3 Хранилище. 

Для этого дописываем в main.tf новый ресурс:

```
resource "yandex_storage_bucket" "lab_bucket" {
  bucket     = "lab-storage-${random_id.bucket_suffix.hex}" # Уникальное имя
  access_key = var.access_key
  secret_key = var.secret_key

  max_size           = 1073741824 # 1 ГБ
  default_storage_class = "STANDARD"
  force_destroy      = true

  anonymous_access_flags {
    read = false
    list = false
  }
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}

```
В файл с переменными дописываем наши access_key и secret_key, чтобы их получить нужно в Identity and Access Management создать сервисный аккаунт и создать новый статический ключ доступа. Не забываем выдать права админа!!

```
variable "access_key" {
  description = "Access key для S3 совместимого доступа"
  type        = string
}

variable "secret_key" {
  description = "Secret key для S3 совместимого доступа"
  type        = string
}

```

После apply'ем наш новый ресурс и все работает:

![image](https://github.com/user-attachments/assets/87fded07-b43e-4998-956c-38fe7e077f63)

На этом закончим. Резюмируя, хочется сказать что Terraform позволяет автоматизировать развертывание облачных ресурсов, сокращая время настройки, что хорошо показыает принцип IaC. 
Я смог и научился:
Установить и настроить Terraform для работы с Yandex Cloud. Создал конфигурационные файлы для декларативного описания инфраструктуры. Развернул виртуальную машину в Yandex Cloud с использованием Terraform. Настроил SSH-доступ для безопасного подключения к ВМ.

Спасибо за прочтение!





