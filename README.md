# Adding a SIP doorbell to Homekit with two-way audio

This example shows how to add SIP audio intercom to Homekit

Used Rasberry Pi 4 with Homebridge image installed

Intercom control is carried out via DTMF commands

Make changes to the system settings:
- 24-hour time format
- date format DD/MM/YYYY

* Adding the ALSA Loopback Sound Device

  ```
  sudo su
  echo 'snd-aloop' >> /etc/modules
  exit
  ```

* Making changes to the ALSA configuration

  Create or edit the ```/etc/asound.conf``` file

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

    _!!! if you get homebridge error when open doorbell device on iphone/ipad/AppleTV:_

    ```
    [error] cannot open audio device sipdoorbell_main (No such file or directory)
    [error] sipdoorbell_main: Input/output error
    ```

    _or_

    ```
    [error] cannot open audio device sipdoorbell_return (No such file or directory)
    [error] sipdoorbell_return: Input/output error
    ```

    _put this code at the end ```/usr/share/alsa/alsa.conf``` instead ```/etc/asound.conf```_

* Installing baresip

  ```
  sudo apt install baresip
  ```

* Run baresip to create a configuration file, after launch press 'q' to exit
* Making changes to the baresip configuration file: file ```/home/pi/.baresip/config```

  Remove the comment from the following lines and make changes

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

* Add a SIP account to the file ```/home/pi/.baresip/accounts```

  * When using P2P SIP call
  
    When using P2P, the call must be in the format "100 @ <homebridge_ip_address>"

    ```
    <sip:100@127.0.0.1>;auth_pass=none;regint=0;
    ```

  * When using SIP registration

    ```
    <sip:extention_number@sip_server_ip>;auth_pass=account_password;
    ```

* Install the Homebridge Camera FFmpeg plugin

* Add a camera in the Homebridge Camera FFmpeg plugin

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
    Two-way Audio Target (EXPERIMENTAL - WIP): "-f alsa -ac 2 -ar 48000 sipdoorbell_return"
    Two-way FFmpeg Debug Logging: checked
    Audio Stream: "1:a"
    Enable doorbell: checked
    ```

  _Global Automation_

    ```
    HTTP Port:8080
    ```

  _If there is an RTSP stream from the intercom, replace "-re -stream_loop -1 -i /opt/sipdoorbell/homekit.avi" and  
  "Still Image Source: "-i /opt/sipdoorbell/homekit.jpg"" with the link of the RTSP stream_


* Homebridge Camera FFmpeg Plugin Configuration

  ```
  "source": "-re -stream_loop -1 -i /opt/sipdoorbell/homekit.avi -f alsa -ac 2 -ar 48000 -i sipdoorbell_main",
  "stillImageSource": "-i /opt/sipdoorbell/homekit.jpg",
  "returnAudioTarget": "-f alsa -ac 2 -ar 48000 sipdoorbell_return",
  "mapaudio": "1:a",
  "audio": true,
  "debugReturn": true
  ```

  _If there is an RTSP stream from the intercom, replace "-re -stream_loop -1 -i /opt/sipdoorbell/homekit.avi" and  
  "Still Image Source: "-i /opt/sipdoorbell/homekit.jpg"" with the link of the RTSP stream_
  
* Install the Homebridge Http Switch plugin (if nessesary)

* Homebridge Http Switch Plugin Configuration (if nessesary)

  Command for open the door via send HTTP request to Baresip
  
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
  
  Command for manual answer and hangup SIP call (for Homekit Automation etc.)
  
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
  
  In this configuration, the open lock is performed by sending the DTMF code "0". Replace "sndcode 0" with the required value
  
  In this configuration, the SIP call hangup is performed by sending the DTMF code "#". Replace "sndcode %23" with the required value
  
* Download the necessary files

  ```
  cd /opt
  sudo git clone https://github.com/spbroot/sipdoorbell.git
  sudo mv /usr/share/baresip/ring.wav /usr/share/baresip/ring.wav.backup
  sudo ln -s /opt/sipdoorbell/doorbell.wav /usr/share/baresip/ring.wav
  ```
  
* Create a script for check the incoming call to baresip and notify ffmpeg about it

  ```
  sudo mkdir /opt/sipdoorbell
  sudo nano /opt/sipdoorbell/monitor.sh
  ```

  ```
  #!/bin/bash

  homebridge_log_path='/var/lib/homebridge/homebridge.log'
  doorbell_device_name='Doorbell'
  doorbell_ring_repeat=20
  date_format='%d/%m/%Y'

  count=0
  while true;
  do
    if wget -qO- http://localhost:8000/?/callstat | grep -q "INCOMING"; then
      if [[ "$count" -eq "0" ]]; then
        wget -q http://localhost:8080/doorbell?$doorbell_device_name > /dev/null 2>&1
        ((count++))
      fi
      tail -100 $homebridge_log_path | awk -v device_name=$doorbell_device_name -v date=`date -d'now' +$date_format` -v time=`date -d'now-1 seconds' +%H:%M:%S` '$0~date && $0~time && $0~device_name && /Two-way/ && /time=/ && !/time=00:00:00.00/ {system("wget -q http://localhost:8000/?/accept > /dev/null 2>&1"); exit 1}'
      if [ $? -eq 1 ]; then
        sleep 1
        while wget -qO- http://localhost:8000/?/callstat | grep -q "ESTABLISHED";
        do
          tail -100 $homebridge_log_path | awk -v device_name=$doorbell_device_name -v date=`date -d'now' +$date_format` -v time1=`date -d'now-1 seconds' +%H:%M:%S` -v time2=`date -d'now-1 seconds' +%H:%M:%S` -v time3=`date -d'now-1 seconds' +%H:%M:%S` -v time4=`date -d'now-4 seconds' +%H:%M:%S` -v time5=`date -d'now-5 seconds' +%H:%M:%S` '($0~date && ($0~time1 || $0~time2 || $0~time3 || $0~time4 || $0~time5) && $0~device_name && /Two-way/ && /time=/ && !/time=00:00:00.00/) {found=1} END {if(!found) system("wget -q http://localhost:8000/?/hangup > /dev/null 2>&1")}'
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

  _The device name "Doorbell" in the Camera FFmpeg Plugin must match with variable "doorbell_device_name"_
  
  _"homebridge_log_path" path to homebridge log file_
  
  _"date_format" date format of homebridge log file_ 

  _This script should answer the call when you enable TALK button, and end the call 5 seconds after disable TALK button or closing the application_

* Making the script ```/opt/sipdoorbell/monitor.sh``` executable

  ```
  sudo chmod +x /opt/sipdoorbell/monitor.sh
  ```

* Create baresip service

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
  
  _Change user, group and path if it nessesary_

* Launch the baresip service and add to startup

  ```
  sudo systemctl daemon-reload
  sudo systemctl start baresip.service
  sudo systemctl enable baresip.service
  ```

* Create Sipdoorbell service

  ```
  sudo nano /etc/systemd/system/sipdoorbell.service
  ```

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
  
  _Change user, group and path if it nessesary_
  
* Launch the sipdoorbell service and add to startup

  ```
  sudo systemctl daemon-reload
  sudo systemctl start sipdoorbell.service
  sudo systemctl enable sipdoorbell.service
  ```
