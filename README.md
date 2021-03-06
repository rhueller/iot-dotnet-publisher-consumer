﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿﻿

#  1.Objective

The aim of this repository is to provide code samples for simple IOT device publisher and IOT device consumer using .NET framework and .NET core.

# 2. Why .NET publisher and .NET consumer for AWS IOT

At this point in time, there is no AWS IOT device SDK for Microsoft C#. This does not impact the micro, small and medium sized IOT nodes leveraging AWS IOT Framework. The current plethora of IOT device SDKs offered by AWS is more than enough for micro, small and medium-large nodes. However, the large IOT nodes are typically massive enterprise servers running in a smart city use case or a large IIOT implementation in process industries such as petroleum, paper or pulp. There the very nature of network segmentation of IIOT architecture and presence of enterprise technology stack on these IIOT layers would necessitate to leverage a programming language such as C# or Java in implementing those IOT nodes. AWS already offers IOT device SDKs in Java. This post is all about the covering the edge case of implementing IOT device publisher and device consumer using Microsoft C#. 

# 3. AWS IOT device publisher and consumer using .NET framework


## 3a. Development environment
- Windows 10
- Visual Studio 2017 with latest updates
- Windows Subsystem for Linux 

## 3b. Create  an AWS IOT Thing

Let's create an IOT thing called 'dotnnetdevice' using the AWS IOT console.

![](/images/pic1.JPG)

![](/images/pic2.JPG)


Let's associate the following policy with the thing.

``` json

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "iot:Publish",
        "iot:Subscribe",
        "iot:Connect",
        "iot:Receive"
      ],
      "Effect": "Allow",
      "Resource": [
        "*"
      ]
    }
  ]
}

``` 
During the Thing creation process you should get the following four security artifacts

- **Device certificate** - This file usually ends with ".pem.crt". When you download this it will save as .txt file extension in windows. Save it as 'dotnet-devicecertificate.pem' for convenience and make sure that it is of file type '.pem', not 'txt' or '.crt'

- **Device public key** - This file usually ends with ".pem" and is of file type ".key".  Rename this file as 'dotnet-public.pem' for convenience. 

- **Device private key** -  This file usually ends with ".pem" and is of file type ".key".  Rename this file as 'dotnet-private.pem' for convenience. Make sure that this file is referred with suffix ".key" in the code while making MQTT connection to AWS IOT.

- **Root certificate** - The default name for this file is VeriSign-Class 3-Public-Primary-Certification-Authority-G5.pem and rename it as "root-CA.crt" for convenience.


##  3c. Converting device certificate from .pem to .pfx

In order to establish an MQTT connection with the AWS IOT platform, the root CA certificate, the private key of the thing and the certificate of the thing/device are needed. The .NET cryptographic APIs can understand root CA (.crt), device private key (.key) out-of-box. It expects the device certificate to be in the .pfx format, not the .pem format. Hence we need to convert the device certificate from .pem to .pfx.

We'll leverage the openssl for converting .pem to .pfx. Navigate to the folder where all the security artifacts are present and launch bash for Windows 10.

The syntax for converting .pem to .pfx is below :-

openssl pkcs12 -export -in **iotdevicecertificateinpemformat** -inkey **iotdevivceprivatekey** -out **devicecertificateinpfxformat** -certfile **rootcertificatefile**

If you replace with actual file names the syntax will look like below

openssl pkcs12 -export -in dotnet-devicecertificate.pem -inkey dotnet-private.pem.key -out dotnet_devicecertificate.pfx -certfile root-CA.crt

![](/images/pic3.JPG)




##  3d. Device publisher using .NET framework

Let's create a windows application in Visual Studio 2017 and name it as 'Iotpublisher'.

On project reference --> Manage Nuget pakcages --> Browse --> 'M2mqtt' and install M2mqtt.

Import the following namespaces.

```  c#

using uPLibrary.Networking.M2Mqtt;
using uPLibrary.Networking.M2Mqtt.Messages;
using System.Security;
using System.Security.Cryptography.X509Certificates;

```

Then create an instance of Mqtt client object with IOT endpoint, broker port for MQTT, X509Certificate object for root certificate, X5092certificate object for device certificate and Mqttsslprotocols enumeration for TLS1.2. 

Once the connection is successful publish to AWS IOT by specifying the  Topic and Payload. The following code snippet covers all of these :-


```  c#

string iotendpoint = "awsiotendpoint";
            int BrokerPort = 8883;
            string Topic = "Hello/World";

            var CaCert = X509Certificate.CreateFromCertFile(@"C:\Iotdevices\dotnetdevice\root-CA.crt");
            var ClientCert = new X509Certificate2(@"C:\Iotdevices\dotnetdevice\dotnet_devicecertificate.pfx", "password1");

            var Message = "Test message";
            string ClientId = Guid.NewGuid().ToString();

            var IotClient = new MqttClient(iotendpoint, BrokerPort, true, CaCert, ClientCert, MqttSslProtocols.TLSv1_2);

           
            IotClient.Connect(ClientId);
            Console.WriteLine("Connected");


            while (true)
            {
                IotClient.Publish(Topic, Encoding.UTF8.GetBytes(Message));
                Console.WriteLine("published" + Message);
                Thread.Sleep(5000);

            }
            
``` 

