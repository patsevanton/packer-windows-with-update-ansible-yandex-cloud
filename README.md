Зачем собирать образ с помощью [Packer](https://www.packer.io/)?

1. Время создания инстанса из готового образа значительно меньше, чем время, которое нужно затратить с нуля на подготовку виртуальной машины к работе. Это достаточно критичный момент, так как порой очень важно ввести в работу новый инстанс нужного типа за кратчайшее время для того, чтобы начать пускать на него трафик.
2. Помимо того что образ виртуальной машины для DEV, TEST, Staging  окружений, он всегда будет соответствовать по набору ПО и его настройкам тому серверу, который используется в production. Важность этого момента трудно недооценить — крайне желательно, чтобы деплой нового кода на продакшн привел к тому, чтобы сайт продолжал корректную работу с новой функциональностью, а не упал из-за какой-то ошибки, связанной с недостающим модулем или отсутствующим ПО.
3. Автоматизация сборки production- и development-окружений экономит время системного администратора. В глазах работодателя это также должно быть несомненным плюсом, так как это означает, что за то же время администратор сможет выполнить больший объем работы.
4. Время для тестирования набора ПО, его версий, его настроек. Когда мы подготавливаем новый образ заранее, у нас есть возможность (и, что самое главное, время!) для того, чтобы спокойно и вдумчиво проанализировать различные ошибки, которые возникли при сборке образа, и исправить их. Также есть время для тестирования работы приложения на собранном образе и внесения каких-то настроек для оптимизации приложений. В случае же, если мы настраиваем инстанс, который нужно было ввести в работу еще вчера, все возникающие ошибки, как правило, исправляются по факту их возникновения уже на работающей системе — конечно же, это не совсем правильный подход.

<cut />

Более подробно можете прочитать про packer [здесь](https://xakep.ru/2014/10/08/using-packer/).

В процессе сборки запускается [Ansible](https://github.com/ansible/ansible), который устанавливает обновления Windows. С помощью ansible гораздо проще управлять установкой приложений, настройкой Windows чем Powershell. Подробнее про использование ansible и windows [здесь](https://habr.com/ru/company/veeam/blog/455604/).

В посте описаны ошибки, которые вы можете получить, и их решение.

 Готовый работе репозиторий: https://github.com/patsevanton/packer-windows-with-update-ansible-yandex-cloud

#### Установка необходимых пакетов

```
sudo apt update
sudo apt install git jq python3-pip -y
```

#### Устанавливаем ansible pywinrm

```
sudo pip3 install ansible pywinrm
```
#### Устанавливаем коллекцию ansible.windows

```
ansible-galaxy collection install ansible.windows
```

#### Установка Packer.

Необходимо установить Packer по инструкции с официального сайта https://learn.hashicorp.com/tutorials/packer/get-started-install-cli

#### Установка Yandex.Cloud CLI

https://cloud.yandex.com/en/docs/cli/quickstart#install

```
curl https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
```

#### Загружаем обновленные настройки .bashrc без выхода из системы

```
source ~/.bashrc
```

#### Инициализация Yandex.Cloud CLI

```
yc init
```

#### Клонируем репозиторий с исходным кодом

```
git clone https://github.com/patsevanton/packer-windows-with-update-ansible-yandex-cloud.git
cd packer-windows-with-update-ansible-yandex-cloud
```

#### Скачиваем ConfigureRemotingForAnsible.ps1

```
wget https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1
```

#### Создайте сервисный аккаунт и передайте его идентификатор в переменную окружения, выполнив команды

```
yc iam service-account create --name <имя пользователя>
yc iam key create --service-account-name <имя пользователя> -o service-account.json
SERVICE_ACCOUNT_ID=$(yc iam service-account get --name <имя пользователя> --format json | jq -r .id)
```

В документации к Yandex Cloud параметр folder_id описывается как <имя_каталога>.

Получите folder_id из `yc config list`

Назначьте сервисному аккаунту роль admin в каталоге, где будут выполняться операции:
```
yc resource-manager folder add-access-binding <folder_id> --role admin --subject serviceAccount:$SERVICE_ACCOUNT_ID
```

Заполняем файл credentials.json.
```
    "folder_id": "<folder_id>",
    "password": "<Пароль для Windows>"
```

#### Заполняем credentials.json

#### Запускаем сборку образа

```
packer build -var-file credentials.json windows-ansible.json
```

Готовый образ можно будет найти в сервисе **Compute Cloud** на вкладке **Образы**.

Через Yandex Cloud CLI будет виден вот так.

```
yc compute image list
+----------------------+-------------------------------------+--------+----------------------+--------+
|          ID          |                NAME                 | FAMILY |     PRODUCT IDS      | STATUS |
+----------------------+-------------------------------------+--------+----------------------+--------+
| fd88ak25ivc8rn1e4bn7 | windows-server-2021-09-20t04-47-57z |        | f2e01pla7dr1fpbeihp9 | READY  |
+----------------------+-------------------------------------+--------+----------------------+--------+
```

### Описание кода в user-data

При сборке образа сначала выполняется скрипты, описанные в user-data:
Установка пароля пользователя Administrator
```
net user Administrator {{user `password`}}
```

Удаление скриптов из директории C:\Program Files\Cloudbase Solutions\Cloudbase-Init\LocalScripts\
```
ls \"C:\\Program Files\\Cloudbase Solutions\\Cloudbase-Init\\LocalScripts\" | rm
```

Удаляет доверенные хосты службы WS-Management по адресу \\Localhost\\listener\\listener*
```
Remove-Item -Path WSMan:\\Localhost\\listener\\listener* -Recurse
```

Удаляет сертификаты с локального сервера
```
Remove-Item -Path Cert:\\LocalMachine\\My\\*
```

Получает DNS имя из службы метаданных
```
$DnsName = Invoke-RestMethod -Headers @{\"Metadata-Flavor\"=\"Google\"} \"http://169.254.169.254/computeMetadata/v1/instance/hostname\"
```

Получает hostname имя из службы метаданных
```
$HostName = Invoke-RestMethod -Headers @{\"Metadata-Flavor\"=\"Google\"} \"http://169.254.169.254/computeMetadata/v1/instance/name\"
```

Получает сертификат из службы метаданных
```
$Certificate = New-SelfSignedCertificate -CertStoreLocation Cert:\\LocalMachine\\My -DnsName $DnsName -Subject $HostName
```

Создает новое значение `HTTP` в службе WS-Management по адресу \\LocalHost\\Listener c параметром `Address *`
```
New-Item -Path WSMan:\\LocalHost\\Listener -Transport HTTP -Address * -Force
```

Создает новое значение `HTTPS` в службе WS-Management по адресу \\LocalHost\\Listener c параметром `Address *`, HostName и Certificate
```
New-Item -Path WSMan:\\LocalHost\\Listener -Transport HTTPS -Address * -Force -HostName $HostName -CertificateThumbPrint $Certificate.Thumbprint
```

Добавляет в firewall правило разрешающее трафик на порт 5986
```
netsh advfirewall firewall add rule name=\"WINRM-HTTPS-In-TCP\" protocol=TCP dir=in localport=5986 action=allow profile=any
```

### Тестирование и отладка
Для отладки запускаем packer с опцией debug

```
packer build -debug -var-file credentials.json windows-ansible.json
```

Но так как опция `-debug` требует потверждения на каждый шаг, для отладки добавляем конструкцию после шага, который нужно отладить, например, после запуска ansible.
```
{
    "type": "shell",
    "inline": [
        "sleep 9999999"
    ]
},
```

Отредактируем файл ansible/inventory. Проверим win_ping.

```
ansible windows -i ansible/test-inventory -m win_ping
```

Тестируем установку обновлений Windows без Packer
```
ansible-playbook -i ansible/test-inventory ansible/playbook.yml
```

### Ошибки

Если не запустить ConfigureRemotingForAnsible.ps1 на Windows, то будет такая ошибка
```
basic: the specified credentials were rejected by the server
```

Если забыли установить "use_proxy" в false, то будет такая ошибка
```
basic: HTTPSConnectionPool(host=''127.0.0.1'', port=5986): Max retries exceeded with url: /wsman (Caused by NewConnectionError(''<urllib3.connection.VerifiedHTTPSConnection object at 0x7f555c2d07c0>: Failed to establish a new connection: [Errno 111] Connection refused''))
```

Если в таске `Ensure the local user Administrator has the password specified for TEST\Administrator` будет ошибка 
```
requests.exceptions.HTTPError: 401 Client Error:  for url: https://xxx:5986/wsman
...
winrm.exceptions.InvalidCredentialsError: the specified credentials were rejected by the server
```
То вы забыли синхронизировать пароли в переменных `pdc_administrator_password` и `ansible_password`

Временный ansible inventory с именем /tmp/packer-provisioner-ansiblexxxx будет иметь вот такой вид:
```
default ansible_host=xxx.xxx.xxx.xxx ansible_connection=winrm ansible_winrm_transport=basic ansible_shell_type=powershell ansible_user=Administrator ansible_port=5986
```


### Тестирование сборки Windows c Active Directory

Так же тестировал сборку Windows c Active Directory, используя Galaxy роль `justin_p.pdc`, но при запуске нового instance из собранного образа получал ошибку.

```
2021-09-18 19:34:31, Error SYSPRP ActionPlatform::LaunchModule: Failure occurred while executing 'CryptoSysPrep_Specialize' from C:\Windows\system32\capisp.dll; dwRet = 0x32
2021-09-18 19:34:31, Error SYSPRP SysprepSession::ExecuteAction: Failed during sysprepModule operation; dwRet = 0x32
2021-09-18 19:34:31, Error SYSPRP SysprepSession::ExecuteInternal: Error in executing action for Microsoft-Windows-Cryptography; dwRet = 0x32
2021-09-18 19:34:31, Error SYSPRP SysprepSession::Execute: Error in executing actions from C:\Windows\System32\Sysprep\ActionFiles\Specialize. xml; dwRet = 0x32
2021-09-18 19:34:31, Error SYSPRP RunPlatformActions:Failed while executing Sysprep session actions; dwRet = 0x32
2021-09-18 19:34:31, Error [0x060435] IBS Callback_Specialize: An error occurred while either deciding if we need to specialize or while specializing; dwRet = 0x32
```

![](https://habrastorage.org/webt/yp/tm/ik/yptmiktn4i2qoystu4_deevbkuc.png)
