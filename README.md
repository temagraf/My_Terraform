# Домашнее задание "`«Основы Terraform. Yandex Cloud»`"

---

### Задание 1

В качестве ответа всегда полностью прикладывайте ваш terraform-код в git. Убедитесь что ваша версия Terraform ~>1.8.4

1) Изучите проект. В файле variables.tf объявлены переменные для Yandex provider.
2) Создайте сервисный аккаунт и ключ. service_account_key_file.
3) Сгенерируйте новый или используйте свой текущий ssh-ключ. Запишите его открытую(public) часть в переменную vms_ssh_public_root_key.
4) Инициализируйте проект, выполните код. Исправьте намеренно допущенные синтаксические ошибки. Ищите внимательно, посимвольно. Ответьте, в чём заключается их суть.
5) Подключитесь к консоли ВМ через ssh и выполните команду  curl ifconfig.me. Примечание: К OS ubuntu "out of a box, те из коробки" необходимо подключаться под пользователем ubuntu: "ssh ubuntu@vm_ip_address". Предварительно убедитесь, что ваш ключ добавлен в ssh-агент: eval $(ssh-agent) && ssh-add Вы познакомитесь с тем как при создании ВМ создать своего пользователя в блоке metadata в следующей лекции.;
6) Ответьте, как в процессе обучения могут пригодиться параметры preemptible = true и core_fraction=5 в параметрах ВМ.

В качестве решения приложите:

- скриншот ЛК Yandex Cloud с созданной ВМ, где видно внешний ip-адрес;
- скриншот консоли, curl должен отобразить тот же внешний ip-адрес;
- ответы на вопросы.

### Выполнения задания 1

1) Изучил проект.

2) Переименовал файл personal.auto.tfvars_example в personal.auto.tfvars, заполнил переменные.

3) Создал короткий ssh ключ используя ssh-keygen -t ed25519, записал его pub часть в переменную vms_ssh_root_key.

4) Инициализировал проект, выполнил код. Нашел ошибки в блоке resource "yandex_compute_instance" "platform" {.

Выявленные ошибки:

- В строке platform_id = "standart-v4" правильно писать standard
- Вместо версии v4 могут быть только v1, v2 и v3.
- Cores 1 не может быть, потомучто согласно документации минимальное количество виртуальных ядер процессора для всех платформ равно двум.

5)

![image.jpg](https://github.com/temagraf/My_Terraform/blob/main/1-2.png)


![image.jpg](https://github.com/temagraf/My_Terraform/blob/main/1.png)


![image.jpg](https://github.com/temagraf/My_Terraform/blob/main/1-3.png)

6) Параметры  preemptible = true - это прерываемая ВМ, т.е. работает не более 24 часов и может быть остановлена Compute Cloud в любой момент.
Параметр core_fraction = 5 - указывает базовую производительность ядра в процентах. Указывается для экономии ресурсов.

---

### Задание 2

1) Замените все хардкод-значения для ресурсов yandex_compute_image и yandex_compute_instance на отдельные переменные. К названиям переменных ВМ добавьте в начало префикс vm_web_ . Пример: vm_web_name.
2) Объявите нужные переменные в файле variables.tf, обязательно указывайте тип переменной. Заполните их default прежними значениями из main.tf.
3) Проверьте terraform plan. Изменений быть не должно.

### Выполнения задания 2


1) Заменил хардкод-значения для ресурсов yandex_compute_instance на переменные и добавил префикс vm_web_.

   Обновленный main.tf

```hcl
resource "yandex_vpc_network" "develop" {
  name = var.vpc_name
}

resource "yandex_vpc_subnet" "develop" {
  name           = var.vpc_name
  zone           = var.default_zone
  network_id     = yandex_vpc_network.develop.id
  v4_cidr_blocks = var.default_cidr
}

resource "yandex_compute_instance" "platform" {
  name        = var.vm_web_name
  platform_id = var.vm_web_platform_id
  zone        = var.vm_web_zone

  resources {
    core_fraction = var.vm_web_core_fraction
    cores         = var.vm_web_cores
    memory        = var.vm_web_memory
  }

  boot_disk {
    initialize_params {
      image_id = var.vm_web_image_id
      type     = var.vm_web_disk_type
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.develop.id
    nat       = var.vm_web_nat
  }

  metadata = {
    "serial-port-enable" = var.vm_web_serial_port_enable
    "ssh-keys"           = var.vm_web_ssh_keys
  }

  scheduling_policy {
    preemptible = var.vm_web_preemptible
  }
}

output "external_ip_address" {
  value = yandex_compute_instance.platform.network_interface.0.nat_ip_address
}

output "instance_id" {
  value = yandex_compute_instance.platform.id
}

```


