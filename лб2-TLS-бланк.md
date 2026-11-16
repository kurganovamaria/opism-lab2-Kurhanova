<!--
Бланк звіту до лабораторної роботи «TLS та сертифікати».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ |Курганова Марія |
| Група |F2 2.01 |
| Ідентифікатор |se-201-17 |
| Дата виконання |15.11.2026 |
| Середовище | GitHub Codespaces |
| Версія OpenSSL | OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024) |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер | `origin.se-201-17.lab` |
| Імена для сертифіката | `www.se-201-17.lab`, `files.se-201-17.lab` |
| Ваш сервер (IP) |`f1.se-201-17.lab`, `f2.se-201-17.lab`, `f3.se-201-17.lab`, `f4.se-201-17.lab`, `f5.se-201-17.lab` |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```text
root@se-201-17:/lab/work# timeout 8 tcpdump -ni any -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt &
sleep 1; curl -s http://origin.se-201-17.lab/; wait
grep -a -E 'GET /|Host:|server: web' t1-http.txt
[1] 48
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
server: web
Host: origin.se-201-17.lab
SNI: 
protocol: HTTP/1.1 
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni any -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
16:56:33.172067 eth0  Out IP 10.53.1.1.46012 > 10.53.0.80.80: Flags [P.], seq 1:84, ack 1, win 502, options [nop,nop,TS val 2449406696 ecr 2883457545], length 83: HTTP: GET / HTTP/1.1
....... GET / HTTP/1.1
Host: origin.se-201-17.lab
server: web
Host: origin.se-201-17.lab
```

Вивід кроку 1.2:

```text
root@se-201-17:/lab/work# timeout 8 tcpdump -ni any -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt &
sleep 1; curl -s --cacert $STAND_CA https://origin.se-201-17.lab/; wait
grep -a -c -E 'GET /|server: web' t1-https.txt
grep -a -o 'origin.se-201-17.lab' t1-https.txt | wc -l
[1] 54
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
17 packets captured
17 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni any -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
0
1
```

| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` | Є | Ні |
| заголовок `Host` | Є | Ні |
| тіло відповіді (`server: web`) | Є | Ні |
| ім'я віддаленого сервера (кількість входжень) | 0 | 1 |

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?

Ім'я сервера передається у повідомленні ClientHello протоколу TLS (на етапі рукостискання у відкритому вигляді, оскільки шифрування ще не увімкнено) у полі розширення server_name.

2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?

Протокол HTTP працює поверх TCP безпосередньо і не містить TLS-обгортки. Поле SNI є частиною виключно протоколу TLS (розширення ClientHello), тому під час перехоплення неширфрованого HTTP-трафіку воно залишається порожнім (ім'я хоста передається у заголовку Host самого HTTP-запиту).

3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?

Версію протоколу визначило узгодження за допомогою розширення ALPN під час рукостискання TLS. Клієнт curl за замовчуванням підтримує і пропонує протокол h2 (HTTP/2) для безпечних з'єднань через HTTPS, тоді як для звичайного HTTP за замовчуванням використовується класичний HTTP/1.1.

## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```text
root@se-201-17:/lab/work# openssl s_client -connect origin.se-201-17.lab:443 -servername origin.se-201-17.lab -alpn h2,http/1.1 \
  -CAfile $STAND_CA -trace </dev/null > t2-13.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-13.txt
grep -a -E '^(ALPN|New,)|Verify return' t2-13.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=331
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Certificate, Length=983
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=74
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Alert (21)
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
ALPN protocol: h2
Verify return code: 0 (ok)
```

Вивід кроку 2.2 (TLS 1.2):

```text
root@se-201-17:/lab/work# openssl s_client -connect origin.se-201-17.lab:443 -servername origin.se-201-17.lab -alpn h2,http/1.1 \
  -tls1_2 -CAfile $STAND_CA -trace </dev/null > t2-12.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-12.txt
grep -a -E '^(ALPN|New,)|Verify return' t2-12.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=226
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=978
Received Record
  Content Type = Handshake (22)
    ServerKeyExchange, Length=111