Hit F5 in visual studio and you should see the messages getting pushed to the AWS IOT Mqtt topic.

![](/images/pic5.JPG)


The complete visual studio solution for this publisher is available under the 'Dotnetsamples' folder in this repository. 


##  3e. Device consumer using .NET framework

Let's create a windows application in Visual Studio 2017 and name it as 'Iotconsumer'.

On project reference --> Manage Nuget pakcages --> Browse --> 'M2mqtt' and install M2mqtt.

Import the following namespaces.

```  c#
using uPLibrary.Networking.M2Mqtt;
using uPLibrary.Networking.M2Mqtt.Messages;
using System.Security;
using System.Security.Cryptography.X509Certificates;
using System.Threading;

```

Then create an instance of Mqtt client object with IOT endpoint, broker port for MQTT, X509Certificate object for root certificate, X5092certificate object for device certificate and Mqttsslprotocols enumeration for TLS1.2. 

You can subscribe to the AWS IOT messages by specifying the Topic as string array and Qos level as byte array. Prior to this event callbacks for MqttMsgSubscribed and MqttMsgPublishReceived should be implemented. The following code snippet covers all of that :-

```  c#

 static void Main(string[] args)
        {
            string IotEndPoint = "awsiotendpoint";

            int BrokerPort = 8883;
            string Topic = "Hello/World";

            var CaCert = X509Certificate.CreateFromCertFile(@"C:\Iotdevices\dotnetdevice\root-CA.crt");
            var clientCert = new X509Certificate2(@"C:\Iotdevices\dotnetdevice\dotnet_devicecertificate.pfx", "password1");


            string ClientID = Guid.NewGuid().ToString();

            var IotClient = new MqttClient(IotEndPoint, BrokerPort, true, CaCert, clientCert, MqttSslProtocols.TLSv1_2);
            IotClient.MqttMsgPublishReceived += Client_MqttMsgPublishReceived;
            IotClient.MqttMsgSubscribed += Client_MqttMsgSubscribed;

            IotClient.Connect(ClientID);
            Console.WriteLine("Connected");
            IotClient.Subscribe(new string[] { Topic }, new byte[] { MqttMsgBase.QOS_LEVEL_AT_LEAST_ONCE });

            while (true)
            {
                //keeping the main thread alive for the event call backs
            }

        }

        private static void Client_MqttMsgSubscribed(object sender, MqttMsgSubscribedEventArgs e)
        {
            Console.WriteLine("Message subscribed");
        }

        private static void Client_MqttMsgPublishReceived(object sender, MqttMsgPublishEventArgs e)
        {
            Console.WriteLine("Message Received is      " + System.Text.Encoding.UTF8.GetString(e.Message));
        }

``` 
The complete visual studio solution for this publisher is available under the 'Dotnetsamples' folder in this repository.

Hit F5 in visual studio and you should see the messages getting consumed by subscriber.

![](/images/pic6.JPG)

# 4. AWS IOT device publisher and consumer using .NET core

## 4a. Development environment

The following constitutes the development environment for developing AWS IOT device publisher and consumer using .NET core.

- Ubuntu 16.0.4 or higher (or) any other latest Linux distros
- .NET core 2.0 or higher
- AWS cli
- Openssl latest version


## 4b. Create an AWS IOT Thing 
Let's leverage the same IOT thing 'dotnetdevice' created in the section 3b for .NET core as well.

## 4c. Copying security artifacts to Linux environment 
Let's copy the device certificate, device private key and root certificate covered in the section 3c to the Linux the environment where we are going to develop IOT publisher and consumer using .NET core.

## 4d. Device publisher using .NET core 

Let's create the .NET core console application for the producer by issuing the following commands in the Terminal.


``` shell
mkdir Iotdotnetcorepublisher
cd Iotdotnetcorepublisher
dotnet new console
dotnet add package M2MqttClientDotnetCore --version 1.0.1
dotnet restore

```
Open the program.cs in Visual Studio and import the following namespaces.

``` c#

using System.Security;
using System.Security.Cryptography.X509Certificates;
using System.Text;
using System.Threading;
using M2Mqtt;
using M2Mqtt.Messages;
using M2Mqtt.Net;
using M2Mqtt.Utility;
using M2Mqtt.Internal;

``` 

Then perform a 'dotnet restore' in the terminal. It will grab the assemblies for System.Security.Cryptography.X509Certificates.

