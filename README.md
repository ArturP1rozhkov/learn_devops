# Оркестрация группой Docker контейнеров на примере Docker Compose
---
## ЗАДАЧА 1
### Установил docker и docker compose plugin на свою linux ВМ.
Установлено было в рамках предыдущего задания. Вместе с тем, согласно официальной документации:
```bash
# 1. Обновляю систему и устанавливаю зависимости

sudo apt update && sudo apt install -y ca-certificates curl

# 2. Создаю директорию для GPG-ключей

sudo install -m 0755 -d /etc/apt/keyrings  
sudo curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg)  
-o /etc/apt/keyrings/docker.asc  
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 3. Добавляю официальный репозиторий Docker

echo  
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc]  
[https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu)  
(. /etc/os-release && echo "{UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" |  
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Установливаю Docker Engine + Compose Plugin (всё в одной команде)

sudo apt update  
sudo apt install -y docker-ce docker-ce-cli containerd.io  
docker-buildx-plugin docker-compose-plugin

# 5. Проверяю установку

docker --version  
docker compose version

# 6. Добавляю себя в группу docker
sudo usermod -aG docker $USER  
newgrp docker
```
### Скачал образ nginx:1.29.0
```bash
docker pull nginx:1.29.0

# Проверяю, что образ скачан

docker images | grep nginx
```
### Создал Dockerfile и сменил содержимое дефолтной индекс-страницы(/usr/share/nginx/html/index.html), на файл index.html с необходимым содержимым:

#### Создал рабочую директорию и нужные файлы:

```bash
mkdir ~/custom-nginx && cd ~/custom-nginx
```

#### **Создал index.html:**

```bash
cat > index.html <<'EOF'
<html>
<head>
Hey, Netology
</head>
<body>
<h1>I will be DevOps Engineer!</h1>
</body>
</html>
EOF
```

#### **Создал Dockerfile:**

```bash
cat > Dockerfile <<'EOF'
FROM nginx:1.29.0
COPY index.html /usr/share/nginx/html/index.html
EOF
```

#### **Собрал образ**

```bash
docker build -t custom-nginx:1.0.0 .
```

#### **Проверка локально:**

```bash
docker run -d -p 8080:80 --name test-nginx custom-nginx:1.0.0
curl http://localhost:8080
# вернул index.html (или в браузере по адресу: http://localhost:8080 )
docker stop test-nginx && docker rm test-nginx
```

### Запушил на Docker Hub

```bash
# 1. Залогинился в Docker Hub
docker login

# 2. Тегировал образ 
docker tag custom-nginx:1.0.0 turistosartur/custom-nginx:1.0.0

# 3. Запушил
docker push turistosartur/custom-nginx:1.0.0
```
### Итоговая ссылка: [https://hub.docker.com/repository/docker/turistosartur/custom-nginx/general](https://hub.docker.com/repository/docker/turistosartur/custom-nginx/general)
---
## ЗАДАЧА 2
### **Запуск контейнера**

```bash
# Запуск с КВА в имени
docker run -d \
  --name "KVA-custom-nginx-t2" \
  -p 127.0.0.1:8080:80 \
  turistosartur/custom-nginx:1.0.0
```
### Переименовал (не удаляя)**

```bash
docker rename "KVA-custom-nginx-t2" "custom-nginx-t2"
```

**Проверка:**

```bash
docker ps
```

### **выполнил команду**

```bash
date +"%d-%m-%Y %T.%N %Z" ; sleep 0.150 ; docker ps ; ss -tlpn | grep 127.0.0.1:8080 ; docker logs custom-nginx-t2 -n1 ; docker exec -it custom-nginx-t2 base64 /usr/share/nginx/html/index.html
```

| Команда                           | Что делает                                                       |
| --------------------------------- | ---------------------------------------------------------------- |
| `date +"%d-%m-%Y %T.%N %Z"`       | Дата/время с миллисекундами                                      |
| `sleep 0.150`                     | Пауза 150мс                                                      |
| `docker ps`                       | Статус контейнеров                                               |
| `ss -tlpn \| grep 127.0.0.1:8080` | Слушающий порт 8080                                              |
| `docker logs ... -n1`             | Последняя строка логов                                           |
| `docker exec ... base64 ...`      | подключение к контейнеру и вывод **Base64-кодировки** index.html |
### **проверил доступность**

```bash
# Через curl
curl http://127.0.0.1:8080

# Через браузер
# http://127.0.0.1:8080
```
### **ответ:**

```bash
<html>
<head>Hey, Netology</head>
<body><h1>I will be DevOps Engineer!</h1></body>
</html>

```

---
## ЗАДАЧА 3
### **Подключился к потокам ввода/вывода контейнера**
   
   ```bash
   docker attach custom-nginx-t2
   ```
   
   - Подключаешься к **PID 1** (nginx) через stdin/stdout/stderr  (устройство ввода  - клавиатура, вывода - монитор, лог ошибок)
       
   - **Ctrl+C** → SIGINT отправляет  nginx команду остановки → **контейнер останавливается**.  Ctrl+C убил главный процесс nginx (PID 1). Docker останавливается, когда главный процесс завершён.
       
   
   **После Ctrl+C:** `docker ps -a` контейнер **Exited**
   
### **Перезапуск**
   
   ```bash
   docker start custom-nginx-t2
   ```
   
### **Вошел в контейнер в интерактивном режиме, установил nano
   
   ```bash
   docker exec -it custom-nginx-t2 bash
   
   # Внутри контейнера: обновил репозитории, установил nano, curl
   apt-get update
   apt-get install -y nano curl
   
   # отредактировал конфиг nginx и заменил `listen 80` на `listen 81`
   nano /etc/nginx/conf.d/default.conf
   nginx -s reload
   ```
   