Received Record
  Content Type = Handshake (22)
    ServerHelloDone, Length=0
Sent Record
  Content Type = Handshake (22)
    ClientKeyExchange, Length=33
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = Handshake (22)
    NewSessionTicket, Length=230
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = ApplicationData (23)
Sent Record
  Content Type = Alert (21)
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES256-GCM-SHA384
ALPN protocol: h2
    Verify return code: 0 (ok)
```

<!-- Один рядок на кожен запис з виводу 2.1, у тому самому порядку. Напрям: К→С або С→К. -->

| № | Напрям | `Content Type` | `Inner Content Type` | Повідомлення | `Length` |
|---|---|---|---|---|---|
| 1 |К→С |Handshake (22) | - |ClientHello | 331 |
| 2 |С→К |Handshake (22) | - |ServerHello | 118 |
| 3 |С→К |ChangeCipherSpec (20) | - | - | - |
| 4 |С→К |ApplicationData (23) |Handshake (22) |EncryptedExtensions | 15 |
| 5 |С→К |ApplicationData (23) |Handshake (22) |Certificate | 983 |
| 6 |С→К |ApplicationData (23) |Handshake (22) |CertificateVerify | 74 |
| 7 |С→К |ApplicationData (23) |Handshake (22) |Finished | 48 |
| 8 |К→С |ChangeCipherSpec (20) | - | - | - |
| 9 |К→С |ApplicationData (23) |Handshake (22) |Finished | 48 |

Вивід кроків 2.3 і 2.4:

```text
root@se-201-17:/lab/work# grep -a -A3 'extension_type=server_name' t2-13.txt | head -4
        extension_type=server_name(0), length=25
          0000 - 00 17 00 00 14 6f 72 69-67 69 6e 2e 73 65 2d   .....origin.se-
          000f - 32 30 31 2d 31 37 2e 6c-61 62                  201-17.lab
        extension_type=ec_point_formats(11), length=4
root@se-201-17:/lab/work# for v in 1.3 1.2; do
  timeout 6 tcpdump -ni any -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt &
  sleep 1; curl -s -o /dev/null --tlsv1.2 --tls-max $v --cacert $STAND_CA https://origin.se-201-17.lab/; wait
  echo "TLS $v: $(grep -a -c 'OPISM Stand Intermediate' t2-dump-$v.txt)"
done
[1] 70
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
17 packets captured
17 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni any -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.3: 0
[1] 76
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
15 packets captured
15 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni any -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.2: 2
```

| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт | 331 | 226 |
| Номер першого запису, тип якого спостерігач не бачить | 4 | - |
| Кількість RTT від ClientHello до першого HTTP-запиту | 1 RTT | 2 RTT |
| Чи видно сертифікат у дампі (результат 2.4) | Ні | Так |
| Обраний протокол ALPN | h2 | h2 |

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?

Шифрування в TLS 1.3 починається з повідомлення сервера EncryptedExtensions. Спостерігач у дампі бачить зовнішній тип запису як ApplicationData (23), а справжній тип повідомлення ховається всередині як Inner Content Type = Handshake (22).

2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.

ClientHello у TLS 1.3 довший, оскільки клієнт одразу передає параметри для обміну ключами (наприклад, підтримку конкретних груп і попередні параметри у розширенні key_share). Це розширення дозволяє серверу одразу обчислити спільний ключ і надіслати свої параметри разом із сертифікатом та відповіддю, скорочуючи рукостискання до 1 RTT.

3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?

Значення складається з:
2 байти - тип імені (Name Type, зазвичай 0x00 для host_name).
2 байти - загальна довжина самого імені в байтих.
13 байт - рядок імені origin.se-201-17.lab (17 символів).
Для доменного імені www.se-201-17.lab (15 символів) загальна довжина розширення становитиме 2 + 2 + 15 = 19 байт.

4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?

Він передається виключно заради забезпечення зворотної сумісності зі старими проміжними пристроями, які очікують побачити цей пакет за традиціями TLS 1.2 і блокували б з'єднання, якби він був повністю відсутній.

## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```text
root@se-201-17:/lab/work# acme-root
openssl x509 -in /lab/tls/acme-root.crt -noout -subject -dates
curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-17.lab/
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 54f0fc
notBefore=Nov 16 15:53:26 2026 GMT
notAfter=Nov 16 15:53:26 2056 GMT
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.

