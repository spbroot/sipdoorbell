

# Добавление SIP домофона в Homekit c двухсторонним аудио

Данный пример описывает добавление аудио SIP домофона в Homekit

Используется Rasberry Pi 4 c установленным образом Homebridge

Управление домофоном осуществляется через DTMF команды

* Внесите изменения в параметры системы
  
  ```
  формат времени 24-х часовой
  формат даты ДД/ММ/ГГГГ
  ```
  
* Добавление звукового устройства ALSA Loopback

  ```
  sudo su
  echo 'snd-aloop' >> /etc/modules
  exit
  ```

* Вносим изменения в конфигурацию ALSA

  Создаем или редактируем файл /etc/asound.conf

  ```
  # output device
  pcm.loop_out {
    type dmix
    ipc_key 100001
    slave.pcm "hw:Loopback,0,0"
  }

  # input device
  pcm.loop_in {
    type dsnoop
    ipc_key 100002
    slave.pcm "hw:Loopback,1,1"
  }

  # plug device
  pcm.sipdoorbell_return {
    type plug
    slave.pcm 'loop_out'
    hint {
      show on
      description 'sipdoorbell return channel'
    }
  }

  # plug device
  pcm.sipdoorbell_main {
    type plug
    slave.pcm loop_in
    hint {
      show on
      description 'sipdoorbell main channel'
    }
  }
  ```

    _!!! Если при открытии домофона на iphone/ipad/AppleTV в homebridge останавливается FFmpeg вот с такой или похожей ошибкой:_

    ```
    [error] cannot open audio device sipdoorbell_main (No such file or directory)
    [error] sipdoorbell_main: Input/output error
    ```

    _или_

    ```
    [error] cannot open audio device sipdoorbell_return (No such file or directory)
    [error] sipdoorbell_return: Input/output error
    ```

    _размещаем это в конце ```/usr/share/alsa/alsa.conf``` вместо ```/etc/asound.conf```_
    
    _Не очень правильно, но на некоторых системах, по непонятной мне причине, FFmpeg не видит устройств описанных в ```/etc/asound.conf``` (при таком поведении выдача ```ffmpeg -sources alsa``` не содержит описанных в ```/etc/asound.conf``` или ```~/.asoundrc``` устройств)_

* Устанавливаем baresip

  ```
  sudo apt install baresip
  ```

* Запускаем baresip для создания конфигурационного файла, после запуска жмём 'q' для выхода из baresip
* Вносим изменения в файл конфигурации baresip: файл /home/pi/.baresip/config

  Снимаем комментарий со следующих строк (если закомментированы) и вносим изменения

  ```
  sip_listen              0.0.0.0:5060

  audio_player            alsa,hw:Loopback,0,1
  audio_source            alsa,hw:Loopback,1,0
  audio_alert             alsa,hw:Loopback,0,1

  ausrc_srate             48000
  auplay_srate            48000
  ausrc_channels          2
  auplay_channels         2

  module                  httpd.so

  http_listen             0.0.0.0:8000
  ```

* В файл /home/pi/.baresip/accounts добавляем SIP аккаунт

  * При использовании P2P SIP вызова
  
    При использовании P2P, вызов должен осуществояться в формате "100@<homebridge_ip_address>"

    ```
    <sip:100@127.0.0.1>;auth_pass=none;regint=0;
    ```

  * При использовании SIP регистрации

    ```
    <sip:extention_number@sip_server_ip>;auth_pass=account_password;
    ```

* В Homebridge устанавливаем плагин Homebridge Camera FFmpeg

