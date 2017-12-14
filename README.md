# aws-iot

Creating certificate for device connectivity

AWS IoT only supports authenticated and secure connection. For the usage of certificate, AWS provides 3 options: bring-your-own, create with your CSR or one-click (with AWS IoT Cert Authority). For simplicity sake, we’ll use the one-click option for our device to communicate with AWS.

In the AWS IoT console, go to Security > Certificates > Create > Create Certificate (1st Option)

Save all 4 files into your device (i.e. Raspberry Pi). Don’t forget to download the root CA from Symantec (click here if you missed it).

Now, we need to create a policy and assign it to the certificate. Without a policy, your device is not authorised to perform any action (e.g. connect, publish).

Go to Security > Policies > Create > Create a policy

We’ll create a policy that allows all AWS IoT related actions to be performed. In a production environment, it should be done differently, by following security best practices – authorising only necessary actions.

Next, attach the policy to the certificate we created earlier. Go to Security > Certificates. Click on the “3 dots” on the certificate to bring out the option. Click “Attach a policy” and select the policy we created earlier. Activate the certificate if it is not already “Active”.
Setting up the Raspberry Pi and DHT11

Ensure that your device has internet connection.

Connect the DHT11 to the Raspberry Pi.

Install AWS IoT Python SDK

```bash
git clone https://github.com/aws/aws-iot-device-sdk-python
cd aws-iot-device-sdk-python
sudo python setup.py install
```
The following successful message should be seen, otherwise run the last step again.

Running install
…
Running install_egg_info
Removing…
Writing /usr/local/…

Download DHT11 Python library and create our script that will read data from the sensor and publish to AWS IoT.

```bash
git clone https://github.com/szazo/DHT11_Python
cd DHT11_Python
vi read_and_publish.py
```

Content of read_and_publish.py is as below. Change line 14 and 15 to reflect your broker address and path of the certificate. The broker address can be found in the AWS IoT console under Settings tab.

```python
import RPi.GPIO as GPIO
import dht11
from AWSIoTPythonSDK.MQTTLib import AWSIoTMQTTClient
from time import sleep
from datetime import date, datetime
 
# initialize GPIO
GPIO.setwarnings(False)
GPIO.setmode(GPIO.BCM)
GPIO.cleanup()
 
# AWS IoT certificate based connection
myMQTTClient = AWSIoTMQTTClient("123afhlss456")
myMQTTClient.configureEndpoint("A3XXX.iot.eu-west-1.amazonaws.com", 8883)
myMQTTClient.configureCredentials("/home/pi/cert/CA.pem", "/home/pi/cert/xxxx-private.pem.key", "/home/pi/cert/xxxxx-certificate.pem.crt")
myMQTTClient.configureOfflinePublishQueueing(-1)  # Infinite offline Publish queueing
myMQTTClient.configureDrainingFrequency(2)  # Draining: 2 Hz
myMQTTClient.configureConnectDisconnectTimeout(10)  # 10 sec
myMQTTClient.configureMQTTOperationTimeout(5)  # 5 sec
 
#connect and publish
myMQTTClient.connect()
myMQTTClient.publish("thing01/info", "connected", 0)
 
#loop and publish sensor reading
while 1:
    now = datetime.utcnow()
    now_str = now.strftime('%Y-%m-%dT%H:%M:%SZ') #e.g. 2016-04-18T06:12:25.877Z
    instance = dht11.DHT11(pin = 4) #BCM GPIO04
    result = instance.read()
    if result.is_valid():
        payload = '{ "timestamp": "' + now_str + '","temperature": ' + str(result.temperature) + ',"humidity": '+ str(result.humidity) + ' }'
        print payload
        myMQTTClient.publish("thing01/data", payload, 0)
        sleep(4)
    else:
        print (".")
        sleep(1)
```