```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text
root@se-201-17:/lab/work# lego --server $ACME_DIR --email admin@se-201-17.lab --accept-tos --path /lab/work/lego \
  --http --http.webroot /var/www/acme \
  --domains www.se-201-17.lab --domains files.se-201-17.lab run --no-bundle
grep acme-challenge /var/log/nginx/access.log
2026/11/16 17:08:42 No key found for account admin@se-201-17.lab. Generating a P256 key.
2026/11/16 17:08:42 Saved key to /lab/work/lego/accounts/ca.lab_14000/admin@se-201-17.lab/keys/admin@se-201-17.lab.key
2026/11/16 17:08:42 [INFO] acme: Registering account for admin@se-201-17.lab
!!!! HEADS UP !!!!

Your account credentials have been saved in your Let's Encrypt
configuration directory at "/lab/work/lego/accounts".

You should make a secure backup of this folder now. This
configuration directory will also contain certificates and
private keys obtained from Let's Encrypt so making regular
backups of this folder is ideal.
2026/11/16 17:08:43 [INFO] [www.se-201-17.lab, files.se-201-17.lab] acme: Obtaining SAN certificate
2026/11/16 17:08:43 [INFO] [files.se-201-17.lab] AuthURL: https://ca.lab:14000/authZ/ctO14v5AM5AYtdswG06EoT2M14IIqtp7NJeM830TxnM
2026/11/16 17:08:43 [INFO] [www.se-201-17.lab] AuthURL: https://ca.lab:14000/authZ/ubb0XPC9_MGxjSYWmsFqdVWWJcdIzxP3e2-7E8Nsp0o
2026/11/16 17:08:43 [INFO] [files.se-201-17.lab] acme: Could not find solver for: tls-alpn-01
2026/11/16 17:08:43 [INFO] [files.se-201-17.lab] acme: use http-01 solver
2026/11/16 17:08:43 [INFO] [www.se-201-17.lab] acme: Could not find solver for: tls-alpn-01
2026/11/16 17:08:43 [INFO] [www.se-201-17.lab] acme: use http-01 solver
2026/11/16 17:08:43 [INFO] [files.se-201-17.lab] acme: Trying to solve HTTP-01
2026/11/16 17:08:49 [INFO] [files.se-201-17.lab] The server validated our request
2026/11/16 17:08:49 [INFO] [www.se-201-17.lab] acme: Trying to solve HTTP-01
2026/11/16 17:08:53 [INFO] [www.se-201-17.lab] The server validated our request
2026/11/16 17:08:53 [INFO] [www.se-201-17.lab, files.se-201-17.lab] acme: Validations succeeded; requesting certificates
2026/11/16 17:08:53 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/16 17:08:53 [INFO] [www.se-201-17.lab] Server responded with a certificate.
10.53.0.10 - - [16/Nov/2026:17:08:43 +0000] "GET /.well-known/acme-challenge/0GfCR0XrWnX_DcuLD8cozJr46HfuNrQQHR_eKZCNr-g HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:17:08:43 +0000] "GET /.well-known/acme-challenge/0GfCR0XrWnX_DcuLD8cozJr46HfuNrQQHR_eKZCNr-g HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:17:08:43 +0000] "GET /.well-known/acme-challenge/0GfCR0XrWnX_DcuLD8cozJr46HfuNrQQHR_eKZCNr-g HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:17:08:49 +0000] "GET /.well-known/acme-challenge/sho7fwg0W0CTCo610mV7Y3igeFhXU0RqO9Qoo6i_N3o HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:17:08:49 +0000] "GET /.well-known/acme-challenge/sho7fwg0W0CTCo610mV7Y3igeFhXU0RqO9Qoo6i_N3o HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [16/Nov/2026:17:08:49 +0000] "GET /.well-known/acme-challenge/sho7fwg0W0CTCo610mV7Y3igeFhXU0RqO9Qoo6i_N3o HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
```

