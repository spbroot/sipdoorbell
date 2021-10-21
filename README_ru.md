

# Добавление SIP домофона в Homekit c двухсторонним аудио

Данный пример описывает добавление аудио SIP домофона в Homekit

Используется Rasberry Pi 4 c установленным образом Homebridge

Управление домофоном осуществляется через DTMF команды

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
    slave.pcm "loop_out"
  }

  # plug device
  pcm.sipdoorbell_main {
    type plug
    slave.pcm "loop_in"
  }
  ```

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
    <sip:extention_number@sip_server_ip>;auth_pass=account_password;regint=0;
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
                  "http://localhost:8000/?/sndcode 0",
                  "http://localhost:8000/?/sndcode %04"
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
              "onUrl": "http://localhost:8000/?/accept"
          },
          {
              "accessory": "HTTP-SWITCH",
              "name": "Hangup",
              "switchType": "stateless",
              "timeout": 3000,
              "multipleUrlExecutionStrategy": "series",
              "onUrl": [
                  "http://localhost:8000/?/sndcode %23",
                  "http://localhost:8000/?/sndcode %04",
                  "delay(1000)",
                  "http://localhost:8000/?/hangup"
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
  
* Создаем скрипт /opt/sipdoorbell/monitor.sh для запуска baresip при старте системы и дальнейшей проверки входящего вызова в baresip и уведомления о нём ffmpeg

  ```
  sudo mkdir /opt/sipdoorbell
  sudo nano /opt/sipdoorbell/monitor.sh
  ```

  со следующим содержимым:

  ```
   #!/bin/bash


  homebridge_log_path='/var/lib/homebridge/homebridge.log'
  baresip_path='/home/pi/.baresip'
  doorbell_device_name='Doorbell'
  doorbell_ring_repeat=20
  
  baresip -d -f $baresip_path

  count=0
  while true;
  do
    if wget -qO- http://localhost:8000/?/callstat | grep -q "INCOMING"; then
      if [[ "$count" -eq "0" ]]; then
        wget -q http://localhost:8080/doorbell?$doorbell_device_name > /dev/null 2>&1
        ((count++))
      fi
      tail -100 $homebridge_log_path | awk -v device_name='['+$doorbell_device_name+']' -v date=`date -d'now' +%d/%m/%Y` -v time=`date -d'now-1 seconds' +%H:%M:%S` '$0~date && $0~time && $0~device_name && /[Two-way]/ && /time=/ && !/time=00:00:00.00/ {system("wget -q http://localhost:8000/?/accept > /dev/null 2>&1"); exit 1}'
      if [ $? -eq 1 ]; then
        sleep 1
        while wget -qO- http://localhost:8000/?/callstat | grep -q "ESTABLISHED";
        do
          tail -100 $homebridge_log_path | awk -v device_name='['+$doorbell_device_name+']' -v date=`date -d'now' +%d/%m/%Y` -v time1=`date -d'now-1 seconds' +%H:%M:%S` -v time2=`date -d'now-1 seconds' +%H:%M:%S` -v time3=`date -d'now-1 seconds' +%H:%M:%S` -v time4=`date -d'now-4 seconds' +%H:%M:%S` -v time5=`date -d'now-5 seconds' +%H:%M:%S` '($0~date && ($0~time1 || $0~time2 || $0~time3 || $0~time4 || $0~time5) && $0~device_name && /[Two-way]/ && /time=/ && !/time=00:00:00.00/) {found=1} END {if(!found) system("wget -q http://localhost:8000/?/hangup > /dev/null 2>&1")}'
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

* Делаем скрипт /opt/sipdoorbell/monitor.sh исполняемым

  ```
  sudo chmod +x /opt/sipdoorbell/monitor.sh
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
  Type=simple
  ExecStart=/opt/sipdoorbell/monitor.sh

  [Install]
  WantedBy=multi-user.target
  ```

* Запускаем созданный нами сервис Sipdoorbell и добавляем в автозагрузку

  ```
  sudo systemctl daemon-reload
  sudo systemctl start sipdoorbell.service
  sudo systemctl enable sipdoorbell.service
  ```
