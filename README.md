RAW STEPS TAKEN - make these automatable!

sudo su
apt install usb-modeswitch mosh tmux modem-cmd ppp rsyslog mg

wget https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-arm.zip
unzip ngrok-stable-linux-arm.zip
mv ngrok /usr/bin/

nano /etc/systemd/system/ngrok.service
-->
[Unit]
Description=Share local port(s) with ngrok
After=network.target

[Service]
PrivateTmp=true
Type=simple
Restart=always
StandardOutput=null
StandardError=null
ExecStart=/usr/bin/ngrok start --config /etc/ngrok/ngrok.yml --all
ExecStop=/usr/bin/killall ngrok

[Install]
WantedBy=multi-user.target

nano /etc/ngrok2/ngrok.yml
-->
authtoken: <auth_token_from_website>
region: eu
log_level: info
log: /var/log/ngrok.log
tunnels:
  ssh:
    addr: 22
    proto: tcp

nano ~/.tmux.conf
-->
unbind C-b
set -g prefix C-q
bind C-q send-prefix

usb_modeswitch -W -v 05c6 -p 1000 -K

!!FIXME!!
nano /etc/udev/rules.d/qualcomm_usb_storage.rules
-->
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="05c6", ATTRS{idProduct}=="1000", RUN+="/usr/sbin/usb_modeswitch -W -v 05c6 -p 1000 -K"

Modem Info:
AT+GMI: ALEKA INCORPORATED
AT+GMM: HSPA Modem
AT+GMR: MSM7X27-02.02.03UFI  1  [Oct 29 2014 16:20:35]

nano /etc/udev/rules.d/qualcomm_usb_modem.rules
-->
KERNEL=="ttyUSB[0-9]*", SUBSYSTEMS=="usb", DRIVERS=="option", ATTRS{bInterfaceNumber}=="00", ATTRS{bInterfaceClass}=="ff", ATTRS{bInterfaceSubClass}=="ff", ATTRS{bInterfaceProtocol}=="ff", SYMLINK+="USBModem"

nano /etc/ppp/peers/meteor
-->
debug
/dev/USBModem
921600
connect '/usr/sbin/chat -v -f /etc/ppp/chat/dial-meteor'
disconnect '/usr/sbin/chat -v -f /etc/ppp/chat/hangup-meteor'
# mru 296
mtu 512
noccp
novj
novjccomp
crtscts
modem -detach
noauth
noipdefault
usepeerdns
defaultroute
replacedefaultroute
local
nodeflate
nodetach
persist
holdoff 15
ipcp-accept-local
ipcp-accept-remote
ipcp-restart 10
# Check the line every 20 seconds and presume
# the peer is gone if no replay for 4 times.
lcp-echo-interval 20
lcp-echo-failure 4

mkdir -p /etc/ppp/chat
nano /etc/ppp/chat/dial-meteor
-->
'' ATZ
OK AT+CGDCONT=1,"IP","isp.mymeteor.ie"
OK "ATD*99***1#"
CONNECT ''

nano /etc/ppp/chat/hangup-meteor
-->
"" "\K"
"" "+++ATH0"

nano /etc/rsyslog.d/pppd.conf
-->
if $programname == 'pppd' then /var/log/pppd.log

nano /etc/systemd/system/pppd.service
-->
[Unit]
Description=PPP daemon
After=network.target

[Service]
ExecStart=/usr/sbin/pppd debug call meteor nodetach
SyslogIdentifier=pppd

[Install]
WantedBy=multi-user.target

systemctl restart rsyslog
systemctl enabled pppd
systemctl enabled ngrok
systemctl start pppd
systemctl start ngrok