Вивід кроку 3.4:

```text
root@se-201-17:/lab/work/lego/certificates# cd /lab/work/lego/certificates
openssl x509 -in www.se-201-17.lab.crt -noout -subject -issuer -dates -serial -ext subjectAltName
subject=CN = www.se-201-17.lab
issuer=CN = Pebble Intermediate CA 28a84f
notBefore=Nov 16 17:08:53 2026 GMT
notAfter=Nov 16 17:08:52 2031 GMT
serial=46466156B1C6E7F3
X509v3 Subject Alternative Name: 
    DNS:www.se-201-17.lab, DNS:files.se-201-17.lab
```

Вивід кроку 3.5:

```text
root@se-201-17:/lab/work/lego/certificates# cp www.se-201-17.lab.crt /lab/tls/server.crt
cp www.se-201-17.lab.key /lab/tls/server.key
web-reload

curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-17.lab/
openssl s_client -connect www.se-201-17.lab:443 -servername www.se-201-17.lab -CAfile /lab/tls/acme-root.crt \
  </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/16 17:14:16 [notice] 107#107: signal process started
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
Certificate chain
 0 s:CN = www.se-201-17.lab
   i:CN = Pebble Intermediate CA 28a84f
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 16 17:08:53 2026 GMT; NotAfter: Nov 16 17:08:52 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```

Вивід кроку 3.6:

```text
root@se-201-17:/lab/work/lego/certificates# cd /lab/work/lego/certificates
root@se-201-17:/lab/work/lego/certificates# cat www.se-201-17.lab.crt www.se-201-17.lab.issuer.crt > /lab/tls/server.crt
root@se-201-17:/lab/work/lego/certificates# cp www.se-201-17.lab.key /lab/tls/server.key
root@se-201-17:/lab/work/lego/certificates# web-reload
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/16 17:18:16 [notice] 120#120: signal process started
root@se-201-17:/lab/work/lego/certificates# curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-17.lab/
openssl s_client -connect www.se-201-17.lab:443 -servername www.se-201-17.lab -CAfile /lab/tls/acme-root.crt \
  </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
server: se-201-17 (https)
Host: www.se-201-17.lab
SNI: www.se-201-17.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.se-201-17.lab
   i:CN = Pebble Intermediate CA 28a84f
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 16 17:08:53 2026 GMT; NotAfter: Nov 16 17:08:52 2031 GMT
 1 s:CN = Pebble Intermediate CA 28a84f
   i:CN = Pebble Root CA 54f0fc
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 16 15:53:26 2026 GMT; NotAfter: Nov 16 15:53:26 2056 GMT
---
Verify return code: 0 (ok)
```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера |CN = Pebble Intermediate CA 28a84f |
| Видавець сертифіката видавця |CN = Pebble Root CA 54f0fc |
| Імена в SAN |DNS:www.se-201-17.lab, DNS:files.se-201-17.lab |
| Строк дії (від – до) |Nov 16 17:08:53 2026 GMT - Nov 16 17:08:52 2031 GMT|
| Тип ключа сервера і алгоритм підпису |id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256 |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | У 3.5 - 1, а у 3.6 - 2 |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.

еревірку здійснив центр сертифікації (ACME-сервер Pebble) за методом валідації HTTP-01: клієнт lego розмістив на нашому вебсервері спеціальний токен перевірки, а сервер центру сертифікації успішно звернувся до нього по HTTP за обома доменними іменами (www.se-201-17.lab та files.se-201-17.lab). У журналі виконання команди про це свідчать повідомлення про використання способу перевірки http-01 solver та фінальне підтвердження успішної валідації запиту сервером (The server validated our request) для кожного з доменів.