* В плагине Homebridge Camera FFmpeg добавляем камеру

  _Cameras_

    ``` 
    Name: Doorbell
    Video Source: "-re -stream_loop -1 -i /opt/sipdoorbell/homekit.avi -f alsa -ac 2 -ar 48000 -i sipdoorbell_main"
    Still Image Source: "-i /opt/sipdoorbell/homekit.jpg"
    Enable audio: checked
    ```

  _Advanced_
  _EXPERIMENTAL - WIP_

    ```
    Two-way Audio Target (EXPERIMENTAL - WIP): "-filter:a volume=12dB -f alsa -ac 2 -ar 48000 sipdoorbell_return"
    Two-way FFmpeg Debug Logging: checked
    Audio Stream: "1:a"
    Enable doorbell: checked
    ```

  _Global Automation_

    ```
    HTTP Port:8080
    ```

  _При наличии RTSP потока с домофона заменяем "-re -stream_loop -1 -i /opt/sipdoorbell/homekit.avi" на ссылку RTSP потока_


* Конфигурация плагина Homebridge Camera FFmpeg

  ```
  "source": "-re -stream_loop -1 -i /opt/sipdoorbell/homekit.avi -f alsa -ac 2 -ar 48000 -i sipdoorbell_main",
  "stillImageSource": "-i /opt/sipdoorbell/homekit.jpg",
  "returnAudioTarget": "-filter:a volume=12dB -f alsa -ac 1 -ar 48000 sipdoorbell_return",
  "mapaudio": "1:a",
  "audio": true,
  "debugReturn": true
  ```

  _При наличии RTSP потока с домофона заменяем "-re -stream_loop -1 -i /opt/sipdoorbell/homekit.avi" на ссылку RTSP потока_
  
  
* В Homebridge устанавливаем плагин Homebridge Http Switch (если необходимо)

* Конфигурация плагина Homebridge Http Switch (если необходимо)

  Кнопка для отправки HTTP запроса в Baresip для открытия двери
  
  ```
          {
              "accessory": "HTTP-SWITCH",
              "name": "Open",
              "switchType": "stateless",
              "timeout": 3000,
              "multipleUrlExecutionStrategy": "series",
              "onUrl": [
                  "http://127.0.0.1:8000/?/sndcode 0",
                  "http://127.0.0.1:8000/?/sndcode %04"
              ]
          }
  ```
  
  Кнопки для ручного ответа и завершения вызова (например для создания Homekit автоматизаций автоматического открытия двери домофоном)
  
  ```
          {
              "accessory": "HTTP-SWITCH",
              "name": "Answer",
              "switchType": "stateless",
              "timeout": 3000,
              "onUrl": "http://127.0.0.1:8000/?/accept"
          },
          {
              "accessory": "HTTP-SWITCH",
              "name": "Hangup",
              "switchType": "stateless",
              "timeout": 3000,
              "multipleUrlExecutionStrategy": "series",
              "onUrl": [
                  "http://127.0.0.1:8000/?/sndcode %23",
                  "http://127.0.0.1:8000/?/sndcode %04",
                  "delay(1000)",
                  "http://127.0.0.1:8000/?/hangup"
              ]
          }          
  ```
  
  В данной конфигурации открыкие замка осуществляется отправкой DTMF кода "0". Заменяем "sndcode 0" на необходимое значение.
  
  В данной конфигурации завершение SIP вызова осуществляется отправкой DTMF кода "#". Заменяем "sndcode %23" на необходимое значение.

* Загрузим необходимые файлы для формирования видеопотока

  ```
  cd /opt
  sudo git clone https://github.com/spbroot/sipdoorbell.git
  sudo mv /usr/share/baresip/ring.wav /usr/share/baresip/ring.wav.backup
  sudo ln -s /opt/sipdoorbell/doorbell.wav /usr/share/baresip/ring.wav
  ```
  
