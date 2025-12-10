<h1>Building a gateway with Raspberry Pi and IC880A</h1>
<h3>This guide can help you build your own LoRaWAN gateway using a Raspberry Pi and an iC880A LoRa concentrator board, and run LoRa Basics™ Station on it.</h3>
<h2>Requirements#
</h2>
<h3>For building this gateway you will need the following hardware elements:<br>

1)iC880A-SPI concentrator board<br>
2)3.5dBi - 7.5dBi antenna<br>
3)iC880A pigtail for antenna<br>
4)Raspberry Pi Model 2 or newer<br>
5)2.5A power supply with micro USB connector<br>
6)MicroSD Card with minimum 4GB of storage<br>
7)7x dual female jumper wires<br>
8)Ethernet cable or WiFi dongle (if using Raspberry PI 3+ this isn’t required, because it has an integrated WiFi interface)
</h3>
<h2>Install Raspberry Pi OS on Raspberry Pi#
</h2>
<h2># 1. Download and Flash Raspberry Pi OS # Step 1.1 – Download Raspberry Pi Imager:</h2>
<p>Link:<a hrf:"https://www.raspberrypi.com/software/">https://www.raspberrypi.com/software/</a><br>Install it for your platform. (Windows/macOS/Linux)

</p>
<h3>
Insert the SD card into your computer.<br>
Open Raspberry Pi Imager.<br>

Choose OS: Raspberry Pi OS (32-bit) → Raspberry Pi OS Lite (headless, no GUI) or with Desktop.<br>

Choose SD card (double-check the right disk).<br>

Click Next.<br>

>Configure Wi-Fi & Enable SSH (Headless Setup) If you're using a monitor, you can skip this part and configure directly after boot.<br>
Just fill the requirments according to it's not  need.

</h3>
<h2>Boot and Configure Your Raspberry Pi#
</h2>
<p>Insert the SD card back into the Raspberry Pi and power it on.
If SSH wasn’t enabled, you must use a monitor and keyboard for setup.
If SSH was enabled, find the Pi’s IP address on your network (e.g., using nmap) and connect remotely via SSH using the default login: username: pi, password: raspberry.</p>
<h3 style="color:"green"   >ssh pi@192.168.1.2
</h3>
<p>After logging in in your Raspberry Pi, upgrade the system packages to the latest versions:</p>
<h3 style="color:"green"   >sudo apt-get update<br>
sudo apt-get upgrade
</h3>
<p>Enable the SPI interface by running the raspi-config tool:</p>
<h3 style="color:"green">sudo raspi-config</h3>
<p>A raspi-config wizard will appear, so use the arrow keys and the Enter key to navigate through it. Choose Interface Options → SPI, then select to enable the SPI interface. After enabling the SPI interface, hit the Escape key to exit the raspi-config tool.

Now install packages needed to build the packet forwarder:</p>
<h3 style="color:"green">sudo apt-get install git gcc make</h3>
<h2>Build the LoRa Basics™ Station Packet Forwarder#</h2>
<p>First, clone the The Things Stack repository:</p>
<h3 style="color:"green">git clone https://github.com/lorabasics/basicstation</h3>
<p>Then build the LoRa Basics™ Station binary:</p>
<h3 style="color:"green">cd basicstation<br>
make platform=rpi variant=std ARCH=$(gcc --print-multiarch)</h3>
<p>Make sure the binary was successfully built:
</p>
<h3 style="color:"green">./build-rpi-std/bin/station --version</h3>
<br>
<p>The binary was successfully built if you see something like this:</p>
<h3 style="color:"green">Station: 2.0.6(rpi/std) 2022-03-26 17:43:16<br>
Package: (null)</h3>
<p>Now install the LoRa Basics™ Station binary:</p>
<h3 style="color:"green">sudo mkdir -p /opt/ttn-station/bin <br>
sudo cp ./build-rpi-std/bin/station /opt/ttn-station/bin/station
</h3>
<h2>Derive the Gateway EUI#
</h2>
<p>To derive the gateway EUI, you can use a combination of the gateway’s MAC address and FFFE, as follows:</p>
<h3 style="color:"green">export MAC=`cat /sys/class/net/eth0/address`<br>
export EUI=`echo $MAC | awk -F: '{print $1$2$3 "fffe" $4$5$6}'`<br>
echo "The Gateway EUI is $EUI" </h3>
<p>The output will look something like:</p>
<h3 style="color:"green">The Gateway EUI is b827ebfffee00c83
</h3>
<h2>Register the Gateway on The Things Stack#
</h2>
<p> To registar gateway click this link<br><a hrf:"https://www.thethingsindustries.com/docs/hardware/gateways/concepts/adding-gateways/">https://www.thethingsindustries.com/docs/hardware/gateways/concepts/adding-gateways/</a></p>
<h2>Create an API Key#</h2>
<p> To registar API ke click this link<br><a hrf:"https://www.thethingsindustries.com/docs/hardware/gateways/concepts/lora-basics-station/lns/#create-an-api-key">https://www.thethingsindustries.com/docs/hardware/gateways/concepts/lora-basics-station/lns/#create-an-api-key</a></p>
<h2>Configure LoRa Basics™ Station#
</h2>
<p>The next step is to create the configuration files required for LoRa Basics™ Station gateway to connect to The Things Stack.<br>