2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?

Сервер надсилав лише власний кінцевий сертифікат (--no-bundle), але не передавав проміжний сертифікат (Intermediate CA). Через це клієнт curl не зміг самостійно побудувати ланцюжок довіри між кінцевим сертифікатом і довіреним коренем (Root CA), видавши помилку unable to get local issuer certificate.

3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?

id-ecPublicKey, 256 — це тип і параметри асиметричного відкритого ключа самого сервера, в той час як RSA-SHA256 — це алгоритм, за допомогою якого проміжний центр сертифікації (Issuer) засвідчив (підписав своєю цифровою підписом) цей сертифікат.
Вони не суперечать одне одному, оскільки власник ключа може використовувати сучасну криптографію на еліптичних кривих (ECDSA), тоді як центр сертифікації може видати та підписати цей сертифікат за допомогою традиційного алгоритму RSA.

4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?

Після перезапуску стенда внутрішній стан ACME-сервера Pebble (і його бази даних) може скинутися, або видані сертифікати втратяться, якщо вони не зберігаються на персистентному дискові. У реальній інфраструктурі це відповідає ситуації аварійного збою центру сертифікації або втрати системних ключів облікового запису ACME, що вимагає повторної реєстрації та перевипуску сертифікатів.

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text
root@se-201-17:/lab/work/lego/certificates# for i in 1 2 3 4 5; do
  n=f$i.se-201-17.lab; echo "=== $n"
  curl -sS --cacert $STAND_CA https://$n/ 2>&1 | head -1
  openssl s_client -connect $n:443 -servername $n -CAfile $STAND_CA </dev/null 2>/dev/null \
    | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p' | grep -v PKEY
done
=== f1.se-201-17.lab
server: web, f1
Certificate chain
 0 s:CN = f1.se-201-17.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 16 15:53:24 2026 GMT; NotAfter: Feb 14 15:53:24 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 16 15:53:24 2026 GMT; NotAfter: Nov 15 15:53:24 2031 GMT
---
Verify return code: 0 (ok)
=== f2.se-201-17.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f2.se-201-17.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Apr 30 15:53:24 2026 GMT; NotAfter: Oct 27 15:53:24 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 16 15:53:24 2026 GMT; NotAfter: Nov 15 15:53:24 2031 GMT
---
Verify return code: 10 (certificate has expired)
=== f3.se-201-17.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f3.se-201-17.lab'
Certificate chain
 0 s:CN = old.se-201-17.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 16 15:53:24 2026 GMT; NotAfter: Feb 14 15:53:24 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 16 15:53:24 2026 GMT; NotAfter: Nov 15 15:53:24 2031 GMT
---
Verify return code: 0 (ok)
=== f4.se-201-17.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f4.se-201-17.lab
   i:O = Foreign Org, CN = Foreign Intermediate CA
   v:NotBefore: Nov 16 15:53:24 2026 GMT; NotAfter: Feb 14 15:53:24 2027 GMT
 1 s:O = Foreign Org, CN = Foreign Intermediate CA
   i:O = Foreign Org, CN = Foreign Root CA
   v:NotBefore: Nov 16 15:53:24 2026 GMT; NotAfter: Nov 15 15:53:24 2031 GMT
---
Verify return code: 20 (unable to get local issuer certificate)
=== f5.se-201-17.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f5.se-201-17.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 16 15:53:24 2026 GMT; NotAfter: Feb 14 15:53:24 2027 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 | - | 0 (ok) |Пройдені всі перевірки |Verify return code: 0 (ok) |Зміни не потрібні |
| f2 |certificate has expired |10 (certificate has expired) |Прострочений термін дії сертифіката |NotAfter: Oct 27 15:53:24 2026 GMT |Оновити (перевипустити) сертифікат із актуальним терміном дії |
| f3 |no alternative certificate subject name matches target host name | 0 (ok) |Ім'я хоста відсутнє у полі SAN |s:CN = old.se-201-17.lab |Випустити новий сертифікат із правильним ім'ям f3.se-201-17.lab у полі SAN |
| f4 |unable to get local issuer certificate | 20 (unable to get local issuer  certificate) |Видано центром, якому клієнт не довіряє |i:O = Foreign Org, CN = Foreign Intermediate CA |Додати кореневий сертифікат Foreign Root CA у довірені або випустити через OPISM Lab |
| f5 |unable to get local issuer certificate | 21 (unable to verify the first certificate) |Сервер не надсилає проміжний сертифікат |0 s:CN = f5.se-201-17.lab |Налаштувати передачу повного ланцюжка |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?