Обновленный variables.tf

```hcl
variable "token" {
  type        = string
  description = "OAuth-token; https://cloud.yandex.ru/docs/iam/concepts/authorization/oauth-token"
}

variable "cloud_id" {
  type        = string
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/cloud/get-id"
}

variable "folder_id" {
  type        = string
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/folder/get-id"
}

variable "default_zone" {
  type        = string
  default     = "ru-central1-a"
  description = "https://cloud.yandex.ru/docs/overview/concepts/geo-scope"
}

variable "default_cidr" {
  type        = list(string)
  default     = ["10.0.1.0/24"]
  description = "https://cloud.yandex.ru/docs/vpc/operations/subnet-create"
}

variable "vpc_name" {
  type        = string
  default     = "develop"
  description = "VPC network & subnet name"
}

variable "vms_ssh_root_key" {
  type        = string
  description = "ssh-keygen -t ed25519"
}

variable "vm_web_name" {
  type        = string
  default     = "netology-develop-platform-web"
  description = "Name of the VM"
}

variable "vm_web_platform_id" {
  type        = string
  default     = "standard-v1"
  description = "Platform ID for the VM"
}

variable "vm_web_zone" {
  type        = string
  default     = "ru-central1-a"
  description = "Zone where the VM will be created"
}

variable "vm_web_core_fraction" {
  type        = number
  default     = 5
  description = "Core fraction for the VM"
}

variable "vm_web_cores" {
  type        = number
  default     = 2
  description = "Number of cores for the VM"
}

variable "vm_web_memory" {
  type        = number
  default     = 1
  description = "Memory size (in GB) for the VM"
}

variable "vm_web_image_id" {
  type        = string
  default     = "fd8t24r7o6m7fdvlp47l"
  description = "Image ID for the boot disk"
}

variable "vm_web_disk_type" {
  type        = string
  default     = "network-hdd"
  description = "Type of the boot disk"
}

variable "vm_web_nat" {
  type        = bool
  default     = true
  description = "Enable NAT for the VM"
}

variable "vm_web_serial_port_enable" {
  type        = string
  default     = "1"
  description = "Enable serial port for the VM"
}

variable "vm_web_ssh_keys" {
  type        = string
  default     = "ubuntu:ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICMLupjJ1DJ6oImS9OyvFFNenNk8/hiRrzkWIZ171DFa root@debian12-2"
  description = "SSH keys for the VM"
}

variable "vm_web_preemptible" {
  type        = bool
  default     = true
  description = "Enable preemptible flag for the VM"
}


```

![image.jpg](https://github.com/temagraf/My_Terraform/blob/main/2.png)


---

### Задание 3

1) Создайте в корне проекта файл 'vms_platform.tf' . Перенесите в него все переменные первой ВМ.
2) Скопируйте блок ресурса и создайте с его помощью вторую ВМ в файле main.tf: "netology-develop-platform-db" , cores  = 2, memory = 2, core_fraction = 20. Объявите её переменные с префиксом vm_db_ в том же файле ('vms_platform.tf'). ВМ должна работать в зоне "ru-central1-b"
3) Примените изменения.

### Выполнения задания 3


1) Создал в корне проекта файл 'vms_platform.tf' . Перенес в него все переменные первой ВМ.

2) Скопировал блок ресурса и создал с его помощью вторую ВМ в файле main.tf: "netology-develop-platform-db" , cores  = 2, memory = 2, core_fraction = 20. Объявите её переменные с префиксом vm_db_ в том же файле ('vms_platform.tf'). ВМ работает в зоне "ru-central1-b".

