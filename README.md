# Adding a SIP intercom to Homekit with two-way audio

This example shows how to add SIP audio intercom to Homekit

Used Rasberry Pi 4 with Homebridge image installed

Intercom control is carried out via DTMF commands

* Adding the ALSA Loopback Sound Device

  ```
  sudo su
  echo 'snd-aloop' >> /etc/modules
  exit
  ```

* Making changes to the ALSA configuration

  Create or edit the /etc/asound.conf file

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

* Installing baresip

  ```
  sudo apt install baresip
  ```

* Run baresip to create a configuration file, after launch press 'q' to exit
* Making changes to the baresip configuration file: file /home/pi/.baresip/config

  Remove the comment from the following lines and make changes

  ```
  sip_listen              0.0.0.0:5060

  audio_player            alsa,hw:Loopback,0,1
  audio_source            alsa,hw:Loopback,1,0
  audio_alert             alsa,hw:Loopback,0,1

  ausrc_srate             48000
  auplay_srate            48000
  ausrc_channels          1
  auplay_channels         2

  module                  httpd.so

  http_listen             0.0.0.0:8000
  ```

* Add a SIP account to the file /home/pi/.baresip/accounts

  * When using P2P SIP call
  
    When using P2P, the call must be in the format "100 @ <homebridge_ip_address>"

    ```
    <sip:100@127.0.0.1>;auth_pass=none;regint=0;
    ```

  * When using SIP registration

    ```
    <sip:extention_number@sip_server_ip>;auth_pass=account_password;regint=0;
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
    Two-way Audio Target (EXPERIMENTAL - WIP): "-filter:a volume=12dB -f alsa -ac 1 -ar 48000 sipdoorbell_return"
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
  "returnAudioTarget": "-filter:a volume=12dB -f alsa -ac 1 -ar 48000 sipdoorbell_return",
  "mapaudio": "1:a",
  ```

  _If there is an RTSP stream from the intercom, replace "-re -stream_loop -1 -i /opt/sipdoorbell/homekit.avi" and  
  "Still Image Source: "-i /opt/sipdoorbell/homekit.jpg"" with the link of the RTSP stream_
  
* Install the Homebridge Http Switch plugin

* Homebridge Http Switch Plugin Configuration

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
              "name": "Open",
              "switchType": "stateless",
              "timeout": 3000,
              "multipleUrlExecutionStrategy": "series",
              "onUrl": [
                  "http://localhost:8000/?/sndcode 0",
                  "http://localhost:8000/?/sndcode %04"
              ]
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
  
* Download the necessary files to generate a video stream
  
  _Necessary only if there is no RTSP stream from the intercom_

  ```
  cd /opt
  sudo git clone https://github.com/spbroot/sipdoorbell.git
  ```
  
* Create a script /opt/sipdoorbell/monitor.sh to start baresip at system startup and further check the incoming call to baresip and notify ffmpeg about it

  ```
  sudo mkdir /opt/sipdoorbell
  sudo nano /opt/sipdoorbell/monitor.sh
  ```

  with the following content:

  ```
  #!/bin/bash
  baresip -d -f /home/pi/.baresip
  while true;
  do
    if wget -qO- http://localhost:8000/?/callstat | grep -q "INCOMING"
    then
      wget -q http://localhost:8080/doorbell?Doorbell > /dev/null 2>&1
      sleep 20
    fi
    sleep 1
  done
  ```

* Making the script /opt/sipdoorbell/monitor.sh executable

  ```
  sudo chmod +x /opt/sipdoorbell/monitor.sh
  ```

* Create a script /opt/sipdoorbell/control.sh to add the ability to answer and end a call by pressing the TALK button in the Homekit app

  ```
  sudo mkdir /opt/sipdoorbell
  sudo nano /opt/sipdoorbell/control.sh
  ```

  with the following content:

  ```  
  #!/bin/bash
  while true;
  do
    if wget -qO- http://localhost:8000/?/callstat | grep -q "INCOMING"
      then
        tail -100 /var/lib/homebridge/homebridge.log | awk -v date=`date -d'now' +%d/%m/%Y` -v time=`date -d'now-1 seconds' +%H:%M:%S` '$0~date && $0~time && /[Doorbell]/ && /[Two-way]/ && /time=/ {system("wget -q http://localhost:8000/?/accept")}'
    fi
    if wget -qO- http://localhost:8000/?/callstat | grep -q "ESTABLISHED"
      then
        tail -100 /var/lib/homebridge/homebridge.log | awk -v date=`date -d'now' +%d/%m/%Y` -v time=`date -d'now-1 seconds' +%H:%M:%S` -v time15=`date -d'now-15 seconds' +%H:%M:%S` '($0~date && ($0~time || $0~time15) && /[Doorbell]/ && /[Two-way]/ && /time=/) {found=1} END {if(!found) system("wget -q http://localhost:8000/?/hangup")}'
    fi
    sleep 1
  done
  ```
  
  _This script should answer the call when you press the TALK button, and end the call 15 seconds after disconnecting TALK or closing the application_
  
  _The device name "Doorbell" in the Camera FFmpeg Plugin must match the name on the line "&& /[Doorbell]/ &&"_


* Making the script /opt/sipdoorbell/control.sh executable

  ```
  sudo chmod +x /opt/sipdoorbell/control.sh
  ```

* Create a file /etc/systemd/system/sipdoorbell.service to create a Sipdoorbell service

  ```
  sudo nano /etc/systemd/system/sipdoorbell.service
  ```

  with the following content:

  ```
  [Unit]
  Description=Sipdoorbell
  After=network.target

  [Service]
  Type=simple
  ExecStartPre=/opt/sipdoorbell/control.sh
  ExecStart=/opt/sipdoorbell/monitor.sh
  
  [Install]
  WantedBy=multi-user.target
  ```

* Launch the Sipdoorbell service we created and add to startup

  ```
  sudo systemctl daemon-reload
  sudo systemctl start sipdoorbell.service
  sudo systemctl enable sipdoorbell.service
  ```