Якщо сервер не надсилає проміжний сертифікат (як у f5), то у виводі openssl s_client у секції Certificate chain є лише один елемент (індекс 0), а Verify return code дорівнює 21 (unable to verify the first certificate).
А якщо сертифікат виданий недовіреним центром (як у f4), сервер надсилає повний ланцюжок з двох елементів (0 і 1), але емітент (i:O = Foreign Org, CN = Foreign Intermediate CA) відсутній у локальному сховищі довірених сертифікатів ($STAND_CA), через що Verify return code дорівнює 20 (unable to get local issuer certificate).

2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.

Утиліта openssl s_client перевіряє лише криптографічну цілісність ланцюжка, підписи та дати (тому код повернення 0), тоді як бібліотеки зв'язку та curl обов'язково виконують додаткову перевірку відповідності імені хоста (hostname validation) за полями SAN або CN сертифіката. Оскільки для f3 ім'я не збіглося, curl зафіксував помилку невідповідності домену.

3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?

На запит curl -k (який вимикає перевірку сертифікатів) мали б відповісти усі п'ять серверів. При цьому повністю втрачається безпека автентифікації сервера: клієнт не перевіряє справжність вузла, роблячи з'єднання вразливим до атак типу людина посередині (MitM)

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | 1 RTT | 1 RTT | 0 RTT |
| `time_appconnect` | 1 RTT поверх TCP | 2 RTT поверх TCP | 0 RTT |
| `time_starttransfer` | 1 RTT + обробка | 2 RTT + обробка | 0 RTT + обробка |

Вивід кроків 5.2 і 5.3:

```text
root@se-201-17:/lab/work/lego/certificates# F='%{time_namelookup} %{time_connect} %{time_appconnect} %{time_starttransfer} %{time_total} %{http_version} %{num_connects}\n'
U=https://www.se-201-17.lab/
echo "=== TLS 1.3 (New connections) ==="
for i in 1 2 3; do curl -s -o /dev/null --cacert /lab/tls/acme-root.crt -w "$F" $U; done
=== TLS 1.3 (New connections) ===
0.091308 0.091436 0.095102 0.095225 0.095274 1.1 1
0.047114 0.047298 0.053258 0.053816 0.053881 1.1 1
0.048745 0.048890 0.052766 0.052941 0.052995 1.1 1

root@se-201-17:/lab/work/lego/certificates# echo "=== TLS 1.2 (New connections) ==="
for i in 1 2 3; do curl -s -o /dev/null --tls-max 1.2 --cacert /lab/tls/acme-root.crt -w "$F" $U; done
=== TLS 1.2 (New connections) ===
0.091186 0.091353 0.094963 0.095097 0.095147 1.1 1
0.045646 0.045771 0.049479 0.049611 0.049661 1.1 1
0.045615 0.045756 0.049400 0.049539 0.049591 1.1 1

root@se-201-17:/lab/work/lego/certificates# echo "=== Connection Reuse (Keep-Alive) ==="
curl -s --cacert /lab/tls/acme-root.crt -w "$F" -o /dev/null $U -o /dev/null $U -o /dev/null $U
=== Connection Reuse (Keep-Alive) ===
0.045628 0.045756 0.049446 0.049572 0.049621 1.1 1
0.000019 0.000000 0.000000 0.000119 0.000147 1.1 0
0.000020 0.000000 0.000000 0.000096 0.000113 1.1 0
```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | 0.047114 | 0.047298 | 0.053258 | 0.053816 | 0.053881 | 1 |
| TLS 1.2, нове | 0.045646 | 0.045771 | 0.049479 | 0.049611 | 0.049661 | 1 |
| 5.3, перший запит | 0.045628 | 0.045756 | 0.049446 | 0.049572 | 0.049621 | 1 |
| 5.3, другий запит | 0.000019 | 0.000000 | 0.000000 | 0.000119 | 0.000147 | 0 |
| 5.3, третій запит | 0.000020 | 0.000000 | 0.000000 | 0.000096 | 0.000113 | 0 |

