# ****Отчёт по лабораторной работе: AWS EC2 + ALB + Auto Scaling****

### ****Цель работы****

-   Научиться запускать виртуальные машины (EC2) в AWS.
-   Настроить Application Load Balancer (ALB) для маршрутизации трафика.
-   Настроить Target Groups и Auto Scaling для масштабирования нагрузки.
-   Проверить работу веб-сервера и динамическое масштабирование.

## ****1\. Создание VPC и подсетей****

****VPC:**** `project-vpc2`

-   CIDR: `10.0.0.0/16`
-   DNS hostnames: Enabled

****Подсети:****

| Name                     | Subnet ID                | AZ          | CIDR        | Public |
| ------------------------ | ------------------------ | ----------- | ----------- | ------ |
| project-public-subnet-10 | subnet-0e7990433315bb5ac | eu-north-1a | 10.0.0.0/20 | Да     |
| project-public-subnet-11 | subnet-00cc20aecc83b4fc2 | eu-north-1b | 10.0.1.0/24 | Да     |

****Настройки:****

-   Включена ****Auto-assign public IPv4**** для всех публичных подсетей.
-   Роутинг настроен через ****Internet Gateway****, чтобы Load Balancer и EC2 были доступны из интернета.

## ****2\. Запуск виртуальной машины (EC2)****

****Имя:**** `WebServer10`

****AMI:**** Amazon Linux 2023 (x86\_64)

****Instance type:**** `t3.micro`

****Key Pair:**** `project-ec2-keyy`

****Network:****

-   VPC: `project-vpc2`
-   Subnet: `project-public-subnet-11`
-   Auto-assign Public IP: ****Enabled****

****Security Group:**** `launch-wizard-3`

-   ****Inbound:****
-   -   SSH (22) — источник: твой IP
    -   HTTP (80) — источник: 0.0.0.0/0
-   ****Outbound:**** Все трафики — источник: 0.0.0.0/0

****Storage:**** 1x 8 GiB gp3 root volume

****User Data (init.sh):****

#!/bin/bash  
yum update -y  
amazon-linux-extras install nginx1 -y  
systemctl start nginx  
systemctl enable nginx  
echo "Hello from $(hostname)" > /usr/share/nginx/html/index.html  

****Результат:****

-   Статус проверки: 3/3 passed ✅
-   Проверка веб-сервера:
    
    http://<Public-IP>
    
    отображает: `Hello from WebServer10`


![Шаг](/images/image4.png)
![Шаг](/images/image5.png)

## ****3\. Настройка Application Load Balancer (ALB)****

****ALB:**** `project-alb`

****Тип:**** Application

****Scheme:**** Internet-facing

****Availability Zones:****

-   eu-north-1a → `project-public-subnet-10`
-   eu-north-1b → `project-public-subnet-11`

****Listeners:****

-   HTTP:80 → Forward to Target Group `project-target-group2`

****Security Group:****

-   Разрешает HTTP (80) от 0.0.0.0/0

![Шаг](/images/image3.png)

## ****4\. Создание Target Group****

****Имя:**** `project-target-group2`

****Тип:**** Instance

****Protocol / Port:**** HTTP:80

****VPC:**** `project-vpc2`

****Регистрация Targets:****

-   Зарегистрирован инстанс `WebServer10`
-   Health check path: `/`
-   Статус: ****Healthy**** ✅

## ****5\. Настройка Auto Scaling Group (ASG)****

****Имя:**** `project-asg`

****Launch Template / Configuration:****

-   Используется AMI Amazon Linux 2023
-   Instance type: t3.micro
-   Security Group: launch-wizard-3

![Шаг](/images/image1.png)

![Шаг](/images/image2.png)

****Подсети:****

-   `project-public-subnet-10`
-   `project-public-subnet-11`

****Min / Max / Desired capacity:****

-   Min: 1
-   Max: 3
-   Desired: 1

****Scaling Policy:**** Target Tracking

-   Metric: Average CPU Utilization = 50%
-   Alarm автоматически создаётся в CloudWatch

## ****6\. Тестирование ALB и веб-сервера****

1.  Перейти по ****DNS имени ALB****:
    
    http://project-alb-1648508451.eu-north-1.elb.amazonaws.com/  
    
2.  Страница веб-сервера отображается, при обновлении меняется IP — балансировка нагрузки работает.
3.  Все Targets в Target Group — ****Healthy**** ✅

## ****7\. Тестирование Auto Scaling****

1.  В CloudWatch → Alarms — выбран Alarm `TargetTracking-XX-AlarmHigh-...`
2.  CPU Utilization до нагрузки: 0-1%
3.  Создана нагрузка:
    
    for i in {1..6}; do curl http://project-alb-1648508451.eu-north-1.elb.amazonaws.com/load?seconds=60 & done  
    
4.  Через 2-3 минуты:
5.  -   CPU Utilization вырос
    -   Alarm стал красным
    -   В EC2 количество инстансов увеличилось (Auto Scaling сработал)

****Вывод:****

-   Auto Scaling автоматически добавляет инстансы при росте нагрузки.
-   После снижения нагрузки инстансы уменьшаются (scale-in).

## ****8\. Выводы****

-   ALB корректно распределяет трафик между инстансами.
-   Target Group следит за здоровьем инстансов (health checks).
-   Auto Scaling динамически масштабирует инфраструктуру под нагрузкой.
-   Работа лабораторной подтверждает навыки работы с AWS EC2, ALB, Target Groups и Auto Scaling.