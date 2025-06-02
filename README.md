# Домашнее задание к занятию «Вычислительные мощности. Балансировщики нагрузки»  

### Подготовка к выполнению задания

1. Домашнее задание состоит из обязательной части, которую нужно выполнить на провайдере Yandex Cloud, и дополнительной части в AWS (выполняется по желанию). 
2. Все домашние задания в блоке 15 связаны друг с другом и в конце представляют пример законченной инфраструктуры.  
3. Все задания нужно выполнить с помощью Terraform. Результатом выполненного домашнего задания будет код в репозитории. 
4. Перед началом работы настройте доступ к облачным ресурсам из Terraform, используя материалы прошлых лекций и домашних заданий.

---
## Задание 1. Yandex Cloud 

**Что нужно сделать**

1. Создать бакет Object Storage и разместить в нём файл с картинкой:

 - Создать бакет в Object Storage с произвольным именем (например, _имя_студента_дата_).
 - Положить в бакет файл с картинкой.
 - Сделать файл доступным из интернета.
 
2. Создать группу ВМ в public подсети фиксированного размера с шаблоном LAMP и веб-страницей, содержащей ссылку на картинку из бакета:

 - Создать Instance Group с тремя ВМ и шаблоном LAMP. Для LAMP рекомендуется использовать `image_id = fd827b91d99psvq5fjit`.
 - Для создания стартовой веб-страницы рекомендуется использовать раздел `user_data` в [meta_data](https://cloud.yandex.ru/docs/compute/concepts/vm-metadata).
 - Разместить в стартовой веб-странице шаблонной ВМ ссылку на картинку из бакета.
 - Настроить проверку состояния ВМ.
 
3. Подключить группу к сетевому балансировщику:

 - Создать сетевой балансировщик.
 - Проверить работоспособность, удалив одну или несколько ВМ.
4. (дополнительно)* Создать Application Load Balancer с использованием Instance group и проверкой состояния.

Полезные документы:

- [Compute instance group](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/compute_instance_group).
- [Network Load Balancer](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_network_load_balancer).
- [Группа ВМ с сетевым балансировщиком](https://cloud.yandex.ru/docs/compute/operations/instance-groups/create-with-balancer).


## Решение
1. Создать бакет Object Storage и разместить в нём файл с картинкой:

 - Создать бакет в Object Storage с произвольным именем (например, _имя_студента_дата_).
 - Положить в бакет файл с картинкой.
 - Сделать файл доступным из интернета.

bucket.tf

```terraform
resource "yandex_storage_bucket" "images" {
  bucket    = local.bucket_name
  folder_id = var.yc_folder_id
  anonymous_access_flags {
    read        = true
    list        = true
    config_read = true
  }
}

resource "yandex_storage_object" "image" {
  bucket     = local.bucket_name
  key        = "rick1.png"
  source     = "./images/rick1.png"
  depends_on = [yandex_storage_bucket.images]
}
```
[alt text](https://github.com/VN351/clopro-02/raw/main/images/1-1.jpg)

[alt text](https://github.com/VN351/clopro-02/raw/main/images/1-2.jpg)

2. Создать группу ВМ в public подсети фиксированного размера с шаблоном LAMP и веб-страницей, содержащей ссылку на картинку из бакета:

 - Создать Instance Group с тремя ВМ и шаблоном LAMP. Для LAMP рекомендуется использовать `image_id = fd827b91d99psvq5fjit`.
 - Для создания стартовой веб-страницы рекомендуется использовать раздел `user_data` в [meta_data](https://cloud.yandex.ru/docs/compute/concepts/vm-metadata).
 - Разместить в стартовой веб-странице шаблонной ВМ ссылку на картинку из бакета.
 - Настроить проверку состояния ВМ.

network.tf

```terraform
resource "yandex_vpc_network" "vpc" {
  name = var.vpc_net_name
}

resource "yandex_vpc_subnet" "public" {
  name           = var.subnet_name_public
  zone           = var.yc_zone
  network_id     = yandex_vpc_network.vpc.id
  v4_cidr_blocks = var.public_cidr
}

resource "yandex_vpc_security_group" "lamp_sg" {
  name        = "lamp-security-group"
  network_id  = yandex_vpc_network.vpc.id

  ingress {
    protocol       = "TCP"
    description    = "HTTP"
    port           = 80
    v4_cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    protocol       = "ANY"
    description    = "Outgoing traffic"
    v4_cidr_blocks = ["0.0.0.0/0"]
  }
}
```

lamp.tf

```terraform
resource "yandex_compute_instance_group" "lamp_group" {
  name               = "lamp-group"
  service_account_id = var.service

  instance_template {
    platform_id = var.lamp_group_compute_instance[0].platform_id
    resources {
      cores  = var.lamp_group_compute_instance[0].cores
      core_fraction = var.lamp_group_compute_instance[0].core_fraction
      memory = var.lamp_group_compute_instance[0].memory
    }
   
    boot_disk {
      initialize_params {
        image_id = var.lamp_group_compute_instance[0].image_id
      }
    }
    
    network_interface {
      network_id         = yandex_vpc_network.vpc.id
      subnet_ids         = [yandex_vpc_subnet.public.id]
      security_group_ids = [yandex_vpc_security_group.lamp_sg.id]
      nat                = true
    }

    metadata = local.metadata
  }

  scale_policy {
    fixed_scale {
      size = var.scale_count
    }
  }

  allocation_policy {
    zones = [var.yc_zone]
  }


  deploy_policy {
    max_unavailable = 1
    max_expansion   = 0
  }

  health_check {
    interval = 30
    timeout  = 5
    http_options {
      port = 80
      path = "/"
    }
  }
    load_balancer {
        target_group_name = "lamp-group"
    }
}
```

index.html.tmpl

```tmpl
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>My LAMP Website</title>
    <style>
      body {
        background: #f9f9f9;
        font-family: Arial, sans-serif;
      }
      .container {
        max-width: 600px;
        margin: 40px auto;
        padding: 24px;
        background: #fff;
        border-radius: 12px;
        box-shadow: 0 2px 12px rgba(0,0,0,0.08);
        text-align: center;
      }
      img {
        margin-top: 20px;
        max-width: 100%;
        height: auto;
        border-radius: 8px;
        background: #eee;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <h1>Welcome to my LAMP Website!</h1>
      <img src="${image_url}" alt="My Image">
      <p>Homework assignment “Computing Power. Load Balancers”</p>
      <p>Nevzorov V.V.</p>
    </div>
  </body>
</html>
```

user_data.yml.tmpl

```tmpl
#cloud-config
write_files:
  - path: /var/www/html/index.html
    permissions: '0644'
    owner: root:root
    content: |
      ${indent(10, index_html)}
runcmd:
  - systemctl restart apache2 || systemctl restart httpd
ssh_authorized_keys:
  - ${ssh_key}
```
[alt text](https://github.com/VN351/clopro-02/raw/main/images/1-3.jpg)

[alt text](https://github.com/VN351/clopro-02/raw/main/images/1-4.jpg)

[alt text](https://github.com/VN351/clopro-02/raw/main/images/1-5.jpg)

[alt text](https://github.com/VN351/clopro-02/raw/main/images/1-6.jpg)

[alt text](https://github.com/VN351/clopro-02/raw/main/images/1-7.jpg)


3. Подключить группу к сетевому балансировщику:

 - Создать сетевой балансировщик.
 - Проверить работоспособность, удалив одну или несколько ВМ.


```terraform
resource "yandex_lb_network_load_balancer" "load-balancer-nv" {
  name = "load-balancer-nv"

  listener {
    name = "http-listener"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_compute_instance_group.lamp_group.load_balancer.0.target_group_id

    healthcheck {
      name = "http-healthcheck"
      http_options {
        port = 80
        path = "/index.html"
      }
    }
  }
}
```
остальные файлы необходимые для работы

locals.tf

```terraform
locals {
  current_timestamp = timestamp()
  formatted_date    = formatdate("DD-MM-YYYY", local.current_timestamp)
  bucket_name       = "nevzorov-${local.formatted_date}"
  image_url         = "https://storage.yandexcloud.net/${yandex_storage_bucket.images.bucket}/${yandex_storage_object.image.key}"
  index_html        = templatefile("${path.module}/index.html.tmpl", {
    image_url = local.image_url
  })

  user_data = templatefile("${path.module}/user_data.yml.tmpl", {
    index_html = local.index_html
    ssh_key    = file(var.vms_ssh_root_key)
  })

  metadata = {
    user-data            = local.user_data
    serial-port-enable   = "1"
    ssh-keys             = "${var.ssh_username}:${file(var.vms_ssh_root_key)}"
  }
}
```

variables.tf

```terraform
variable "yc_token" {
  description = "Yandex Cloud OAuth token"
  type        = string
}

variable "yc_cloud_id" {
  description = "Yandex Cloud ID"
  type        = string
}

variable "yc_folder_id" {
  description = "Yandex Cloud Folder ID"
  type        = string
}

variable "subnet_id" {
  description = "Subnet ID"
  type        = string
}

variable "yc_zone" {
  description = "Yandex Cloud zone"
  type        = string
  default     = "ru-central1-a"
}

variable "service" {
      description = "Service"
      type        = string
}

variable "vpc_net_name" {
  type        = string
  default     = "subnet-nevzorov"
  description = "VPC subnet name"
}

variable "subnet_name_public" {
  type        = string
  default     = "subnet-nevzorov"
  description = "VPC subnet name"
}

variable "public_cidr" {
  type        = list(string)
  default     = ["192.168.10.0/24"]
  description = "CIDR"
}

variable "lamp_group_compute_instance" {
  type = list(object({
    name          = string
    cores         = number
    memory        = number
    core_fraction = number
    disk_size     = number
    disk_type     = string
    preemptible   = bool
    nat           = bool
    default_zone  = string
    platform_id   = string
    image_id      = string
  }))
  default = [{
      name          = "lamp-group"
      cores         = 2
      memory        = 1
      core_fraction = 20
      disk_size     = 10
      disk_type     = "network-hdd"
      preemptible   = true
      nat           = true
      default_zone  = "ru-central1-a"
      platform_id   = "standard-v3"
      image_id      = "fd827b91d99psvq5fjit"
            }]
  }

variable "scale_count" {
  type = number
  default = 3
  description = "Scale count"
}

variable "vms_ssh_root_key" {
  description = "Путь к публичному SSH ключу"
  type        = string
  default     = "/home/vlad/.ssh/id_ed25519.pub"
}

variable "ssh_username" {
  description = "Имя пользователя для SSH ключей"
  type        = string
  default     = "vlad"
}
```
[alt text](https://github.com/VN351/clopro-02/raw/main/images/1-8.jpg)

[alt text](https://github.com/VN351/clopro-02/raw/main/images/1-9.jpg)

[alt text](https://github.com/VN351/clopro-02/raw/main/images/1-10.jpg)

[alt text](https://github.com/VN351/clopro-02/raw/main/images/1-11.jpg)

---
## Задание 2*. AWS (задание со звёздочкой)

Это необязательное задание. Его выполнение не влияет на получение зачёта по домашней работе.

**Что нужно сделать**

Используя конфигурации, выполненные в домашнем задании из предыдущего занятия, добавить к Production like сети Autoscaling group из трёх EC2-инстансов с  автоматической установкой веб-сервера в private домен.

1. Создать бакет S3 и разместить в нём файл с картинкой:

 - Создать бакет в S3 с произвольным именем (например, _имя_студента_дата_).
 - Положить в бакет файл с картинкой.
 - Сделать доступным из интернета.
2. Сделать Launch configurations с использованием bootstrap-скрипта с созданием веб-страницы, на которой будет ссылка на картинку в S3. 
3. Загрузить три ЕС2-инстанса и настроить LB с помощью Autoscaling Group.

Resource Terraform:

- [S3 bucket](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket)
- [Launch Template](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/launch_template).
- [Autoscaling group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/autoscaling_group).
- [Launch configuration](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/launch_configuration).

Пример bootstrap-скрипта:

```
#!/bin/bash
yum install httpd -y
service httpd start
chkconfig httpd on
cd /var/www/html
echo "<html><h1>My cool web-server</h1></html>" > index.html
```
### Правила приёма работы

Домашняя работа оформляется в своём Git репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