RTT, визначений вимірюванням, і з якого значення його визначено:

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?

Час connect відображає тривалість TCP 3-way handshake (приблизно 1 RTT). Час appconnect для TLS 1.3 додає ще близько 1 RTT, а для TLS 1.2 — 2 RTT. Вимірювання дещо відрізняються від ідеальних цілих чисел через локальні системні затримки операційної системи на обробку стеків, які у локальному середовищі переважають саму сетеву затримку распространения пакетов.

2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?

Навіть при кешуванні звернення до системного резолвера усередині бібліотеки curl викликає системні виклики та ініціалізацію структур даних. Воно було б приблизно вдвічі більшим за умов відсутності локального кешу, коли б запит реальним трафіком надсилався до зовнішнього DNS-сервера (BIND9) через мережу.

3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.

При новому з'єднанні TLS 1.3 для кожного з 200 запитів витрачається повний цикл встановлення з'єднання (в середньому 53 мс на запит): 200 * 53 мс  = приблизно 10.6 с.При новому з'єднанні TLS 1.2 час буде дещо вищим через додатковий обмін пакетами рукопожаття (50 мс на запит): 200 * 50 мс = приблизно 10.0 с (або більше залежно від реального RTT мережі).При одному спільному з'єднанні (Connection Reuse / Keep-Alive) накладні витрати на встановлення з'єднання відбуваються лише один раз для першого запиту (50 мс), а наступні 199 запитів виконуються миттєво (0.14 мс на запит): 50 мс + (199 * 0.14 мс) = приблизно 50 + 28 = приблизно 78мс (менше 0.1 секунди).

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно: Несподіваним було те, що в Завданні 4.1 для сервера f3.se-201-17.lab утиліта openssl s_client повернула Verify return code: 0 (ok), хоча curl відхилив з'єднання через невідповідність імені хоста.



**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося: Залишилося нез'ясованим, чому під час вимірювань у Завданні 5 (крок 5.2) найперший запит у кожному циклі (і для TLS 1.3, і для TLS 1.2) мав час time_namelookup близько 0.091 с (майже вдвічі довший за наступні два запити по ~0.046 с).

## Декларація використання ШІ

Використовувалися інструменти ШІ: так

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | Gemini | Перевір, чи правильно заповнено таблицю записів TLS 1.3 за виводом openssl s_client -trace | Проведено перевірку порядку записів у таблиці до Завдання 2.1 та уточнено відповідь щодо розширення key_share. |
| 2 | Gemini | Поясни різницю між кодами помилок openssl verify 20 та 21 при перевірці ланцюжка сертифікатів | Уточнено діагностичні ознаки несправностей для Завдання 4.2. Усі команди перевірки виконано самостійно на стенді, результати виводу зведено в таблицю та інтерпретовано для кожного вузла. |
| 3 | Gemini | Як коректно розрахувати медіанний час та виразити етапи TLS-рукостискання через RTT | Використано методику переведення секунд у мілісекунди та прив'язки метрик time_connect і time_appconnect до циклів RTT. Усі заміри виконано власноруч, розрахунки для 200 запитів та висновки написано самостійно. |
| 4 | Gemini | Чому в параметрах сертифіката завдання 3 одночасно вказано PKEY: id-ecPublicKey та sigalg: RSA-SHA256? | Пояснено різницю між типом відкритого ключа сервера та алгоритмом підпису центру сертифікації для відповіді на питання 3.3. |
