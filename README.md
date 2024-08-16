# Установка и настройка Cloudflare Tunnel для Synology NAS
<img src="https://github.com/avenom/synology-cloudflare-tunnel/blob/main/images/synology-cloudflare-tunnel.png">

Cloudflare Tunnel создает зашифрованный туннель между вашим исходным веб-сервером и ближайшим центром обработки данных Cloudflare, не открывая при этом общедоступных входящих портов. Позволяет быстро развернуть инфраструктуру в среде Zero Trust, и все запросы, обращенные к вашим ресурсам, будут сначала проходить через надежные фильтры безопасности, настроенные в сети Cloudflare.

Достаточно добавить ваш домен в Cloudflare, а затем настроить один из доступных клиентов (коннекторов) для Linux, Windows, macOS и Docker. После этого служба станет доступной через интернет на вашем домене.

Плюсы:
- Нет необходимости использовать Synology QuickConnect, Synology DDNS, обратные прокси.
- Для доступа из интернета нет необходимости открывать порты в роутере.
- Автоматический HTTPS с SSL-сертификатом.
- Для отключения доступа достаточно остановить контейнер Cloudflared в Docker.

Минусы:
- Нужен платный домен.
- Весь трафик будет проходить через серверы Cloudflare, так что если вам необходима максимальная конфиденциальность данных, то лучше использовать свой VPN.

## Оглавление