On your Raspberry Pi, create a new directory:</p>
<h3 style="color:"green">sudo mkdir -p /opt/ttn-station/config</h3>
<p>Create a configuration file tc.uri containing an LNS server address. For example, if using eu1 cluster of The Things Stack Sandbox:</p>
<h3 style="color:"green">echo 'wss://eu1.cloud.thethings.network:8887' | sudo tee /opt/ttn-station/config/tc.uri
</h3>
<p>Next, create the tc.key configuration file containing an authorization header. This header will contain the API Key you created in the previous step, and it will be used to authenticate your gateway’s connection.</p>
<h3 style="color:"green">export API_KEY="NNSXS.XXXXXXXXXXXXXXX.YYYYYYYYYYYYYYYY"<br>
echo "Authorization: Bearer $API_KEY" | perl -p -e 's/\r\n|\n|\r/\r\n/g' | sudo tee -a /opt/ttn-station/config/tc.key
</h3>
<p>Create the tc.trust configuration file that will be the root CA used to check your LNS server’s certificates. You can use the system CA certificates:</p>
<h3 style="color:"green">sudo ln -s /etc/ssl/certs/ca-certificates.crt /opt/ttn-station/config/tc.trust</h3>
<p>Now, create the station.conf configuration file containing configuration options for your concentrator:</p>
<h3 style="color:"green">echo '<br>
{<br>
    /* If slave-X.conf present this acts as default settings */<br>
    "SX1301_conf": { /* Actual channel plan is controlled by server */<br>
        "lorawan_public": true, /* is default */<br>
        "clksrc": 1, /* radio_1 provides clock to concentrator */<br>
        /* path to the SPI device, un-comment if not specified on the command line e.g., RADIODEV=/dev/spidev0.0 */<br>
        /*"device": "/dev/spidev0.0",*/<br>
        /* freq/enable provided by LNS - only HW specific settings listed here */<br>
        "radio_0": {<br>
            "type": "SX1257",<br>
            "rssi_offset": -166.0,<br>
            "tx_enable": true,<br>
            "antenna_gain": 0<br>
        },
        "radio_1": {<br>
            "type": "SX1257",<br>
            "rssi_offset": -166.0,<br>
            "tx_enable": false<br>
        }<br>
        /* chan_multiSF_X, chan_Lora_std, chan_FSK provided by LNS */<br>
    },<br>
    "station_conf": {<br>
        "routerid": "'"<p style="color:"red">$EUI</p>"'",<br>
        "log_file": "stderr",<br>
        "log_level": "DEBUG", /* XDEBUG,DEBUG,VERBOSE,INFO,NOTICE,WARNING,ERROR,CRITICAL */<br>
        "log_size": 10000000,<br>
        "log_rotate": 3,<br>
        "CUPS_RESYNC_INTV": "1s"<br>
    }<br>
}<br>
' | sudo tee /opt/ttn-station/config/station.conf</h3>

<p>Finally, create the start.sh script, that will be used to reset the iC880A via its reset pin and start the packet forwarder:

</p>
<h3 style="color:"green">echo '#!/bin/bash<br>

# Reset iC880a PIN<br>
SX1301_RESET_BCM_PIN=25<br>
echo "$SX1301_RESET_BCM_PIN"  > /sys/class/gpio/export<br>
echo "out" > /sys/class/gpio/gpio$SX1301_RESET_BCM_PIN/direction<br>
echo "0"   > /sys/class/gpio/gpio$SX1301_RESET_BCM_PIN/value<br>
sleep 0.1<br>
echo "1"   > /sys/class/gpio/gpio$SX1301_RESET_BCM_PIN/value<br>
sleep 0.1<br>
echo "0"   > /sys/class/gpio/gpio$SX1301_RESET_BCM_PIN/value<br>
sleep 0.1<br>
echo "$SX1301_RESET_BCM_PIN"  > /sys/class/gpio/unexport<br>

# Test the connection, wait if needed.<br>
while [[ $(ping -c1 google.com 2>&1 | grep " 0% packet loss") == "" ]]; do<br>
echo "[TTN Gateway]: Waiting for internet connection..."<br>
sleep 30<br>
done<br>

# Start station<br>
/opt/ttn-station/bin/station<br>
' | sudo tee /opt/ttn-station/bin/start.sh</h3>

<p>You can check if your start.sh script is executable with:</p>
<h3 style="color:"green">sudo chmod +x /opt/ttn-station/bin/start.sh
</h3>
<h2>Test the Packet Forwarder#
</h2>
<p>Start the packet fowarder with:</p>
<h3 style="color:"green">cd /opt/ttn-station/config<br>
sudo RADIODEV=/dev/spidev0.0 /opt/ttn-station/bin/start.sh
</h3>
<p>This will initialize the concentrator board, connect your gateway to The Things Stack, fetch the configuration based on your frequency plan and start listening for packets. If you notice something like:</p>
<p>2022-03-27 02:11:50.009 [S2E:VERB] RX 867.5MHz DR5 SF7/BW125 snr=6.2 rssi=-103 xtime=0xE0000000E34FB4 - updf mhdr=40 DevAddr=260B0748 FCtrl=00 FCnt=35 FOpts=[] 014A mic=1717970429 (14 bytes)<br>
2022-03-27 02:12:06.130 [S2E:VERB] RX 867.3MHz DR5 SF7/BW125 snr=8.0 rssi=-102 xtime=0xE0000001D92CCB - updf mhdr=40 DevAddr=260B0748 FCtrl=00 FCnt=36 FOpts=[] 01EA mic=463407879 (14 bytes)
</p>
<p>in the packet forwarder logs, it means your gateway has started picking up messages, Of course, this is possible if there are end devices transmitting data within the gateway’s reach.

If you go to The Things Stack Console and navigate to your gateway’s Live Data view, it will appear as connected and you will see uplink messages arriving.
</p>
<h2>Run the Packet Forwarder as a System Service#
</h2>
<p>The only thing left to do is to configure the packet forwarder to run as a system service on Raspberry Pi. This ensures that the forwarder will start automatically after the Raspberry Pi boots.<br>

First, create the systemd service configuration file:</p>
<h3 style="color:"green">echo '<br>
[Unit]<br>
Description=The Things Network Gateway<br>

[Service]<br>
WorkingDirectory=/opt/ttn-station/config<br>
ExecStart=/opt/ttn-station/bin/start.sh<br>
SyslogIdentifier=ttn-station<br>
Restart=on-failure<br>
RestartSec=5<br>

[Install]<br>
WantedBy=multi-user.target<br>
' | sudo tee /lib/systemd/system/ttn-station.service<br>
</h3>
<p>Enable the service with:</p>
<h3 style="color:"green">sudo systemctl enable ttn-station</h3>
<p>Start the service:</p>
<h3 style="color:"green">sudo systemctl start ttn-station</h3>
<p>You can observe the packet forwarder logs using the following command:</p>
<h3 style="color:"green"><h3 style="color:"green">sudo systemctl start ttn-station</h3>
</h3>
<p>Now the gateway  is fully configure.</p>
