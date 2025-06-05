# Docker

Установите Docker на хост машину

Set up Docker's apt repository.

Add Docker's official GPG key:

>sudo apt-get update
>sudo apt-get install ca-certificates curl
>sudo install -m 0755 -d /etc/apt/keyrings
>sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
>sudo chmod a+r /etc/apt/keyrings/docker.asc

Add the repository to Apt sources:

>echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

>sudo apt-get update

Install the Docker packages.

>sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

Создаем структуру проекта

>mkdir custom-nginx && cd custom-nginx

>mkdir html

Создаем кастомную страницу

>echo "<h1>Всем привет! это моя кастомная страница</h1>" > html/index.html

Теперь создаем Docker-образ на основе alpine с Nginx, который отдаёт кастомную HTML-страницу вместо стандартной.

Создаем Dockerfile

>nano Dockerfile

вписываем в него 

<pre># Используем официальный образ Nginx на Alpine
FROM nginx:alpine

# Удаляем дефолтную страницу Nginx
RUN rm -rf /usr/share/nginx/html/*

# Копируем нашу кастомную страницу
COPY html/ /usr/share/nginx/html/

# Открываем порт 80
EXPOSE 80

# Запускаем Nginx
CMD ["nginx", "-g", "daemon off;"]</pre>

Собираем образ

>docker build -t custom-nginx .

<pre>[+] Building 7.4s (8/8) FINISHED                                                         docker:default
<span style="color:#12488B"> =&gt; [internal] load build definition from Dockerfile                                               0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; transferring dockerfile: 176B                                                               0.0s</span>
<span style="color:#12488B"> =&gt; [internal] load metadata for docker.io/library/nginx:alpine                                    2.1s</span>
<span style="color:#12488B"> =&gt; [internal] load .dockerignore                                                                  0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; transferring context: 2B                                                                    0.0s</span>
<span style="color:#12488B"> =&gt; [1/3] FROM docker.io/library/nginx:alpine@sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f1  4.7s</span>
<span style="color:#12488B"> =&gt; =&gt; resolve docker.io/library/nginx:alpine@sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f1  0.7s</span>
<span style="color:#12488B"> =&gt; =&gt; sha256:62223d644fa234c3a1cc785ee14242ec47a77364226f1c811d2f669f96dc2ac8 2.50kB / 2.50kB     0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; sha256:f18232174bc91741fdf3da96d85011092101a032a93a388b79e99e69c2d5c870 3.64MB / 3.64MB     1.3s</span>
<span style="color:#12488B"> =&gt; =&gt; sha256:61ca4f733c802afd9e05a32f0de0361b6d713b8b53292dc15fb093229f648674 1.79MB / 1.79MB     0.8s</span>
<span style="color:#12488B"> =&gt; =&gt; sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10 10.33kB / 10.33kB   0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; sha256:6769dc3a703c719c1d2756bda113659be28ae16cf0da58dd5fd823d6b9a050ea 10.79kB / 10.79kB   0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; sha256:b464cfdf2a6319875aeb27359ec549790ce14d8214fcb16ef915e4530e5ed235 629B / 629B         0.6s</span>
<span style="color:#12488B"> =&gt; =&gt; sha256:d7e5070240863957ebb0b5a44a5729963c3462666baa2947d00628cb5f2d5773 955B / 955B         1.0s</span>
<span style="color:#12488B"> =&gt; =&gt; sha256:81bd8ed7ec6789b0cb7f1b47ee731c522f6dba83201ec73cd6bca1350f582948 402B / 402B         1.2s</span>
<span style="color:#12488B"> =&gt; =&gt; sha256:197eb75867ef4fcecd4724f17b0972ab0489436860a594a9445f8eaff8155053 1.21kB / 1.21kB     1.3s</span>
<span style="color:#12488B"> =&gt; =&gt; sha256:34a64644b756511a2e217f0508e11d1a572085d66cd6dc9a555a082ad49a3102 1.40kB / 1.40kB     1.5s</span>
<span style="color:#12488B"> =&gt; =&gt; sha256:39c2ddfd6010082a4a646e7ca44e95aca9bf3eaebc00f17f7ccc2954004f2a7d 15.52MB / 15.52MB   3.4s</span>
<span style="color:#12488B"> =&gt; =&gt; extracting sha256:f18232174bc91741fdf3da96d85011092101a032a93a388b79e99e69c2d5c870          0.1s</span>
<span style="color:#12488B"> =&gt; =&gt; extracting sha256:61ca4f733c802afd9e05a32f0de0361b6d713b8b53292dc15fb093229f648674          0.1s</span>
<span style="color:#12488B"> =&gt; =&gt; extracting sha256:b464cfdf2a6319875aeb27359ec549790ce14d8214fcb16ef915e4530e5ed235          0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; extracting sha256:d7e5070240863957ebb0b5a44a5729963c3462666baa2947d00628cb5f2d5773          0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; extracting sha256:81bd8ed7ec6789b0cb7f1b47ee731c522f6dba83201ec73cd6bca1350f582948          0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; extracting sha256:197eb75867ef4fcecd4724f17b0972ab0489436860a594a9445f8eaff8155053          0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; extracting sha256:34a64644b756511a2e217f0508e11d1a572085d66cd6dc9a555a082ad49a3102          0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; extracting sha256:39c2ddfd6010082a4a646e7ca44e95aca9bf3eaebc00f17f7ccc2954004f2a7d          0.4s</span>
<span style="color:#12488B"> =&gt; [internal] load build context                                                                  0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; transferring context: 152B                                                                  0.0s</span>
<span style="color:#12488B"> =&gt; [2/3] RUN rm -rf /usr/share/nginx/html/*                                                       0.4s</span>
<span style="color:#12488B"> =&gt; [3/3] COPY html/ /usr/share/nginx/html/                                                        0.0s</span>
<span style="color:#12488B"> =&gt; exporting to image                                                                             0.1s</span>
<span style="color:#12488B"> =&gt; =&gt; exporting layers                                                                            0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; writing image sha256:f1160299108be5c2f69407c7aa374fca94c946ab0e1edf38738eb1d7018ad4b8       0.0s</span>
<span style="color:#12488B"> =&gt; =&gt; naming to docker.io/library/custom-nginx    </span></pre>

Запускаем контейнер

>docker run -d -p 8080:80 --name my-nginx custom-nginx

Проверяем какие контейнеры у нас работают

>docker ps -a

<pre>CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                     NAMES
79413cfa1c93   custom-nginx   &quot;/docker-entrypoint.…&quot;   2 minutes ago   Up 2 minutes   0.0.0.0:8080-&gt;80/tcp, [::]:8080-&gt;80/tcp   my-nginx
</pre>

Проверим доступность нашей страницы

>curl http://localhost:8080

<pre>&lt;h1&gt;Всем привет! это моя кастомная страница&lt;/h1&gt;
</pre>

Ответ получен!

# Разница между контейнером и образом

Образ (Image) — это шаблон (read-only), на основе которого создаются контейнеры. Он состоит из слоёв файловой системы и метаданных.

Контейнер (Container) — это запущенный экземпляр образа (read-write). Контейнер добавляет слой для записи поверх образа, что позволяет изменять данные во время работы. 

# Можно ли в контейнере собрать ядро?

Технически — да, но практически бессмысленно.

Контейнеры используют ядро хостовой системы, а не своё. Даже если мы соберём ядро внутри контейнера, оно не сможет заменить ядро хоста.
Для сборки ядра требуются права которые по умолчанию ограничены в контейнерах из соображений безопасности.
Это может пригодиться только для тестирования сборки ядра, но не запуска.
Для работы с ядром лучше использовать виртуальные машины.