* Создаем скрипт /opt/sipdoorbell/monitor.sh для дальнейшей проверки входящего вызова в baresip и уведомления о нём ffmpeg

  ```
  sudo mkdir /opt/sipdoorbell
  sudo nano /opt/sipdoorbell/monitor.sh
  ```

  со следующим содержимым:

    ```
  #!/bin/bash

  homebridge_log_path='/var/lib/homebridge/homebridge.log'
  doorbell_device_name='Doorbell'
  doorbell_ring_repeat=20
  date_format='%d/%m/%Y'

  count=0
  while true;
  do
    if wget -qO- http://127.0.0.1:8000/?/callstat | grep -q "INCOMING"; then
      if [[ "$count" -eq "0" ]]; then
        wget -q http://127.0.0.1:8080/doorbell?$doorbell_device_name > /dev/null 2>&1
        ((count++))
      fi
      tail -100 $homebridge_log_path | awk -v device_name=$doorbell_device_name -v date=`date -d'now' +$date_format` -v time=`date -d'now-1 seconds' +%H:%M:%S` '$0~date && $0~time && $0~device_name && /Two-way/ && /time=/ && !/time=00:00:00.00/ {system("wget -q http://127.0.0.1:8000/?/accept > /dev/null 2>&1"); exit 1}'
      if [ $? -eq 1 ]; then
        sleep 1
        while wget -qO- http://127.0.0.1:8000/?/callstat | grep -q "ESTABLISHED";
        do
          tail -100 $homebridge_log_path | awk -v device_name=$doorbell_device_name -v date=`date -d'now' +$date_format` -v time1=`date -d'now-1 seconds' +%H:%M:%S` -v time2=`date -d'now-1 seconds' +%H:%M:%S` -v time3=`date -d'now-1 seconds' +%H:%M:%S` -v time4=`date -d'now-4 seconds' +%H:%M:%S` -v time5=`date -d'now-5 seconds' +%H:%M:%S` '($0~date && ($0~time1 || $0~time2 || $0~time3 || $0~time4 || $0~time5) && $0~device_name && /Two-way/ && /time=/ && !/time=00:00:00.00/) {found=1} END {if(!found) system("wget -q http://127.0.0.1:8000/?/hangup > /dev/null 2>&1")}'
          sleep 1
        done
      elif (( "$count" > $doorbell_ring_repeat )); then
        count=0
      else
        ((count++))
      fi
    else
      count=0
    fi
    sleep 1
  done
  ```


  _Имя устройства "Doorbell" в Camera FFmpeg Plugin должно совпадать с "doorbell_device_name"_
  
  _"homebridge_log_path" путь к логам homebridge_
  
  _"date_format" формат даты в логах homebridge_ 

  _Скрипт отвечает на SIP вызов при нажатии кнопки "Говорить", и разрывает SIP соединение через 5 секунд после отключения кнопки "Говорить" или закрытии приложения._



* Делаем скрипт /opt/sipdoorbell/monitor.sh исполняемым

  ```
  sudo chmod +x /opt/sipdoorbell/monitor.sh
  ```

* Создаем файл /etc/systemd/system/baresip.service для создания сервиса Baresip

  ```
  sudo nano /etc/systemd/system/baresip.service
  ```

  ```
  [Unit]
  Description=baresip
  After=network.target

  [Service]
  User=pi
  Group=pi
  Type=simple
  KillMode=process
  ExecStart=baresip -v -d -f '/home/pi/.baresip'
  Restart=on-failure
  RestartSec=3

  [Install]
  WantedBy=multi-user.target
  ```
  
  _Измените имя пользователя, группу, путь если необходимо_

* Запускаем созданный нами сервис Baresip и добавляем в автозагрузку

  ```
  sudo systemctl daemon-reload
  sudo systemctl start baresip.service
  sudo systemctl enable baresip.service
  ```

* Создаем файл /etc/systemd/system/sipdoorbell.service для создания сервиса Sipdoorbell

  ```
  sudo nano /etc/systemd/system/sipdoorbell.service
  ```

  со следующим содержимым:

  ```
  [Unit]
  Description=Sipdoorbell
  After=network.target

  [Service]
  User=pi
  Group=pi
  Type=simple
  ExecStart=/opt/sipdoorbell/monitor.sh
  KillMode=process
  Restart=on-failure
  RestartSec=3

  [Install]
  WantedBy=multi-user.target
  ```
  
  _Измените имя пользователя, группу, путь если необходимо_

* Запускаем созданный нами сервис Sipdoorbell и добавляем в автозагрузку

  ```
  sudo systemctl daemon-reload
  sudo systemctl start sipdoorbell.service
  sudo systemctl enable sipdoorbell.service
  ```