Then create an instance of Mqtt client object with IOT endpoint, broker port for MQTT, X509Certificate object for root certificate, X5092certificate object for device certificate and Mqttsslprotocols enumeration for TLS1.2. 

Once the connection is successful publish to AWS IOT by specifying the  Topic and Payload. The following code snippet covers all of these :-


``` c#

string IotEndPoint = "awsiotendpoint.iot.us-east-1.amazonaws.com";
            Console.WriteLine("AWS IOT Dotnet core message publiser starting");
            int BrokerPort = 8883;
            string Topic = "Hello/World";

            var CaCert = X509Certificate.CreateFromCertFile("/home/youruser/dotnetdevice/root-CA.crt");
            var ClientCert = new X509Certificate2("/home/youruser/dotnetdevice/dotnet_devicecertificate.pfx", "password1");

            var Message = "Test message";
            string ClientId = Guid.NewGuid().ToString();

            var IotClient = new MqttClient(IotEndPoint, BrokerPort, true, CaCert, ClientCert, MqttSslProtocols.TLSv1_2);

            IotClient.Connect(ClientId);
            Console.WriteLine("Connected to AWS IOT");


            

            while (true)
            {
                IotClient.Publish(Topic, Encoding.UTF8.GetBytes(Message));
                Console.WriteLine("Message published");
                Thread.Sleep(5000);

            }

``` 
Make sure that the appropriate linux path format is followed for accessing the root certificate and .pfx. It is mentioned in the above snippet. The comple .NET core project source for the publisher is available under the Dotnetcoresamples folder in this repository.

Run the application using 'dotnet run' and you should see messages published by dotnet core.

![](/images/pic7.png)



## 4e. Device consumer using .NET core 

Let's create the .NET core console application for the consumer by issuing the following commands in the Terminal.


``` shell
mkdir Iotdotnetcoreconsumer
cd Iotdotnetcoreconsumer
dotnet new console
dotnet add package M2MqttClientDotnetCore --version 1.0.1
dotnet restore

```
Open the program.cs in Visual Studio and import the following namespaces.

``` c#

using System.Security;
using System.Security.Cryptography.X509Certificates;
using System.Text;
using System.Threading;
using M2Mqtt;
using M2Mqtt.Messages;
using M2Mqtt.Net;
using M2Mqtt.Utility;
using M2Mqtt.Internal;

``` 

Then perform a 'dotnet restore' in the terminal. It will grab the assemblies for System.Security.Cryptography.X509Certificates.

Then create an instance of Mqtt client object with IOT endpoint, broker port for MQTT, X509Certificate object for root certificate, X5092certificate object for device certificate and Mqttsslprotocols enumeration for TLS1.2. 

You can subscribe to the AWS IOT messages by specifying the Topic as string array and Qos level as byte array. Prior to this event callbacks for MqttMsgSubscribed and MqttMsgPublishReceived should be implemented. The following code snippet covers all of that :-


``` c#

 static void Main(string[] args)
        {
          Console.WriteLine("AWS IOT dotnetcore message consumer starting");
            string IotEndPoint = "awsiotendpoint.amazonaws.com";
            int BrokerPort = 8883;
           string Topic = "Hello/World";
           
            var CaCert = X509Certificate.CreateFromCertFile("/home/youruser/dotnetdevice/root-CA.crt");
            var ClientCert = new X509Certificate2("/home/youruser/dotnetdevice/dotnet_devicecertificate.pfx", "password1");

          
            string ClientId = Guid.NewGuid().ToString();

            var IotClient = new MqttClient(IotEndPoint, BrokerPort, true, CaCert, ClientCert, MqttSslProtocols.TLSv1_2);

            IotClient.MqttMsgSubscribed += IotClient_MqttMsgSubscribed;
            IotClient.MqttMsgPublishReceived += IotClient_MqttMsgPublishReceived;

            IotClient.Connect(ClientId);

            Console.WriteLine("Connected to AWS IOT");
            IotClient.Subscribe(new string[] { Topic}, new byte[] {MqttMsgBase.QOS_LEVEL_AT_LEAST_ONCE });

            while (true)
            {
                //Keeping the mainthread alive for the event receivers to get invoked

            }

        }

              private static void IotClient_MqttMsgPublishReceived(object sender, MqttMsgPublishEventArgs e)
        {
            Console.WriteLine("Message recived is " + System.Text.Encoding.UTF8.GetString(e.Message));
        }


       private static void IotClient_MqttMsgSubscribed(object sender, MqttMsgSubscribedEventArgs e)
        {
            Console.WriteLine("Subscribed to the AWS IOT MQTT topic  ");
        }



``` 

The comple .NET core project source for the publisher is available under the Dotnetcoresamples folder in this repository.

Run the application using 'dotnet run' and you should see messages consumed by the dotnetcore

![](/images/pic8.png)






















