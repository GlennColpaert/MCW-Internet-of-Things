![Microsoft Cloud Workshops](https://github.com/Microsoft/MCW-Template-Cloud-Workshop/raw/master/Media/ms-cloud-workshop.png 'Microsoft Cloud Workshops')

<div class="MCWHeader1">
Internet of Things
</div>

<div class="MCWHeader2">
Hands-on lab step-by-step
</div>

<div class="MCWHeader3">
December 2019
</div>

Information in this document, including URL and other Internet Web site references, is subject to change without notice. Unless otherwise noted, the example companies, organizations, products, domain names, e-mail addresses, logos, people, places, and events depicted herein are fictitious, and no association with any real company, organization, product, domain name, e-mail address, logo, person, place or event is intended or should be inferred. Complying with all applicable copyright laws is the responsibility of the user. Without limiting the rights under copyright, no part of this document may be reproduced, stored in or introduced into a retrieval system, or transmitted in any form or by any means (electronic, mechanical, photocopying, recording, or otherwise), or for any purpose, without the express written permission of Microsoft Corporation.

Microsoft may have patents, patent applications, trademarks, copyrights, or other intellectual property rights covering subject matter in this document. Except as expressly provided in any written license agreement from Microsoft, the furnishing of this document does not give you any license to these patents, trademarks, copyrights, or other intellectual property.

The names of manufacturers, products, or URLs are provided for informational purposes only and Microsoft makes no representations and warranties, either expressed, implied, or statutory, regarding these manufacturers or the use of the products with any Microsoft technologies. The inclusion of a manufacturer or product does not imply endorsement of Microsoft of the manufacturer or product. Links may be provided to third party sites. Such sites are not under the control of Microsoft and Microsoft is not responsible for the contents of any linked site or any link contained in a linked site, or any changes or updates to such sites. Microsoft is not responsible for webcasting or any other form of transmission received from any linked site. Microsoft is providing these links to you only as a convenience, and the inclusion of any link does not imply endorsement of Microsoft of the site or the products contained therein.

© 2019 Microsoft Corporation. All rights reserved.

Microsoft and the trademarks listed at <https://www.microsoft.com/en-us/legal/intellectualproperty/Trademarks/Usage/General.aspx> are trademarks of the Microsoft group of companies. All other trademarks are property of their respective owners.

**Contents**

- [Internet of Things hands-on lab step-by-step](#internet-of-things-hands-on-lab-step-by-step)
  - [Abstract and learning objectives](#abstract-and-learning-objectives)
  - [Overview](#overview)
  - [Solution architecture](#solution-architecture)
  - [Requirements](#requirements)
  - [Exercise 1: IoT Hub provisioning](#exercise-1-iot-hub-provisioning)
    - [Task 1: Provision IoT Hub](#task-1-provision-iot-hub)
    - [Task 2: Configure the Smart Meter Simulator](#task-2-configure-the-smart-meter-simulator)
  - [Exercise 2: Completing the Smart Meter Simulator](#exercise-2-completing-the-smart-meter-simulator)
    - [Task 1: Implement device management with the IoT Hub](#task-1-implement-device-management-with-the-iot-hub)
    - [Task 2: Implement the communication of telemetry with IoT Hub](#task-2-implement-the-communication-of-telemetry-with-iot-hub)
    - [Task 3: Verify device registration and telemetry](#task-3-verify-device-registration-and-telemetry)
  - [Exercise 3: Hot path data processing with Stream Analytics](#exercise-3-hot-path-data-processing-with-stream-analytics)
    - [Task 1: Create a Stream Analytics job for hot path processing to Power BI](#task-1-create-a-stream-analytics-job-for-hot-path-processing-to-power-bi)
    - [Task 2: Visualize hot data with Power BI](#task-2-visualize-hot-data-with-power-bi)
  - [Exercise 4: Sending commands to the IoT devices](#exercise-5-sending-commands-to-the-iot-devices)
    - [Task 1: Add your IoT Hub connection string to the CloudToDevice console app](#task-1-add-your-iot-hub-connection-string-to-the-cloudtodevice-console-app)
    - [Task 2: Run the device simulator](#task-2-run-the-device-simulator)
    - [Task 3: Run the console app and send cloud-to-device messages](#task-3-run-the-console-app-and-send-cloud-to-device-messages)
  - [After the hands-on lab](#after-the-hands-on-lab)
    - [Task 1: Delete the resource group](#task-1-delete-the-resource-group)

# Internet of Things hands-on lab step-by-step

If you have not yet completed the steps to set up your environment in [Before the hands-on lab setup guide](./Before%20the%20HOL%20-%20Internet%20of%20Things.md), you will need to do that before proceeding.

## Abstract and learning objectives

In this hands-on lab, you will construct an end-to-end IoT solution simulating high velocity data emitted from smart meters and analyzed in Azure. You will design a lambda architecture, filtering a subset of the telemetry data for real-time visualization on the hot path, and storing all the data in long-term storage for the cold path.

At the end of this hands-on lab, you will be better able to build an IoT solution implementing device registration with the IoT Hub Device Provisioning Service and visualizing hot data with Power BI.

## Overview

Fabrikam provides services and smart meters for enterprise energy (electrical power) management. Their "_You-Left-The-Light-On_" service enables the enterprise to understand their energy consumption.

## Solution architecture

Below is a diagram of the solution architecture you will build in this lab. Please study this carefully, so you understand the whole of the solution as you are working on the various components.

![Diagram of the preferred solution. From a high-level, the commerce solution uses an API App to host the Payments web service with which the Vending Machine interacts to conduct purchase transactions. The Payment Web API invokes a 3rd party payment gateway as needed for authorizing and capturing credit card payments, and logs the purchase transaction to SQL DB. The data for these purchase transactions is stored using an In-Memory table with a Columnar Index, which will support the write-heavy workload while still allowing analytics to operate, such as queries coming from Power BI Desktop.](./media/preferred-solution-architecture.png 'Preferred high-level architecture')

Messages are ingested from the Smart Meters via IoT Hub and temporarily stored there. A Stream Analytics job pulls telemetry messages from IoT Hub and sends the messages to two different destinations. There are two Stream Analytics jobs, one that retrieves all messages and sends them to Blob Storage (the cold path), and another that selects out only the important events needed for reporting in real time (the hot path). Data entering the hot path will be reported on using Power BI visualizations and reports. For the cold path, Azure Databricks can be used to apply the batch computation needed for the reports at scale.

Other alternatives for processing of the ingested telemetry would be to use an HDInsight Storm cluster, a WebJob running the EventProcessorHost in place of Stream Analytics, or HDInsight running with Spark streaming. Depending on the type of message filtering being conducted for hot and cold stream separation, IoT Hub Message Routing might also be used, but this has the limitation that messages follow a single path, so with the current implementation, it would not be possible to send all messages to the cold path, while simultaneously sending some of the same messages into the hot path. An important limitation to keep in mind for Stream Analytics is that it is very restrictive on the format of the input data it can process: the payload must be UTF8 encoded JSON, UTF8 encoded CSV (fields delimited by commas, spaces, tabs, or vertical pipes), or AVRO, and it must be well-formed. If any devices transmitting telemetry cannot generate output in these formats (e.g., because they are legacy devices), or their output can be not well formed at times, then alternatives that can better deal with these situations should be investigated. Additionally, any custom code or logic cannot be embedded with Stream Analytics---if greater extensibility is required, the alternatives should be considered.

> **Note**: The preferred solution is only one of many possible, viable approaches.

## Requirements

- Microsoft Azure subscription must be pay-as-you-go or MSDN.
  - Trial subscriptions will not work.
- A virtual machine configured with:
  - Visual Studio Community 2019 or later
  - Azure SDK 2.9 or later (Included with Visual Studio)
- A running Azure Databricks cluster (see [Before the hands-on lab](./Before%20the%20HOL%20-%20Internet%20of%20Things.md))

## Exercise 1: IoT Hub provisioning

Duration: 15 minutes

In your architecture design session with Fabrikam, it was agreed that you would use an Azure IoT Hub to manage both the device registration and telemetry ingest from the Smart Meter Simulator. Your team also identified the Microsoft provided Device Explorer project that Fabrikam can use to view the list and status of devices in the IoT Hub registry.

### Task 1: Provision IoT Hub

In these steps, you will provision an instance of IoT Hub.

1. In your browser, navigate to the [Azure portal](https://portal.azure.com), select **+Create a resource** in the navigation pane, enter "iot" into the Search the Marketplace box.

2. Select **IoT Hub** from the results, and then select **Create**.

   ![+Create a resource is highlighted in the navigation page of the Azure portal, and "iot" is entered into the Search the Marketplace box. IoT Hub is highlighted in the search results.](./media/create-resource-iot-hub.png 'Create an IoT Hub')

3. On the IoT Hub blade Basics tab, enter the following:

   - **Subscription**: Select the subscription you are using for this hands-on lab.

   - **Resource group**: Choose Use existing and select the hands-on-lab-SUFFIX resource group.

   - **Region**: Select the location you are using for this hands-on lab.

   - **IoT Hub Name**: Enter a unique name, such as smartmeter-hub-SUFFIX.

     ![The Basics blade for IoT Hub is displayed, with the values specified above entered into the appropriate fields.](./media/iot-hub-basics-blade.png 'Create IoT Hub Basic blade')

   - Select **Next: Size and Scale**.

   - On the Size and scale blade, accept the default Pricing and scale tier of S1: Standard tier, and select **Review + create**.

   - Select **Create** on the Review + create blade.

4. When the IoT Hub deployment is completed, you will receive a notification in the Azure portal. Select **Go to resource** in the notification.

   ![Screenshot of the Deployment succeeded message, with the Go to resource button highlighted.](./media/iot-hub-deployment-succeeded.png 'Deployment succeeded message')

5. From the IoT Hub's Overview blade, select **Shared access policies** under Settings on the left-hand menu.

   ![Screenshot of the Overview blade, settings section. Under Settings, Shared access policies is highlighted.](./media/iot-hub-shared-access-policies.png 'Overview blade, settings section')

6. Select **iothubowner** policy.

   ![The Azure portal is shown with the iothubowner selected.](./media/iot-hub-shared-access-policies-iothubowner.png 'IoT Hub Owner shared access policy')

7. In the **iothubowner** blade, select the Copy button to the right of the **Connection string - primary key** field. You will be pasting the connection string value into a TextBox's Text property value in the next task.

   ![Screenshot of the iothubowner blade. The connection string - primary key field is highlighted.](./media/iot-hub-shared-access-policies-iothubowner-blade.png 'iothubowner blade')

### Task 2: Configure the Smart Meter Simulator

If you want to save this connection string with your project (in case you stop debugging or otherwise close the simulator), you can set this as the default text for the text box. Follow these steps to configure the connection string:

1. Return to the `SmartMeterSimulator` solution in Visual Studio on your Lab VM.

2. In the Solution Explorer, expand the SmartMeterSimulator project and double-click `MainForm.cs` to open it. (If the Solution Explorer is not in the upper-right corner of your Visual Studio instance, you can find it under the View menu in Visual Studio.)

    > **Note**:  If the form does not display, it is due to the `TODO` tasks in the code.  You can remove them in the following steps or simply add the following code to the `MainForm.Designer.cs` on line 260

    ```csharp
    this.txtIotHubCnString.Text = "YOUR CONNECTION STRING";
    ```

   ![In the Visual Studio Solution Explorer window, SmartMeterSimulator is expanded, and under it, MainForm.cs is highlighted.](media/visual-studio-solution-explorer-mainform-cs.png 'Visual Studio Solution Explorer')

3. In the Windows Forms designer surface, select the **IoT Hub Connection String TextBox** to select it.

   ![The Windows Form designer surface is opened to the MainForm.cs tab. The IoT Hub Connection String is highlighted, but is empty.](./media/smart-meter-simulator-iot-hub-connection-string.png 'Windows Form designer surface')

4. In the Properties panel, scroll until you see the **Text** property. Paste your IoT Hub connection string value copied in step 6 of the previous task into the value for the Text property. (If the properties window is not visible below the Solution Explorer, right-click the TextBox, and select **Properties**.)

   ![In the Properties panel, the Text property is highlighted, and is set to HostName=smartmeter-hub.](./media/smart-meter-simulator-iot-hub-connection-string-text-property.png 'Solution Explorer')

5. Your connection string should now be present every time you run the Smart Meter Simulator.

   ![The Windows Form designer surface is opened to the MainForm.cs tab. The IoT Hub Connection String now displays.](./media/smart-meter-simulator-iot-hub-connection-string-populated.png 'IoT Hub Connection String dialog')

6. Save `MainForm.cs`.

## Exercise 2: Completing the Smart Meter Simulator

Duration: 60 minutes

Fabrikam has left you a partially completed sample in the form of the Smart Meter Simulator solution. You will need to complete the missing lines of code that deal with device registration management and device telemetry transmission that communicate with your IoT Hub.

### Task 1: Implement device management with the IoT Hub

1. In Visual Studio on your Lab VM, use Solution Explorer to open the file `DeviceManager.cs`.

2. From the Visual Studio **View** menu, choose **Task List**.

   ![On the Visual Studio View menu, Task List is selected.](media/visual-studio-view-menu-task-list.png 'Visual Studio View menu')

3. In the Task List, you will see a list of `TODO` tasks, where each task represents one line of code that needs to be completed. Complete the line of code below each `TODO` using the code below as a reference.

4. The following code represents the completed tasks in `DeviceManager.cs`:

   ```csharp
   class DeviceManager
   {
       static string connectionString;
       static RegistryManager registryManager;

       public static string HostName { get; set; }

       public static void IotHubConnect(string cnString)
       {
           connectionString = cnString;

           //TODO: 1.Create an instance of RegistryManager from connectionString
           registryManager = RegistryManager.CreateFromConnectionString(connectionString);

           var builder = IotHubConnectionStringBuilder.Create(cnString);

           HostName = builder.HostName;
       }

       /// <summary>
       /// Register a single device with the IoT hub. The device is initially registered in a
       /// disabled state.
       /// </summary>
       /// <param name="connectionString"></param>
       /// <param name="deviceId"></param>
       /// <returns></returns>
       public async static Task<string> RegisterDevicesAsync(string connectionString, string deviceId)
       {
           //Make sure we're connected
           if (registryManager == null)
               IotHubConnect(connectionString);

           //TODO: 2.Create new device
           Device device = new Device(deviceId);

           //TODO: 3.Initialize device with a status of Disabled
           //Enabled in a subsequent step
           device.Status = DeviceStatus.Disabled;

           try
           {
               //TODO: 4.Register the new device
               device = await registryManager.AddDeviceAsync(device);
           }
           catch (Exception ex)
           {
               if (ex is DeviceAlreadyExistsException ||
                   ex.Message.Contains("DeviceAlreadyExists"))
               {
                   //TODO: 5.Device already exists, get the registered device
                   device = await registryManager.GetDeviceAsync(deviceId);

                   //TODO: 6.Ensure the device is disabled until Activated later
                   device.Status = DeviceStatus.Disabled;

                   //TODO: 7.Update IoT Hubs with the device status change
                   await registryManager.UpdateDeviceAsync(device);
               }
               else
               {
                   MessageBox.Show($"An error occurred while registering one or more devices:\r\n{ex.Message}");
               }
           }

           //return the device key
           return device.Authentication.SymmetricKey.PrimaryKey;
       }

       /// <summary>
       /// Activate an already registered device by changing its status to Enabled.
       /// </summary>
       /// <param name="connectionString"></param>
       /// <param name="deviceId"></param>
       /// <param name="deviceKey"></param>
       /// <returns></returns>
       public async static Task<bool> ActivateDeviceAsync(string connectionString, string deviceId, string deviceKey)
       {
           //Server-side management function to enable the provisioned device
           //to connect to IoT Hub after it has been installed locally.
           //If device id and device key are valid, Activate (enable) the device.

           //Make sure we're connected
           if (registryManager == null)
               IotHubConnect(connectionString);

           bool success = false;
           Device device;

           try
           {
               //TODO: 8.Fetch the device
               device = await registryManager.GetDeviceAsync(deviceId);

               //TODO: 9.Verify the device keys match
               if (deviceKey == device.Authentication.SymmetricKey.PrimaryKey)
               {
                   //TODO: 10.Enable the device
                   device.Status = DeviceStatus.Enabled;

                   //TODO: 11.Update IoT Hubs
                   await registryManager.UpdateDeviceAsync(device);

                   success = true;
               }
           }
           catch(Exception)
           {
               success = false;
           }

           return success;
       }

       /// <summary>
       /// Deactivate a single device in the IoT Hub registry.
       /// </summary>
       /// <param name="connectionString"></param>
       /// <param name="deviceId"></param>
       /// <returns></returns>
       public async static Task<bool> DeactivateDeviceAsync(string connectionString, string deviceId)
       {
           //Make sure we're connected
           if (registryManager == null)
               IotHubConnect(connectionString);

           bool success = false;
           Device device;

           try
           {
               //TODO: 12.Lookup the device from the registry by deviceId
               device = await registryManager.GetDeviceAsync(deviceId);

               //TODO: 13.Disable the device
               device.Status = DeviceStatus.Disabled;

               //TODO: 14.Update the registry
               await registryManager.UpdateDeviceAsync(device);

               success = true;
           }
           catch (Exception)
           {
               success = false;
           }

           return success;
       }

       /// <summary>
       /// Unregister a single device from the IoT Hub Registry
       /// </summary>
       /// <param name="connectionString"></param>
       /// <param name="deviceId"></param>
       /// <returns></returns>
       public async static Task UnregisterDevicesAsync(string connectionString, string deviceId)
       {
           //Make sure we're connected
           if (registryManager == null)
               IotHubConnect(connectionString);

               //TODO: 15.Remove the device from the Registry
               await registryManager.RemoveDeviceAsync(deviceId);
       }

       /// <summary>
       /// Unregister all the devices managed by the Smart Meter Simulator
       /// </summary>
       /// <param name="connectionString"></param>
       /// <returns></returns>
       public async static Task UnregisterAllDevicesAsync(string connectionString)
       {
           //Make sure we're connected
           if (registryManager == null)
              IotHubConnect(connectionString);

           for(int i = 0; i <= 9; i++)
           {
               string deviceId = "Device" + i.ToString();

               //TODO: 16.Remove the device from the Registry
               await registryManager.RemoveDeviceAsync(deviceId);
           }
       }
   }
   ```

   >**Note**:  Be sure you only replace the DeviceManager class and not any other code in the file.

5. Save `DeviceManager.cs`.

### Task 2: Implement the communication of telemetry with IoT Hub

1. Open `Sensor.cs` from the Solution Explorer, and complete the `TODO` items indicated within the code that are responsible for transmitting telemetry data to the IoT Hub, as well as receiving data from IoT Hub.

2. The following code shows the completed result:

   ```csharp
   class Sensor
    {
        private DeviceClient _DeviceClient;
        private string _IotHubUri { get; set; }
        public string DeviceId { get; set; }
        public string DeviceKey { get; set; }
        public DeviceState State { get; set; }
        public string StatusWindow { get; set; }
        public string ReceivedMessage { get; set; }
        public double? ReceivedTemperatureSetting { get; set; }
        public double CurrentTemperature
        {
            get
            {
                double avgTemperature = 70;
                Random rand = new Random();
                double currentTemperature = avgTemperature + rand.Next(-6, 6);

                if (ReceivedTemperatureSetting.HasValue)
                {
                    // If we received a cloud-to-device message that sets the temperature, override with the received value.
                    currentTemperature = ReceivedTemperatureSetting.Value;
                }

                if(currentTemperature <= 68)
                    TemperatureIndicator = SensorState.Cold;
                else if(currentTemperature > 68 && currentTemperature < 72)
                    TemperatureIndicator = SensorState.Normal;
                else if(currentTemperature >= 72)
                    TemperatureIndicator = SensorState.Hot;

                return currentTemperature;
            }
        }
        public SensorState TemperatureIndicator { get; set; }

        public Sensor(string iotHubUri, string deviceId, string deviceKey)
        {
            _IotHubUri = iotHubUri;
            DeviceId = deviceId;
            DeviceKey = deviceKey;
            State = DeviceState.Registered;
        }
        public void InstallDevice(string statusWindow)
        {
            StatusWindow = statusWindow;
            State = DeviceState.Installed;
        }

        /// <summary>
        /// Connect a device to the IoT Hub by instantiating a DeviceClient for that Device by Id and Key.
        /// </summary>
        public void ConnectDevice()
        {
            //TODO: 17. Connect the Device to Iot Hub by creating an instance of DeviceClient
            _DeviceClient = DeviceClient.Create(_IotHubUri, new DeviceAuthenticationWithRegistrySymmetricKey(DeviceId, DeviceKey));

            //Set the Device State to Ready
            State = DeviceState.Ready;
        }
        public void DisconnectDevice()
        {
            //Delete the local device client
            _DeviceClient = null;

            //Set the Device State to Activate
            State = DeviceState.Activated;
        }

        /// <summary>
        /// Send a message to the IoT Hub from the Smart Meter device
        /// </summary>
        public async void SendMessageAsync()
        {
            var telemetryDataPoint = new
            {
                id = DeviceId,
                time = DateTime.UtcNow.ToString("o"),
                temp = CurrentTemperature
            };

            //TODO: 18.Serialize the telemetryDataPoint to JSON
            var messageString = JsonConvert.SerializeObject(telemetryDataPoint);

            //TODO: 19.Encode the JSON string to ASCII as bytes and create new Message with the bytes
            var message = new Message(Encoding.ASCII.GetBytes(messageString));

            //TODO: 20.Send the message to the IoT Hub
            var sendEventAsync = _DeviceClient?.SendEventAsync(message);
            if (sendEventAsync != null) await sendEventAsync;
        }

        /// <summary>
        /// Check for new messages sent to this device through IoT Hub.
        /// </summary>
        public async void ReceiveMessageAsync()
        {
            try
            {
                Message receivedMessage = await _DeviceClient?.ReceiveAsync();
                if (receivedMessage == null)
                {
                    ReceivedMessage = null;
                    return;
                }

                //TODO: 21.Set the received message for this sensor to the string value of the message byte array
                ReceivedMessage = Encoding.ASCII.GetString(receivedMessage.GetBytes());
                if(double.TryParse(ReceivedMessage, out var requestedTemperature))
                {
                    ReceivedTemperatureSetting = requestedTemperature;
                }
                else
                {
                    ReceivedTemperatureSetting = null;
                }

                // Send acknowledgement to IoT Hub that the has been successfully processed.
                // The message can be safely removed from the device queue. If something happened
                // that prevented the device app from completing the processing of the message,
                // IoT Hub delivers it again.

                //TODO: 22.Send acknowledgement to IoT hub that the message was processed
                await _DeviceClient?.CompleteAsync(receivedMessage);
            }
            catch (NullReferenceException ex)
            {
                // The device client is null, likely due to it being disconnected since this method was called.
                System.Diagnostics.Debug.WriteLine("The DeviceClient is null. This is likely due to it being disconnected since the ReceiveMessageAsync message was called.");
            }
        }
    }
   ```

    > **Note**:  Be sure you only replace the Sensor class and not any other code in the file.

3. Save `Sensor.cs`.

### Task 3: Verify device registration and telemetry

In this task, you will build and run the Smart Meter Simulator project.

1. In Visual Studio select **Build** from the Visual Studio menu, then select **Build Solution**.

   > **Note**: If you receive the following error after building the solution, complete the steps that follow: `Couldn't process file MainForm.resx due to its being in the Internet or Restricted zone or having the mark of the web on the file. Remove the mark of the web if you want to process these files.`

   - Open Windows Explorer and navigate to the starter project folder: `C:\SmartMeter\Hands-on lab\lab-files\starter-project\SmartMeterSimulator\`.
   - Right-click on the `MainForm.resx` file, then select **Properties**.
   - Check the **Unblock** checkbox on the bottom of the General tab, then select Apply then OK.

   ![Right-click MainForm.resx, go to Properties, then check the box next to Unblock](media/unblock-file.png 'Unblock file')

   - Close and reopen Visual Studio. You should now be able to build successfully.

2. Run the Smart Meter Simulator, by selecting the green Start button on the Visual Studio toolbar.

   ![The green Start button is highlighted on the Visual Studio toolbar.](media/visual-studio-toolbar-start.png 'Visual Studio toolbar')

3. Select **Register** on the Smart Meter Simulator dialog, which should cause the windows within the building to change from black to gray.

   ![In addition to the IoT Hub Connection String, the Smart Meter Simulator has two buildings with 10 windows. The color of the windows indicating the status of the devices. Currently, all windows are gray.](media/smart-meter-simulator-register.png 'Fabrikam Smart Meter Simulator')

4. Select a few of the windows. Each represents a device for which you want to simulate device installation. The selected windows should turn yellow.

   ![The Smart Meter Simulator now has three white windows, with the other seven remaining gray.](media/smart-meter-simulator-window-select.png 'Fabrikam Smart Meter Simulator')

5. Select **Activate** to simulate changing the device status from disabled to enabled in the IoT Hub Registry. The selected windows should turn green.

   ![On the Smart Meter Simulator, the Activate button is highlighted, and the three white windows have now turned to green.](media/smart-meter-simulator-activate.png 'Fabrikam Smart Meter Simulator')

6. At this point, you have registered 10 devices (the gray windows) but activated only the ones you selected (in green). To view this list of devices, you will switch over to the Azure Portal, and open the IoT Hub you provisioned.

7. From the IoT Hub blade, select **IoT Devices** under Explorers on the left-hand menu.

   ![On the IoT Hub blade, in the Explorers section, under Explorers, IoT Devices is highlighted.](media/iot-hub-explorers-iot-devices.png 'IoT Hub blade, Explorers section')

8. You should see all 10 devices listed, with the ones that you activated having a status of **enabled**.

   ![Devices in the Device ID list have a status of either enabled or disabled.](media/iot-hub-iot-devices-list.png 'Device ID list')

9. Return to the **Smart Meter Simulator** window.

10. Select **Connect**. Within a few moments, you should begin to see activity as the windows change color, indicating the smart meters are transmitting telemetry. The grid on the left will list each telemetry message transmitted and the simulated temperature value.

    ![On the Smart Meter Simulator, the Connect button is highlighted, and one of the green windows has now turned to blue. The current windows count is now seven gray, two green, and one blue.](media/smart-meter-simulator-connect.png 'Fabrikam Smart Meter Simulator')

11. Allow the smart meter to continue to run. (Whenever you want to stop the transmission of telemetry, select the **Disconnect** button.)

## Exercise 3: Hot path data processing with Stream Analytics

Duration: 45 minutes

Fabrikam would like to visualize the "hot" data showing the average temperature reported by each device over a 5-minute window in Power BI.

### Task 1: Create a Stream Analytics job for hot path processing to Power BI

1. In the [Azure Portal](https://portal.azure.com), select **+ Create a resource**, enter "stream analytics" into the Search the Marketplace box, select **Stream Analytics job** from the results, and select **Create**.

   ![In the Azure Portal, +Create a resource is highlighted, "stream analytics" is entered into the Search the Marketplace box, and Stream Analytics job is highlighted in the results.](media/create-resource-stream-analytics-job.png 'Create Stream Analytics job')

2. On the New Stream Analytics Job blade, enter the following:

   - **Job name**: Enter hot-stream.
   - **Subscription**: Select the subscription you are using for this hands-on lab.
   - **Resource group**: Choose Use existing and select the hands-on-lab-SUFFIX resource group.
   - **Location**: Select the location you are using for resources in this hands-on lab.
   - **Hosting environment**: Select **Cloud**.
   - **Streaming units**: Change the value to **1** by sliding the slider all the way left.

     ![The New Stream Analytics Job blade is displayed, with the previously mentioned settings entered into the appropriate fields.](media/stream-analytics-job-create.png 'New Stream Analytics Job blade')

3. Select **Create**.

4. Once provisioned, navigate to your new Stream Analytics job in the portal.

5. On the Stream Analytics job blade, select **Inputs** from the left-hand menu, under Job Topology, then select **+Add stream input**, and select **IoT Hub** from the dropdown menu to add an input connected to your IoT Hub.

   ![On the Stream Analytics job blade, Inputs is selected under Job Topology in the left-hand menu, and +Add stream input is highlighted in the Inputs blade, and IoT Hub is highlighted in the drop down menu.](media/stream-analytics-job-inputs-add.png 'Add Stream Analytics job inputs')

6. On the New Input blade, enter the following:

   - **Input alias**: Enter temps.
   - Choose **Select IoT Hub from your subscriptions**.
   - **Subscription**: Select the subscription you are using for this hands-on lab.
   - **IoT Hub**: Select the smartmeter-hub-SUFFIX IoT Hub.
   - **Endpoint**: Select Messaging.
   - **Shared access policy name**: Select **service**.
   - **Consumer Group**: Leave set to **\$Default**.
   - **Event serialization format**: Select **JSON**.
   - **Encoding**: Select **UTF-8**.
   - **Event compression type**: Leave set to **None**.

     ![IoT Hub New Input blade is displayed with the values specified above entered into the appropriate fields.](media/stream-analytics-job-inputs-add-iot-hub-input.png 'IoT Hub New Input blade')

7. Select **Save**.

8. Next, select **Outputs** from the left-hand menu, under Job Topology, and select **+ Add**, then select **Power BI** from the drop-down menu.

   ![Outputs is highlighted in the left-hand menu, under Job Topology, +Add is selected, and Power BI is highlighted in the drop down menu.](media/stream-analytics-job-outputs-add-power-bi.png 'Add Power BI Output')

9. In the **Power BI** blade, select **Authorize** to authorize the connection to your Power BI account. When prompted in the popup window, enter the account credentials you used to create your Power BI account in [Before the hands-on lab setup guide, Task 1](./Before%20the%20HOL%20-%20Internet%20of%20Things.md).

10. Once authorized, enter the following:

    - **Output alias**: Set to **powerbi**.

    - For the remaining Power BI settings, enter the following:

      - **Group Workspace**: Select the default, My Workspace.
      - **Dataset Name**: Enter avgtemps.
      - **Table Name**: Enter avgtemps.
      - **Authentication mode**: Enter **User token**.

    ![Power BI new output blade. Output alias is selected and contains powerbi. Authorize button is highlighted.](media/stream-analytics-job-outputs-add-power-bi-authorize.png 'Power BI new output blade')

11. Select **Save**.

    ![Power BI blade. Output alias is powerbi, dataset name is avgtemps, table name is avgtemps.](media/stream-analytics-job-outputs-add-power-bi-save.png 'Add Power BI Output')

12. Next, select **Query** from the left-hand menu, under Job Topology.

    ![Under Job Topology, Query is selected.](./media/stream-analytics-job-query.png 'Stream Analytics Query')

13. In the query text box, paste the following query.

    ```sql
    SELECT AVG(temp) AS Average, id
    INTO powerbi
    FROM temps
    GROUP BY TumblingWindow(minute, 5), id
    ```

14. Select **Save query**.

    ![Save button on the Query blade is highlighted](./media/stream-analytics-job-query-save.png 'Query Save button')

15. Return to the Overview blade on your Stream Analytics job and select **Start**.

    ![The Start button is highlighted on the Overview blade.](./media/stream-analytics-job-start.png 'Overview blade start button')

16. In the Start job blade, select **Now** (the job will start processing messages from the current point in time onward).

    ![Now is selected on the Start job blade.](./media/stream-analytics-job-start-job.png 'Start job blade')

17. Select **Start**.

18. Allow your Stream Analytics Job a few minutes to start.

19. Once the Stream Analytics Job has successfully started, verify that you are showing a non-zero amount of **Input Events** on the **Monitoring** chart on the overview blade. You may need to reconnect your devices on the Smart Meter Simulator and let it run for a while to see the events.

    ![The Stream Analytics job monitoring chart is diplayed with a non-zero amount of input events highlighted.](media/stream-analytics-job-monitoring-input-events.png 'Monitoring chart for Stream Analytics job')

### Task 2: Visualize hot data with Power BI

1. Sign in to your Power BI subscription (<https://app.powerbi.com>) to see if data is being collected.

2. Select **My Workspace** on the left-hand menu, then select the **Datasets tab**, and locate the **avgtemps** dataset from the list. **Note:** Sometimes it takes few minutes for the dataset to appear in the Power BI Dataset tab under "My Workspace"

   ![On the Power BI window, My Workspace is highlighted in the left pane, and the Datasets tab is highlighted in the right pane, and the avgtemps dataset is highlighted.](media/power-bi-workspaces-datasets-avgtemps.png 'Power BI Datasets')

3. Select the Create Report button under the Actions column.

   ![On the Datasets tab, under Actions, the Create Report button is highlighted.](./media/power-bi-datasets-avgtemps-create-report.png 'Datasets tab, Action column')

4. On the Visualizations palette, select **Stacked column chart** to create a chart visualization.

   ![On the Visualizations palette, the stacked column chart icon is highlighted.](./media/power-bi-visualizations-stacked-column-chart.png 'Visualizations palette')

5. In the Fields listing, drag the **id** field, and drop it into the **Axis** field.

   ![Under Fields, an arrow points from the id field under avgtemps, to the same id field now located in the Visualizations listing, under Axis.](./media/power-bi-visualizations-stacked-column-chart-axis.png 'Visualizations and Fields')

6. Next, drag the **average** field and drop it into the **Value** field.

   ![Under Fields, an arrow points from the average field under avgtemps, to the same id field now located in the Visualizations listing, under Value.](./media/power-bi-visualizations-stacked-column-chart-value.png 'Visualizations and Fields')

7. Now, set the Value to **Max of average**, by selecting the down arrow next to **average**, and select **Maximum**.

   ![On the Value drop-down list, Maximum is highlighted.](./media/power-bi-visualizations-stacked-column-chart-value-maximum.png 'Value drop-down list')

8. Repeat steps 5-8, this time adding a Stacked Column Chart for **Min of average**. (You may need to select on any area of white space on the report designer surface to deselect the Max of average by id chart visualization.)

   ![Min of average is added under Value.](./media/power-bi-visualizations-stacked-column-chart-value-minimum.png 'Min of average')

9. Next, add a **table visualization**.

   ![On the Visualizations palette, the table icon is highlighted.](./media/power-bi-visualizations-table.png 'Visualizations pallete')

10. Set the values to **id** and **Average of average**, by dragging and dropping both fields in the Values field, then selecting the dropdown next to average, and selecting Average.

    ![ID and Average of average now display under Values.](./media/power-bi-visualizations-table-average-of-average.png 'Table Visualization values')

11. Save the report.

    ![Under File, Save is highlighted.](media/power-bi-save-report.png 'Save report')

12. Enter the name "Average Temperatures," and select **Save**.

    ![The report name is set to Average Temperatures.](./media/power-bi-save-report-average-temperatures.png 'Save your report')

13. Switch to **Reading View**.

    ![Power BI report Reading view button](media/power-bi-report-reading-view.png 'Reading view button')

14. Within the report, select one of the columns to see the data for just that device.

    ![The report window has two bar graphs: Max of average by id, and Min of average by id. both bar charts list data for Device0, Device1, Device3, Device8, and Device9. Device1 is selected. On the right, a table displays data for Device1, with an Average of average value of 69.50.](./media/power-bi-report-reading-view-single-device.png 'Report window')

## Exercise 4: Sending commands to the IoT devices

Duration: 20 minutes

Fabrikam would like to send commands to devices from the cloud in order to control their behavior. In this exercise, you will send commands that control the temperature settings of individual devices.

### Task 1: Add your IoT Hub connection string to the CloudToDevice console app

This console app is configured to connect to IoT Hub using the same connection string you use in the SmartMeterSimulator app. Messages are sent from the console app to IoT Hub, specifying a device by its ID, for example `Device1`. IoT Hub then transmits that message to the device when it is connected. This is called a "cloud-to-device" message. The console app is not directly connecting to the device and sending it the message. All messages flow through IoT Hub where the connections and device state are managed.

1. Return to the `SmartMeterSimulator` solution in Visual Studio on your Lab VM.

2. In the Solution Explorer, expand the CloudToDevice project and double-click `Program.cs` to open it. (If the Solution Explorer is not in the upper-right corner of your Visual Studio instance, you can find it under the View menu in Visual Studio.)

    ![In the Visual Studio Solution Explorer window, CloudToDevice is expanded, and under it, Program.cs is highlighted.](media/visual-studio-solution-explorer-program-cs.png 'Visual Studio Solution Explorer')

3. Replace `YOUR-CONNECTION-STRING` on line 15 with your IoT Hub connection string. This is the same string you added to the Main form in the SmartMeterSimulator earlier. The line you need to update looks like this:

    ```csharp
    static string connectionString = "YOUR-CONNECTION-STRING";
    ```

    After updating, your Program.cs file should look similar to the following:

    ![The Program.cs file has been updated with the code change.](media/visual-studio-program-cs.png 'Program.cs')

4. Save the file.

### Task 2: Run the device simulator

In this task, you will register, activate, and connect all devices. You will then leave the simulator running so that you can launch the console app and start sending cloud-to-device messages.

1. Within the `SmartMeterSimulator` Visual Studio solution, right-click the **SmartMeterSimulator** project, select **Debug**, then select **Start new instance** to run the device simulator.

    ![Screenshot displays the debug context menu after right-clicking the SmartMeterSimulator project in Visual Studio.](media/visual-studio-debug-simulator.png 'Debug simulator')

2. Select **Register** on the Smart Meter Simulator dialog, which should cause the windows within the building to change from black to gray.

    ![In addition to the IoT Hub Connection String, the Smart Meter Simulator has two buildings with 10 windows. The color of the windows indicating the status of the devices. Currently, all windows are gray.](media/smart-meter-simulator-register.png 'Fabrikam Smart Meter Simulator')

3. Select **all** of the windows. Each represents a device for which you want to simulate device installation. The selected windows should turn yellow.

    ![The Smart Meter Simulator now has all yellow windows.](media/smart-meter-simulator-window-select-all.png 'Fabrikam Smart Meter Simulator')

4. Select **Activate** to simulate changing the device status from disabled to enabled in the IoT Hub Registry. The selected windows should turn green.

    ![On the Smart Meter Simulator, the Activate button is highlighted, and all the windows have now turned to green.](media/smart-meter-simulator-activate-all.png 'Fabrikam Smart Meter Simulator')

5. Select **Connect**. Within a few moments, you should begin to see activity and the windows change color, indicating the smart meters are transmitting telemetry. The grid on the left will list each telemetry message transmitted and the simulated temperature value.

    ![On the Smart Meter Simulator, the Connect button is highlighted, the windows have turned different colors to signify the temperature.](media/smart-meter-simulator-connect-all.png 'Fabrikam Smart Meter Simulator')

6. Hover over one of the windows. You will see a dialog display information about the associated device, including the Device ID (in this case, `Device1`), Device Key, Temperature, and Indicator. The legend on the bottom shows the indicator displayed for each temperature range. The Device ID is important when sending cloud-to-device messages, as this is how we will target a specific device when we remotely set the desired temperature. Keep the Device ID values in mind when sending the messages in the next task.

    ![A dialog containing device information is displayed after hovering over a window.](media/smart-meter-simulator-device-info.png 'Fabrikam Smart Meter Simulator')

7. Allow the smart meter to continue to run.

### Task 3: Run the console app and send cloud-to-device messages

In this task, you will run the console app to send desired temperature settings to specific devices and observe the simulated device receiving and reacting to the message.

1. Within the `SmartMeterSimulator` Visual Studio solution, right-click the **CloudToDevice** project, select **Debug**, then select **Start new instance** to run the console app.

2. In the console window, enter a device number when prompted. Accepted values are 0-9, since there are 10 devices whose IDs begin with 0. You can hover over the windows in the Smart Meter Simulator to view the Device IDs. When you enter a number, such as `5`, then a message will be sent to `Device5`.

    ![The value of 1 is entered when prompted for the device number in the console window.](media/console-device-number.png 'Console App')

3. Now enter a temperature value between 65 and 85 degrees (F) when prompted. If you set a value above 72 degrees, the window will turn red. If the value is set between 68 and 72 degrees, it will turn green. Values below 68 degrees will turn the window blue. Once you set a value, the device will remain at that value until you set a new value, rather than randomly changing.

    ![A value of 75 has been entered for the temperature. A new log entry in the Smart Meter Simulator appears in yellow showing the message value of 75 sent to Device1.](media/console-temperature.png 'Console App and Smart Meter Simulator')

    If you run the Smart Meter Simulator side-by-side with the console app, you can observe the message logged by the Smart Meter Simulator within seconds. This message appears with a yellow background and displays the temperature request value sent to the device. In our case, we sent a request of 75 degrees to Device1. The console app indicates that it is sending the temperature request to the indicated device.

4. Hover over the device to which you sent the message. You will see that its temperature is set to the value you requested through the console app.

    ![Device1 is hovered over and the dialog appears showing the temperature set to the requested temperature.](media/smart-meter-simulator-set-temp.png 'Fabrikam Smart Meter Simulator')

5. In the console window, you can enter `Y` to send another message. Experiment with setting the temperature on other devices and observe the results.

## After the hands-on lab

Duration: 10 mins

In this exercise, you will delete any Azure resources that were created in support of the lab. You should follow all steps provided after attending the Hands-on lab to ensure your account does not continue to be charged for lab resources.

### Task 1: Delete the resource group

1. Using the [Azure portal](https://portal.azure.com), navigate to the Resource group you used throughout this hands-on lab by selecting Resource groups in the left menu.

2. Search for the name of your research group and select it from the list.

3. Select Delete in the command bar and confirm the deletion by re-typing the Resource group name, and selecting Delete.

You should follow all steps provided _after_ attending the Hands-on lab.