1. [Требования](#requirements)
2. [Добавление и настройка домена в Cloudflare](#cloudflare)
3. [Настройка Cloudflare Zero Trust и Cloudflared Docker Compose](#zero)
4. [Настройка безопасности Cloudflare Tunnel](#security)

## 1. Требования <a name="requirements"></a>

- Доменное имя.
- Учетная запись [Cloudflare](https://dash.cloudflare.com/sign-up).
- Кредитная карта иностранного банка для Cloudflare Zero Trust (нужна только для активации бесплатного плана).
- Container Manager в Synology DSM.

## 2. Добавление и настройка домена в Cloudflare <a name="cloudflare"></a>

1. Купите доменное имя у регистратора, в качестве примера будет использован домен хостера [Timeweb](https://timeweb.com/ru/services/domains/). После покупки, вы сможете начать работать с доменом после обновления информации на DNS-серверах. Обычно это занимает не больше 3 часов, реже - до 24 часов.

2. Залогиньтесь в [Cloudflare](https://dash.cloudflare.com/login).

3. В аккаунте [Cloudflare](https://dash.cloudflare.com/), на верхней панели нажмите + Add > Existing domain, добавьте ваш домен site.ru, выберите Quick scan for DNS records и нажмите Continue.

<img src="https://github.com/avenom/synology-cloudflare-tunnel/blob/main/images/add.png">

<img src="https://github.com/avenom/synology-cloudflare-tunnel/blob/main/images/site.png">

5. Прокрутите страницу вниз, выберите план Free и нажмите Continue.

6. Далее ваш домен просканируется и его DNS-записи импортируются в Cloudflare, нажмите Continue.

7. Найдите на странице Cloudflare nameservers.

Пример:

```
name1.ns.cloudflare.com
name2.ns.cloudflare.com
```

7. Пример настройки nameservers домена хостера Timeweb. На [странице доменов](https://hosting.timeweb.ru/domains) нажмите рядом с вашим доменом кнопку настройки и нажмите "Настройки DNS", сверху страницы нажмите "Редактировать NS-серверы".

<img src="https://github.com/avenom/synology-cloudflare-tunnel/blob/main/images/dns.png">

Оставьте только два сервера Cloudflare:

```
name1.ns.cloudflare.com
name2.ns.cloudflare.com
```

<img src="https://github.com/avenom/synology-cloudflare-tunnel/blob/main/images/dns-ns.png">

8. Дождитесь от вашего регистратора письма с успешнным изменением данных домена и делегированием на серверы Cloudflare. На странице Cloudflare нажмите "Check nameservers now" и дождитесь письма от Cloudflare с содержанием "[Cloudflare]: site.ru is now active on a Cloudflare Free plan" и нажмите "Continue".

9. Далее нажмите Get started и отметьте следующие настройки:

```
Automatic HTTPS Rewrites: Yes
Always use HTTPS: Yes
```

Нажмите Summary и Finish.

## 3. Настройка Cloudflare Zero Trust и Cloudflared Docker Compose <a name="zero"></a>

1. Вернитесь на главную [Cloudflare](https://dash.cloudflare.com), в левом меню выберите Zero Trust.

2. Придумайте имя команды. Имя команды создает уникальный домен для вашей учетной записи Cloudflare Zero Trust. Вы можете изменить его позже. Нажмите "Continue".

3. Выберите план Free > Select plan > Proceed to payment > Add payment method > Введите данные кредитной карты иностранного банка > Done > Purchase > Getting Started with Cloudflare Zero Trust Free.

4. В левом меню выберите Networks > Tunnels > Create a tunnel > Select Cloudflared > Next > Придумайте название для туннеля > Save Tunnel.

5. На странице Configure выберите Docker, скопируйте и сохраните куда-нибудь код docker run (кнопкой возле кода).

6. В Synology DSM откройте File Station и создайте следующую структуру папок:

```
/docker/cloudflared/
```

7. Создайте в Container Manager новый проект с названием cloudflared, выберите путь /docker/cloudflared/, выберите в источнике "Создать docker-compose.yml", вставьте в окно ниже следующий код:

```
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    network_mode: host
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=ВАШТОКЕН # Вставьте из скопированного ранее кода все, что после --token 
    restart: always
```

8. Вернитесь на страницу Configure, в Connectors должен быть Status: Connected, нажмите Next.

9. На странице Route Traffic откройте вкладку Public Hostnames и введите ваши данные:

```
Subdomain: nas
Domain: site.ru
Type: HTTP
URL: synologyip:5000 # Локальный IP-адрес вашего Synology DSM.
```

Нажмите Save tunnel.

10. Теперь можете проверить доступ к Synology NAS с домена https://nas.site.ru - соединение HTTPS с SSL-сертификатом и без открытия порта 443 в роутере.

Таким же способом с помощью субдоменов можно настроить доступ для других пакетов DSM и контейнеров Docker, выбрав в Networks > Tunnels ранее созданный туннель и нажав Edit, открыв вкладку Public Hostname и нажав Add a public hostname.

## 4. Настройка безопасности Cloudflare Tunnel <a name="security"></a>

1. Вернитесь на главную [Cloudflare](https://dash.cloudflare.com), в левом меню выберите Zero Trust > Settings > Authentication. Проверьте выбран ли в Login methods: One-time PIN (одноразовый пин-код на вашу почту).

2. В левом меню выберите Access > Access groups > Add a Group. Добавим доступ к вашему домену только из определенной страны с пин-кодом на вашу почту:

```
Group name: Russia + Email
Set as default group: Включить

Include
Selector: Country
Value: Russian Federation

+ Add require

Require
Selector: Emails
Value: Ваша почта
```

Нажмите Save.

3. В левом меню выберите Access > Applications > Add an application > Self-hosted > Select и введите следующие данные:

```
Application Configuration
Application name: NAS
Session Duration: 24 hours

Application domain
Subdomain: * 
Domain: site.ru
```

С субдоменом * аутентификацию будут проходить Synology DSM и все субдомены. Пройдя аутентификацию на любом субдомене *.site.ru, вы получаете доступ сразу ко всем субдоменам.

Нажмите Next и введите следующие данные:

```
Policy name: NAS Access
Action: Allow
В Assign a group отметить Russia + Email
```

Нажмите Next, нажмите Add application.

4. Теперь когда вы зайдете на https://nas.site.ru, сперва откроется страница Cloudflare с вводом вашей почты:

<img src="https://github.com/avenom/synology-cloudflare-tunnel/blob/main/images/email.png">

После чего нужно будет ввести код, который придет на вашу почту:

<img src="https://github.com/avenom/synology-cloudflare-tunnel/blob/main/images/email-code.png">

Если на сайт зайдет пользователь не из России, то он увидит такую страницу:

<img src="https://github.com/avenom/synology-cloudflare-tunnel/blob/main/images/country.png">

Настройка Cloudflare Tunnel завершена!