3) Применил изменения. Создал еще одну виртуальную машину.


![image.jpg](https://github.com/temagraf/My_Terraform/blob/main/3.1%20ДЗ.png)

![image.jpg](https://github.com/temagraf/My_Terraform/blob/main/3%20.png)


---

### Задание 4

1) Объявите в файле outputs.tf один output , содержащий: instance_name, external_ip, fqdn для каждой из ВМ в удобном лично для вас формате.(без хардкода!!!)
2) Примените изменения.
В качестве решения приложите вывод значений ip-адресов команды terraform output.

### Выполнения задания 4

Создание файла outputs.tf в корне проекта и добавление в него необходимый output.

outputs.tf

```hcl
output "vm_instances_info" {
  value = {
    vm_web = {
      instance_name = yandex_compute_instance.platform.name
      external_ip   = yandex_compute_instance.platform.network_interface.0.nat_ip_address
      fqdn          = yandex_compute_instance.platform.fqdn
    }
    vm_db = {
      instance_name = yandex_compute_instance.platform_db.name
      external_ip   = yandex_compute_instance.platform_db.network_interface.0.nat_ip_address
      fqdn          = yandex_compute_instance.platform_db.fqdn
    }
  }
  description = "Information about VM instances: name, external IP, and FQDN."
}

```

Принял изменение.
Также видим вывод значений ip-адресов команды terraform output.

![image.jpg](https://github.com/temagraf/My_Terraform/blob/main/4.png)


---

### Задание 5

1) В файле locals.tf опишите в одном local-блоке имя каждой ВМ, используйте интерполяцию ${..} с НЕСКОЛЬКИМИ переменными по примеру из лекции.
2) Замените переменные внутри ресурса ВМ на созданные вами local-переменные.
3) Примените изменения.

### Выполнения задания 5

Создал файл locals.tf, в котором объявил local-переменные для имен ВМ, используя интерполяцию, а затем заменим переменные в файле variables.tf на созданные local-переменные.
В корне проекта и добавил в него необходимые local-переменные.

```hcl
locals {
  project = "netology-develop-platform"
  env_web = "web"
  env_db  = "db"
  vm_web_instance_name = "${local.project}-${local.env_web}"
  vm_db_instance_name  = "${local.project}-${local.env_db}"
}

```

![image.jpg](https://github.com/temagraf/My_Terraform/blob/main/5%20-%20locals.png)


---

### Задание 6
1) Вместо использования трёх переменных ".._cores",".._memory",".._core_fraction" в блоке resources {...}, объедините их в единую map-переменную vms_resources и внутри неё конфиги обеих ВМ в виде вложенного map.
2) Создайте и используйте отдельную map переменную для блока metadata, она должна быть общая для всех ваших ВМ.
3) Найдите и закоментируйте все, более не используемые переменные проекта.
4) Проверьте terraform plan. Изменений быть не должно.

### Выполнения задания 6

Обновленный variables.tf

```hcl
variable "token" {
  type        = string
  description = "OAuth-token; https://cloud.yandex.ru/docs/iam/concepts/authorization/oauth-token"
}

variable "cloud_id" {
  type        = string
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/cloud/get-id"
}

variable "folder_id" {
  type        = string
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/folder/get-id"
}

variable "default_zone" {
  type        = string
  default     = "ru-central1-a"
  description = "https://cloud.yandex.ru/docs/overview/concepts/geo-scope"
}

variable "default_cidr" {
  type        = list(string)
  default     = ["10.0.1.0/24"]
  description = "https://cloud.yandex.ru/docs/vpc/operations/subnet-create"
}

variable "vpc_name" {
  type        = string
  default     = "develop"
  description = "VPC network & subnet name"
}

variable "vms_ssh_root_key" {
  type        = string
  description = "ssh-keygen -t ed25519"
}

variable "vms_resources" {
  type = map(map(any))
  default = {
    vm_web = {
      core_fraction = 5
      cores         = 2
      memory        = 1
    }
    vm_db = {
      core_fraction = 20
      cores         = 2
      memory        = 2
    }
  }
  description = "Resource configurations for VMs"
}

variable "vms_metadata" {
  type = map(string)
  default = {
    "serial-port-enable" = "1"
    "ssh-keys"           = "ubuntu:ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICMLupjJ1DJ6oImS9OyvFFNenNk8/hiRrzkWIZ171DFa root@debian12-2"
  }
  description = "Metadata for VMs"
}

```