### **Проверил внутри контейнера:**
   
   ```bash
   curl http://127.0.0.1:80    # недоступно
   curl http://127.0.0.1:81    # вывод содержимого файла index.html
   exit
   ```
   
   
### **Проверил вывод команд:**
   
   ```bash
   ss -tlpn | grep 127.0.0.1:8080     
   docker port custom-nginx-t2        
   curl http://127.0.0.1:8080         
   ```
   
#### **Объяснение:** 
   
   - **Хост пробрасывает порт 8080 в порт 80 контейнера**  
       
   - **Однако после корректировки конфига Nginx внутри контейнера, он слушает 81 порт **  
       
   - **Поскольку порты не совпадают, трафик не доходит**

### Необязательное: перемонтируем конфиг и перезапускаем контейнер:**

```bash
# остановил контейнер
docker stop custom-nginx-t2
# Сохранил изменённый конфиг из контейнера на хост в текущую папку
docker cp custom-nginx-t2:/etc/nginx/conf.d/default.conf ./default.conf

# Пересоздал и запустил в фоновом режиме с новым именем с монтированием исправленного конфига и пробросом порта 8080 в 81 (корректный конфигу контейнера )
docker run -d \
  --name custom-nginx-t2-fixed \
  -p 127.0.0.1:8080:81 \
  -v $(pwd)/default.conf:/etc/nginx/conf.d/default.conf \
  turistosartur/custom-nginx:1.0.0
  
  # проверил
  curl http://127.0.0.1:8080
```

### Удалил контейнер без его остановки**

```bash
docker rm -f custom-nginx-t2
```

**`-f` = force** — принудительно убил и удалил контейнер

---
## ЗАДАЧА 4

```bash
# Запустил centos-контейнер (с docker hub удален образ centos, загружена аналогичная OS almalinux в контейнере с именем centos-container)

docker run -d  
--name centos-container  
-v $(pwd):/data  
almalinux:latest  
sleep infinity

# Запустил debian-контейнер

docker run -d  
--name debian-container  
-v $(pwd):/data  
debian:latest  
sleep infinity

# Проверка что они оба работают

docker ps

# Подключился в интерактивном режиме к centos и создал файл centos-file.txt, проверил

docker exec -it centos-container bash  
echo "I am CentOS!" > /data/centos-file.txt  
ls -la /data  
exit

# Создал на ХОСТЕ файл host-file.txt, проверил

echo "I am HOST!" > host-file.txt  
ls -la

# Зашел в debian контейнер в интерактивном режиме, проверил вывод файлов

docker exec -it debian-container bash  
ls -la /data  
cat /data/centos-file.txt  
cat /data/host-file.txt  
exit
```

---
## ЗАДАЧА 5

### Создал отдельную директорию с файлами compose.yaml и docker-compose.yaml

```bash
mkdir -p /tmp/netology/docker/task5
cd /tmp/netology/docker/task5

cat > compose.yaml <<'EOF'
version: "3"
services:
  portainer:
    network_mode: host
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
EOF


cat > docker-compose.yaml <<'EOF'
version: "3"
services:
  registry:
    image: registry:2
    ports:
    - "5000:5000"
EOF
```

### Запустил compose 

```bash
docker compose up -d
```

**Будет запущен `compose.yaml`** — потому что он имеет более высокий приоритет. Docker Compose ищет файлы в порядке: `compose.yaml` ---> `docker-compose.yaml` и останавливается на первом найденном.

### Отредактировал файл compose.yaml для запуска обеих файлов путем добавления директивы `include` , которая позволяет **подключить другой compose-файл** внутрь основного:`

```bash
cat > compose.yaml <<'EOF'
version: "3"

include:
  - docker-compose.yaml

services:
  portainer:
    network_mode: host
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
EOF

# запустил
docker compose up -d

# проверил
docker ps
```

### Залил custom-nginx в запущенный через docker-compose.yaml локальный registry по адресу 127.0.0.1:5000

```bash
# Тегировал образ для локального registry
docker tag turistosartur/custom-nginx:1.0.0 127.0.0.1:5000/custom-nginx:latest

# 2. Запушил в локальный registry
docker push 127.0.0.1:5000/custom-nginx:latest

# 3. Проверил
curl http://127.0.0.1:5000/v2/_catalog
# вернул: {"repositories":["custom-nginx"]}
```

### Настройка Portainer и деплой compose: в браузере открыл  http://127.0.0.1:9000 (открылось редиректом из https://127.0.0.1:9443), ввел логин, пароль с подтверждением

- Выбрал **"Get Started"** ---> **local** окружение ---> Имя стека: `custom-nginx-stack` ---> в поле Web editor вставил:

```bash
version: '3'

services:
  nginx:
    image: 127.0.0.1:5000/custom-nginx
    ports:
      - "9090:80"
```

----> Нажал **Deploy the stack** внизу страницы ----> перешел в браузере на страницу portainer "http://127.0.0.1:9000/" с nginx, нажал на кнопку "inspect" ---> сделал скрин "Config" от поля "AppArmorProfile" до "Driver".
### Удалил compose.yaml 

```bash
rm compose.yaml
docker compose up -d
```

Получил: WARN: Found orphan containers

**Суть предупреждения:** Docker Compose видит контейнеры, которые были созданы предыдущей версией проекта (с portainer), но в новом файле их нет. Они называются **"orphan"** (сироты) - существуют, но не управляются текущим compose-файлом. Docker  предложил их удалить.

```bash
docker compose up -d --remove-orphans
```

### Погасил всё одной командой

```bash
docker compose down
```

**`docker compose down`** — останавливает и удаляет все контейнеры проекта одной командой в отличие от `docker compose stop` (только останавливает).
