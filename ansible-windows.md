Настройка управления windows 10 и windows server 2012 для управления через ansible с помощью ssh из debian 12: 

Если хост с Debian новый то для установки ansible выполняем:
```
sudo apt update
sudo apt install python3 python3-pip
python3 -m pip install --upgrade pip
python3 -m pip install ansible --break-system-packages
```

На хосте с Debian 12 и ansible выполняем:
```
ssh-keygen
```
Жмем 3 раза enter и далее выполняем
```
cat /USER_NAME/.ssh/id_rsa.pub
```
Сохраняем где то выведенные ключ  

Нам windows машине, которой мы хотим управлять нужно установить OpenSSH-Win64.zip, для его надо перейти по ссылке:  
https://github.com/PowerShell/Win32-OpenSSH/releases  
Качаем только файл OpenSSH-Win64.zip из последего релиза

Далее его нужно разархивиировать обязательно в папку с названием OpenSSH и поместить на диск C, путь должен быть таким  
`C:\OpenSSH`
Ниже будет удобный скрипт для установки всего нужного, поэтому это обязательно

Далее открываем Powershell от администратора, если у нас Windows 10, перед выполнением скрипта нужно выполнить команду ниже, иначе будет ошибка:  
`Set-ExecutionPolicy RemoteSigned -Scope CurrentUser`

Далее идет скрипт для установки ssh-server, заменяем поле "ВАШ КЛЮЧ", на ранее выведенный публичный ключ, и копируем все команды вместе и вставляем  
в powershell от имени админа,(можно просто по одной команде поочередно вводить):
```
cd C:\OpenSSH    
.\install-sshd.ps1
Start-Service sshd
Start-Service ssh-agent
Set-Service -Name sshd -StartupType Automatic
Set-Service -Name ssh-agent -StartupType Automatic
Get-Service sshd
Get-Service ssh-agent
New-Item -Path "C:\ProgramData\ssh\administrators_authorized_keys" -ItemType File
Add-Content -Path "C:\ProgramData\ssh\administrators_authorized_keys" -Value "ВАШ SSH КЛЮЧ"
New-NetFirewallRule -Protocol TCP -LocalPort 22 -Direction Inbound -Action Allow -DisplayName SSH
ipconfig /all
```
Вместо `"ВАШ SSH КЛЮЧ"` вставишь ваш публичный ключ, с хоста, который будет управлять через ansible этой windows машиной  

Сам скрипт: 
Устанавливает ssh и sshd.  
Ставит автоматическое включение этих приложений при запуске пк.  
Включает их сразу, чтобы не надо было перезагружаться.  
Создает разрешающее правило для брандауэра для входящего трафика на 22 порт. 
Вставляет ваш публичный ключ в administrators_authorized_keys, потому что ssh-copy-id с хоста с ansible не работает почему-то.   

В конце выводит сетевые настройки, чтобы можно было нужные данные записать в /etc/ansible/hosts на управляющем хостем. 

Пример файлика hosts, 
```
srv-rds1 ansible_host=192.168.11.66 #тут поочередно указываем все машины с Windows по моему примеру
[windows] #тут обьеденение их в группу, для одновременного управления несколькими
srv-rds1 
[windows:vars] #к каждой группе из windows машин добавляем эти аргументы, для Linux групп это не надо
ansible_user=tsuran
ansible_shell_type = powershell
shell_type = powershell
[all:vars] 
ansible_ssh_private_key_file=/root/.ssh/id_rsa #путь к приватному ключу
ansible_python_interpreter=/usr/bin/python3 
```