Обновил блоки ресурсов в main.tf, чтобы использовались новые переменные vms_resources и vms_metadata.

```hcl
resource "yandex_vpc_network" "develop" {
  name = var.vpc_name
}

resource "yandex_vpc_subnet" "develop" {
  name           = var.vpc_name
  zone           = var.default_zone
  network_id     = yandex_vpc_network.develop.id
  v4_cidr_blocks = var.default_cidr
}

resource "yandex_compute_instance" "platform" {
  name        = local.vm_web_instance_name
  platform_id = var.vm_web_platform_id
  zone        = var.vm_web_zone

  resources {
    core_fraction = var.vms_resources.vm_web.core_fraction
    cores         = var.vms_resources.vm_web.cores
    memory        = var.vms_resources.vm_web.memory
  }

  boot_disk {
    initialize_params {
      image_id = var.vm_web_image_id
      type     = var.vm_web_disk_type
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.develop.id
    nat       = var.vm_web_nat
  }

  metadata = var.vms_metadata

  scheduling_policy {
    preemptible = var.vm_web_preemptible
  }
}

resource "yandex_compute_instance" "platform_db" {
  name        = local.vm_db_instance_name
  platform_id = var.vm_db_platform_id
  zone        = var.vm_db_zone

  resources {
    core_fraction = var.vms_resources.vm_db.core_fraction
    cores         = var.vms_resources.vm_db.cores
    memory        = var.vms_resources.vm_db.memory
  }

  boot_disk {
    initialize_params {
      image_id = var.vm_db_image_id
      type     = var.vm_db_disk_type
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.develop.id
    nat       = var.vm_db_nat
  }

  metadata = var.vms_metadata

  scheduling_policy {
    preemptible = var.vm_db_preemptible
  }
}

output "external_ip_address" {
  value = yandex_compute_instance.platform.network_interface.0.nat_ip_address
}

output "instance_id" {
  value = yandex_compute_instance.platform.id
}

```

Удаление неиспользуемых переменных.
Удалил все неиспользуемые переменные из файла variables.tf.

Обновленный variables.tf (без неиспользуемых переменных):

```hcl
variable "token" {
  type        = string
  description = "OAuth-token; https://cloud.yandex.ru/docs/iam/concepts/authorization/oauth-token"
}

variable "cloud_id" {
  type        = string
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/cloud/get-id"
}

variable "folder_id" {
  type        = string
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/folder/get-id"
}

variable "default_zone" {
  type        = string
  default     = "ru-central1-a"
  description = "https://cloud.yandex.ru/docs/overview/concepts/geo-scope"
}

variable "default_cidr" {
  type        = list(string)
  default     = ["10.0.1.0/24"]
  description = "https://cloud.yandex.ru/docs/vpc/operations/subnet-create"
}

variable "vpc_name" {
  type        = string
  default     = "develop"
  description = "VPC network & subnet name"
}

variable "vms_ssh_root_key" {
  type        = string
  description = "ssh-keygen -t ed25519"
}

variable "vms_resources" {
  type = map(map(any))
  default = {
    vm_web = {
      core_fraction = 5
      cores         = 2
      memory        = 1
    }
    vm_db = {
      core_fraction = 20
      cores         = 2
      memory        = 2
    }
  }
  description = "Resource configurations for VMs"
}

variable "vms_metadata" {
  type = map(string)
  default = {
    "serial-port-enable" = "1"
    "ssh-keys"           = "ubuntu:ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICMLupjJ1DJ6oImS9OyvFFNenNk8/hiRrzkWIZ171DFa root@debian12-2"
  }
  description = "Metadata for VMs"
}

```

 Выполните команду terraform plan, чтобы убедиться, что изменений в инфраструктуре нет.

 ```sh

terraform plan -out=tfplan
terraform apply tfplan


```

![image.jpg](https://github.com/temagraf/My_Terraform/blob/main/6.png)
