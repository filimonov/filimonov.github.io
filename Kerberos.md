%Kerberos and Clickhouse
%Mikhail Filimonov <mfilimonov@altinity.com>
%March 24, 2020

<!--
https://pandoc.org/
pandoc -s Kerberos.md -t slidy -o Kerberos.html
-->


# Kerberos
![](http://web.mit.edu/KERBEROS/images/dog-ring.jpg)

* SSO придуманный в 1988 в MIT.
* Широко используется в крупных фирмах (безопасность!)
* Последние 25 лет - версия 5, ака krb5
* Описан в RFC4120 (до 2005 года - RFC1510)
* Microsoft также способствовали популяризации, начав использовать Kerberos в AD в Windows 2000.

# Краткое описание. Решаемая задача

* Пользователю нужно логиниться на разные сервисы.

* Пользователей в организации много, сервисов тоже

* Пользователь не хочет знать миллион паролей, вводить пароли снова и снова, или хранить их
  в небезопасных местах.

* Админ хочет управлять доступами из одного места (а не
  конфигурировать каждый сервис отдельно); не хочет
  заводить отдельные аккаунты всем везде и всюду

* Одной рабочей станцией могут пользоваться разные люди.

* Сеть потенциально не надёжна.

# Краткое описание. Терминология

* Key Distribution Center (KDC) - центральный сервер, который знает все сервисы и всех пользователей

* realm - все участники организации живут в одном "королевстве" (realm), обычно реалм = имя домена в капсе, типа YANDEX.RU.
 В больших оганизациях может быть несколько реалмов (CLICKHOUSE.YANDEX.RU, MARKETING.YANDEX.RU), KDC которых могут между собой как-то общаться

* И пользователи и серверы - называются принципалы, у каждого есть уникальное имя - строковой идентификатор (UPN milovidov@yandex.ru, SPN http/www-server.yandex.ru) и общий с KDC секрет (пароль в случае пользователя, keytab-файл в случае сервиса).

# Краткое описание. Протокол.

* Пользователь логинится на KDC один раз за сессию (при логине на станцию или с помощью команды kinit) и получает билет для получения других билетов (TGT=ticket granting ticket), и хранит его пока он действителен (или до logout / kdestroy)

* Потом пользуясь этим билетом он может просить билеты на всевозможные сервисы - почта, веб, базы данных и т.д.

* Сервер KDC получает запросы типа "пользователь с принципалом UPN u и с билетом TGT t хочет билет чтобы воспользоваться сервисом с принципалом SPN s". После чего даёт (или не даёт) пользователю билет, позволяющий воспользоваться конкретным сервисом.

* Пользователь идет с этим билетом в сервис и получает доступ.

* Никакой другой пользователь (с другого хоста, перехвативший трафик и т.п.) гарантированно не сможет этим воспользоваться - шифрование, взаимная аутентификация, сессии, expiration и т.д. и т.п.

![](https://upload.wikimedia.org/wikipedia/commons/4/4e/Kerberos.svg)

# Краткое описание. Дополнительные требования накладываемые на систему.
* Требует правильных и разрешающихся (предпочтительно - в обе стороны) доменных имён (у всех участников - и у пользователей и у сервисов)
  если ты предъявишь билет полученный на другом хосте - то не пустят.
* Требует синхронизации часов
  с просроченным билетом не пустят
* Шифрование на уровне канала не обязательно (билетики уже зашифрованы), но КТО логинится может быть видно.

# vs LDAP:
* LDAP authentication is centralized authentication, meaning **you have to login with every service**, but if you change your password it changes everywhere.
* Kerberos is single sign-on (SSO), meaning you login once and get a token and don't need to login to other services.
* LDAP is less convenient but simpler.
* Kerberos is more convenient but more complex.
From: https://security.stackexchange.com/a/109574

# vs LDAP 2:
* При LDAP пароли не будут хранится на КХ, но логиниться нужно каждый раз заново.
* Kerberos защищает пользователя от всевозможных атак, LDAP сам по себе никаких защит не предоставляет (там можно открытым текстом пароли слать), он просто позволяет проверить - соответствует ли логин паролю.
* Kerberos сам по себе описывает только процедуру/протокол аутентификации, но не описывает как KDC должен хранить пароли и пользователей. В простых сценариях можно всех пользователей и сервисы запихать в простой текстовый файлик. По мере роста - появляются всякие всякие сложные ACL, роли и т.п. Хранить всё это в текстовом файлике - становится очень утомительно, и нужна база данных которая это будет хранить
* Поэтому довольно часто Kerberos используют **в сочетании** с LDAP: https://help.ubuntu.com/lts/serverguide/kerberos-ldap.html
  Типичный пример: Active Directory: the usernames and passwords the Kerberos authentication server checks are actually stored in a LDAP directory.

# vs X.509 сертификаты:
* В x.509 при использовании пользовательских сертификатов аутентификация встроена в транспортный уровень (SSL / TSL)
* это удобно и классно, нужно будет поддержать.

# Софт:
* Под *никсами есть 2 сервера KDC: mit vs heimdal (первый раньше имел экспортные ограничения, сейчас всё ок)
* На проде чаще всего люди пользуются Windows Active Directory, который в том числе умеет работать как KDC.

# Базы которые уже умеют
* MsSQL, Oracle, Postgres, MySQL и т.д., короче примерно все mature
  * https://mariadb.com/kb/en/authentication-plugin-gssapi/

# Что нужно пользователям (запрос от крупной security-oriented фирмы):
For kerberos as a first pass:

* all we need is to auth the username against Kerberos and not have to store a password in a config and the user having to store it in all kinds of unsafe locations; no need for role mapping via LDAP etc. (unless it’s really easy to do)

* We’d need the HTTP server in clickhouse to do Kerberos auth - most users start with JDBC driver or http drivers

* If it’s easy to add into the native protocol it would be very useful as well.

* It would be awesome if clickhouse-client would be able to auth using the current kerberos principal (or have an option to choose the principal). Same for clickhouse-cpp.

* As a first pass, we’re fine with adding users to the configs manually (or some scripts) and set a flag to say kerberos instead of sha256_sum for the auth part (we need support **for both options** at the same time)

* We’re fine defining the permissions using the current config format.

# На когда:
::: incremental
* Конец марта.
* ну по-крайней мере какой-то прогресс надо показать. :)
:::

# Декомпозиция задачи
1. создание спецификации (in progress)
2. server-side поддержка в протоколе HTTP (тесты с помощью curl)
3. server-side и client-side поддержка в нативном протоколе (обратная совместимость!)
4. never-ending процесс "добавить поддержку Kerberos для клиента ХХХХ"

# Дополнительные требования к КХ
5. Несколько AD пользователей = один пользователь БД, но видимо нужно знать, кто именно это был (query log и т.п.)
6. возможность ограничить макс. время жизни сессии (после какого-то таймаута - нужен релогин)
7. Delegation Token. Есть возможность "передавать билет дальше" (пока out of scope).
   Т.е. если пользователь во время пользования КХ решит сходить в mysql, то там есть какие-то возможности "передавать билетики".
   Кликхаус также потенциально должен уметь работать в таком режиме, например для Табло
   https://help.tableau.com/current/server/en-us/kerberos_delegation.htm
8. когда-нибудь можно поддержать и для mysql port


# Kerberos в HTTP
* Есть несколько подходов, все не безупречны и имеют [свои недостатки](http://people.apache.org/~jorton/ac08eu/kerb-sso-http.pdf)
* Самый широко используемый, и имеющий собственное RFC: WWW-Authenticate: Negotiate
    * http://tools.ietf.org/html/rfc4559
    * базирутеся на SPNEGO https://tools.ietf.org/html/rfc4178
    * изобретён в недрах Microsoft для IE / IISc в 2005
    * но как-то прижилось и сейчас его умеют более-менее все.

# клиенты, умеющие http/SPNEGO (rfc4559)
* chrome https://www.chromium.org/developers/design-documents/http-authentication
* firefox https://developer.mozilla.org/en-US/docs/Mozilla/Integrated_authentication
* safari https://support.pingidentity.com/s/article/How-to-configure-supported-browsers-for-Kerberos-NTLM
* IE
* curl (ура, можно будет сделать простые тесты)
* поддержка в том или ином виде есть примерно для любого ЯП (хотя примерно всегда это сторонний модуль).

# серверы, умеющие http/SPNEGO (rfc4559)
* IIS: https://techcommunity.microsoft.com/t5/iis-support-blog/setting-up-kerberos-authentication-for-a-website-in-iis/ba-p/347882
* Apache: http://www.microhowto.info/howto/configure_apache_to_use_kerberos_authentication.html
* nginx: ngx_http_auth_spnego_module https://github.com/stnoonan/spnego-http-auth-nginx-module/blob/master/ngx_http_auth_spnego_module.c
* короче референс имплементации тоже известны

# С перспективы кода C++:
::: incremental

* написать свою собственную, векторную, MPP, SIMD-оптимизированную реализацию Kerberos.

* воспользоваться кодом MIT  `#include <krb5.h>`

* воспользоваться GSSAPI (SPNEGO и rfc4559 основаны на нём) `#include <gssapi/gssapi.h>`

* воспользоваться sasl  `#include <sasl.h>`
    * который включает GSSAPI
    * который уже есть у нас в contrib (может быть wrapping w SPNEGO надо будет сделать самим)
    * по этой дороге пошли hadoop, kafka, и др.
    * Потенциально позволяет расширять методы аутентификации до любых других поддерживаемых в sasl
    * готовые биндинги sasl существуют для многих ЯП (писателям клиентов будет проще)
:::

# ClickHouse. Как конфигурировать
* хотим избежать необходимости создавать всех пользователей из AD в КХ
* поэтому кажется логичным подход по типу MySQL
```sql
    CREATE USER sql_admin
      IDENTIFIED WITH authentication_windows
        AS 'user@yandex.ru,user2@yandex.ru';
```
* в конфиге могло бы быть что-то типа
   * в config.xml
```xml
  <sasl>
    <gssapi_keytab_path>/etc/clickhouse-server/clickhhouse.keytab</gssapi_keytab_path>
    <!-- If the value is '*', the web server will attempt to login with every principal specified in the keytab file -->
    <gssapi_principal_name>clickhouse/server-123@YANDEX.RU</gssapi_principal_name>
    <!-- The SPNEGO server principal begins with the prefix HTTP/ by convention  -->
    <gssapi_http_principal_name>HTTP/server-123@YANDEX.RU</gssapi_http_principal_name>
  </sasl>
```
   * в users.xml
```xml
   <default>
      <sasl_authentiaction></sasl_authentiaction> <!-- allow to use sasl for that user, every logged user will be default -->
      <!-- or -->
      <sasl_authentiaction>
        <gssapi_upn>user1@ALTINITY.COM</gssapi_upn><!-- map those users to default -->
        <gssapi_upn>user2@ALTINITY.COM</gssapi_upn>
      </sasl_authentiaction>
   </default>
```
   * что-то ещё?

# ClickHouse. Поддержка в HTTP.
::: incremental

* создание тестового стенда с reference реализацией (nginx поверх КХ), и тестами из curl и/или из питона
* инициализация объекта sasl (`sasl_server_init`) - насколько я понимаю нужет один на всё приложение (conditionally? context? global?)
* если не прислал ничего про авторизацию - отвечает 401 плюс доп. заголовок `WWW-Authenticate: Negotiate`
* если клиент продолжает общение то там должно быть `Authorization: Negotiate spnego_encoded_string`
* декодируем spnego_encoded_string (base64 decode и маппинг в структуру) - проверяем что клиент выбрал что-то поддерживаемое, извлекаем payload
  делаем `sasl_server_new` и передаём ему payload
* ищем какому пользователю БД соответствует залогинившийся и пускаем/не пускаем.
* вроде должно работать (finger crossed), пробуем наш тест, главный вопрос - SPNEGO vs sasl, обратная совместимость c Basic
* когда-нибудь можно будет поддержать другие варианты SPNEGO

:::

# ClickHouse. Поддержка в HTTP. Sample implementations
* https://github.com/stnoonan/spnego-http-auth-nginx-module/blob/master/ngx_http_auth_spnego_module.c
* https://github.com/modauthgssapi/mod_auth_gssapi/blob/master/src/mod_auth_gssapi.c
* https://github.com/samba-team/samba/blob/master/source3/libads/sasl_wrapping.c
* https://github.com/cyrusimap/cyrus-sasl/blob/master/sample/sample-server.c
* https://github.com/cyrusimap/cyrus-sasl/blob/master/sample/http_digest_client.c

# ClickHouse. Поддержка в Native протоколе.
::: incremental

* сейчас имя пользователя и пароль передаются в открытом виде в hello пакете
   * https://github.com/ClickHouse/ClickHouse/blob/fbb51a5a19eb53e85de79a88334e295c21504436/dbms/src/Client/Connection.cpp#L168-L169
   * https://github.com/ClickHouse/ClickHouse/blob/800026c55ef025bb0f7d9ec6d2271780463a1b62/dbms/programs/server/TCPHandler.cpp#L799-L800
* структуру hello пакета лучше не менять (обратная совместимость)
* если клиент хочет работать как раньше - то делает всё как раньше.
* если клиент хочет/готов воспользоваться sasl - то отправляет специальные значение в полях пользователя и пароля (например пустое имя пользователя, и sasl в качестве пароля)
* тогда сервер проводит обычные раунды аутентификации с клиентом - выбор протокола аутентификации, и т.д. Все в соответствии с sasl (можно сразу поддержать какой-то доп. способ аутентификации, например Digest)
* если пользователь пытается отправить имя пользователя и пароль, при этом для него включена **только** sasl аутентификация - то отвечает специальным статусом, типа authentication type rejected
* потребуется немного доп. конфигурации по стороне клиента (TBD).
* если делать всё правильно - то пароль не надо хранить в памяти (обнулять переменные и т.п.) но эта задача на втором плане.

:::

# ClickHouse. Поддержка в Native протоколе. sample implementations
* https://github.com/cyrusimap/cyrus-sasl/blob/master/sample/sample-server.c


# ClickHouse. Альтернативный вариант
* можно предложить аутентификацию с использованием LDAP (проще)


# References. Kerberos
### Почитать по-русски:
* http://www.javacogito.net/index.php/%D0%91%D0%B5%D0%B7%D0%BE%D0%BF%D0%B0%D1%81%D0%BD%D0%BE%D1%81%D1%82%D1%8C_%D0%B2_Hadoop._Kerberos
* https://www.intuit.ru/studies/courses/531/387/lecture/8998
* https://pro-ldap.ru/

### Объяснение принципов и мотивов реализации (в виде диалога создателей)
* http://web.mit.edu/Kerberos/dialogue.html

### По английски:
* https://www.roguelynn.com/words/explain-like-im-5-kerberos/
* https://help.ubuntu.com/lts/serverguide/kerberos.html
* https://help.ubuntu.com/community/Kerberos

### Официальный сайт:
* https://web.mit.edu/kerberos/

### Возможные атаки
* https://www.varonis.com/blog/kerberos-authentication-explained/

# References. Книжка с животным
![](https://covers.oreillystatic.com/images/9780596004033/lrg.jpg)

# Other unsorted sources

### Nice step by step
* https://wiki.zimbra.com/wiki/Running_Kerberos_with_Zimbra_Collaboration_Suite
* https://inbo.github.io/tutorials/installation/user/user_install_kerberos/

### Java world. HTTP/SPNEGO Authentication
* https://docs.oracle.com/javase/10/security/part-vi-http-spnego-authentication.htm#JSSEC-GUID-996F729E-5FEA-4E29-A32A-78FB510B2D80

### Lower-lever SPNEGO roundtrip
* https://medium.com/@robert.broeckelmann/kerberos-wireshark-captures-a-spnego-example-e22e6b1d662a

# Other unsorted sources. Docker (for tests)
* https://hub.docker.com/r/godatadriven/krb5-kdc-server/
* https://github.com/nirko-rnd/Docker.NGINX-Kerberos
* https://github.com/criteo/kerberos-docker/blob/master/build/krb5-centos/kdc-server/Dockerfile.template
* https://hub.docker.com/r/sequenceiq/kerberos/
* https://github.com/ist-dsi/docker-kerberos
* https://github.com/jcmturner/gokrb5-test/tree/master/testenv

