# Toaster Oven Example
This example is supposed so showcase multiple capabilites of the plcnext system. The overarching goal is to control the temperature of an oven with a heater and fan, while reading the current temperature with a thermocouple. The fan and heater are both conrolled with industrial relays and the k-type thermocouple is connected through a transducer.

## Wiring Diagram

![image](../assets/ToasterOven-1.svg)

## Actual Setup

![image](../assets/ToasterOven-2.jpg)
![image](../assets/ToasterOven-3.jpg)
![image](../assets/ToasterOven-4.jpg)
![image](../assets/ToasterOven-5.jpg)
![image](../assets/ToasterOven-6.jpg)

## Ladder Logic Control
The following code converts the slider input of the starterkit into a temperature input and turns the system on when button s3 is pressed and turned off when button s2 is pressed. The heater is turned on when the set temperature is higher than the current temperature (plus a buffer). The fan is turned on when the set temperature is less than the current temperature (minus a buffer).

To run this example open the **ToasterOvenLadderLogic.pcwex** project and upload it. Enter debug mode to observe the controller logic.

Ladder Logic Code:

![image](../assets/ToasterOven-7.png)

Variables:

![image](../assets/ToasterOven-8.png)

## CPP & Rest Control
This example showcases two things: First, it implements the exact same logic as the ladder logic example, with a real-time cyclic cpp program. Secondly, it has the ability to overwrite the heater control using the button and slider on the hardware with a rest interface, where one can control the temperature of the oven using an example website provided.

More information on how this was implemented can be found in the [SimpleIOReadWrite Project](../SimpleIOReadWrite/README.md) and the [RestCommunication Project](../RestCommunication/README.md)

To run this example open the **ToasterOvenCppRest.pcwex** Project and upload it. Then use the slider and buttons on the starterkit to control it or use the website as descibed below.

### Cpp Code

Logic:
```cpp
void ToasterOvenCppProgram::Execute()
{
	currentTemp = thermoInput * 0.058 - 100;

	if (restControl) {
		// control using rest
		on = true;
		setTemp = restSetTemp;

	} else {
		// control using hardware
		setTemp = sliderInput * 0.02;
		if (onButton || on) {
			on = true;
		}
		if (offButton) {
			on = false;
		}
	}

	if ((currentTemp + heatingBuffer) < setTemp) {
		heating = true;
	} else {
		heating = false;
	}
	if ((currentTemp - coolingBuffer) > setTemp) {
		cooling = true;
	} else {
		cooling = false;
	}

	if (heating && on) {
		heaterOutput = true;
		fanOutput = false;
	} else if (cooling && on) {
		heaterOutput = false;
		fanOutput = true;
	} else {
		heaterOutput = false;
		fanOutput = false;
	}

	log.Info("on: {0} heating: {1} cooling: {2} currentTemp: {3} setTemp: {4} REST: {5}", on, heating, cooling, currentTemp, setTemp, restControl);
}
```

Variables:
```cpp
public:
    //#port
    //#attributes(Input)
    uint16 sliderInput;

    //#port
    //#attributes(Input)
    uint16 thermoInput;

    //#port
    //#attributes(Input)
    bool onButton;

    //#port
    //#attributes(Input)
    bool offButton;

    //#port
    //#attributes(Output)
    bool heaterOutput; // Output pin array (16 output pins)

    //#port
    //#attributes(Output)
    bool fanOutput; // Output pin array (16 output pins)

    //#port
    //#attributes(Input|Ehmi)
    float32 restSetTemp= 0;

    //#port
    //#attributes(Input|Ehmi)
    bool restControl = false;

    //#port
    //#attributes(Output|Ehmi)
    bool on = false;
    //#port
    //#attributes(Output|Ehmi)
    bool heating = false;
    //#port
    //#attributes(Output|Ehmi)
    bool cooling = false;

    //#port
    //#attributes(Output|Ehmi)
    float32 currentTemp;

    //#port
    //#attributes(Output|Ehmi)
    float32 setTemp;

private: // fields
    ToasterOvenCpp::ToasterOvenCppComponent& toasterOvenCppComponent;

    double heatingBuffer = 5.0;
    double coolingBuffer = 5.0;

};
```
### PLCNext Engineer Configuration

GDS Port Configuration:

![image](../assets/ToasterOven-9.png)

Cyclic Task Configuration:

![image](../assets/ToasterOven-10.png)

### Website
Follow the instructions from the [RestCommunication Example](../RestCommunication/README.md) on using the CORS extension.

If used correctly, opening the index.html file in the website folder should look like this:

![image](../assets/ToasterOven-12.png)

## Remote GRPC Control
This does exactly the same as the ladder logic example, but all computations are done on a remote device using python through grpc communication (see GrpcRemoteCommunication for more information [here](../GrpcRemoteCommunication/README.md))

To run this example open the **ToasterOvenGrpc.pcwex** project and upload it. On a remote device with python and linux follow the following steps.

1. Install Dependencies (follow steps [here](../GrpcRemoteCommunication/README.md#installing-dependencies))

2. Compile protobuf (run within the python folder)

```sh
bash ./build_all.sh
```

4. Configure Certificate (follow steps [here](../GrpcRemoteCommunication/README.md#configuring-the-certificate))

5. Get certificate from controller (follow steps [here](../GrpcRemoteCommunication/README.md#downloading-the-certificate))

6. Run the code

```sh
python3 example.py
```

Example Output:

![image](../assets/ToasterOven-11.png)

### Code:

```python
import grpc
import sys
import os
import time
import matplotlib.pyplot as plt
import numpy as np

# Get the absolute path to the directory containing the module and add it to the python path
module_dir = os.path.abspath(os.path.join(os.path.dirname(__file__), 'ServiceStubs'))
sys.path.insert(0, module_dir)

## Import Protobuf librarys
from ServiceStubs.Plc.Gds.IDataAccessService_pb2_grpc import IDataAccessServiceStub
from ServiceStubs.Plc.Gds.IDataAccessService_pb2 import IDataAccessServiceReadRequest
# GDS Write Multiple
from ServiceStubs.Plc.Gds.IDataAccessService_pb2 import IDataAccessServiceWriteRequest
from ServiceStubs.Plc.Gds.WriteItem_pb2 import WriteItem

# function to read input from controller
def readInput(stub):
    item = IDataAccessServiceReadRequest()
    item.portNames.extend(['Arp.Plc.Eclr/sliderInput', 'Arp.Plc.Eclr/thermoInput', 'Arp.Plc.Eclr/onButton', 'Arp.Plc.Eclr/offButton'])
    response = stub.Read(item)
    sliderInput = response._ReturnValue[0].Value.Uint16Value
    thermoInput = response._ReturnValue[1].Value.Uint16Value
    onButton = response._ReturnValue[2].Value.BoolValue
    offButton = response._ReturnValue[3].Value.BoolValue
    return [sliderInput, thermoInput, onButton, offButton]

# function to write relays on controller 
def writeOutput(stub, heater, fan):
    request = IDataAccessServiceWriteRequest()
    item1 = WriteItem()
    item1.PortName = 'Arp.Plc.Eclr/heaterOutput'
    item1.Value.BoolValue = heater
    item1.Value.TypeCode = 2
    item2 = WriteItem()
    item2.PortName = 'Arp.Plc.Eclr/fanOutput'
    item2.Value.BoolValue = fan
    item2.Value.TypeCode = 2
    request.data.extend([item1, item2])
    return stub.Write(request)

###### Setup Graph
# Initialize the data containers
x_data = []
y1_data = []
y2_data = []
start_time = time.perf_counter()

# Set up the plot
plt.ion()
fig, ax = plt.subplots()
line1, = ax.plot([], [], 'r-', label='Set')
line2, = ax.plot([], [], 'g-', label='Current')

ax.legend()
plt.xlabel('Time')
plt.ylabel('Temperature')
ax.set_ylim(20, 200)

###### Create GRPC Connection
# read certificate
with open('certificate.pem', 'rb') as f:
    root_certificates = f.read()

# open channel and check for health
addr = "192.168.1.10:50051"
cred = grpc.ssl_channel_credentials(root_certificates=root_certificates)
channel = grpc.secure_channel(addr, cred)
try:
    grpc.channel_ready_future(channel).result(timeout=5)
except grpc.FutureTimeoutError:
    sys.exit('Error connecting to server')

# Create IDataAccessServiceStub
stub = IDataAccessServiceStub(channel)

###### Main Control
# turn off everything
writeOutput(stub, False, False)
# variables
on = False
heating = False
cooling = False
heatingBuffer = 5.0
coolingBuffer = 5.0

while (1):
    # get input from controller
    sliderInput, thermoInput, onButton, offButton = readInput(stub)

    # calculate temperatures
    currentTemp = np.round(thermoInput * 0.058 - 100, 1)
    setTemp = np.round((sliderInput / 10000) * 200, 1)

    # turn system on or off
    if (onButton and not offButton) or (on):
        on = True
    if offButton:
        on = False

    # decide wheter to heat or cool oven
    if (currentTemp + heatingBuffer) < setTemp:
        heating = True
    else:
        heating = False

    if (currentTemp - coolingBuffer) > setTemp:
        cooling = True
    else:
        cooling = False

    # turn on heater or fan
    if heating and on:
        mode = "Heating"
        writeOutput(stub, True, False)
    elif cooling and on:
        mode = "Cooling"
        writeOutput(stub, False, True)
    else:
        mode = "Idle"
        writeOutput(stub, False, False)
    if not on:
        mode = "Off"
        writeOutput(stub, False, False)

    # print status
    print(f"Mode: {mode} SetTemp: {setTemp} currentTemp: {currentTemp}")

    # graph stuff
    end_time = time.perf_counter()

    # Append new data
    x_data.append(end_time - start_time)
    y1_data.append(setTemp)
    y2_data.append(currentTemp)

    # Update the data in the line objects
    line1.set_xdata(x_data)
    line1.set_ydata(y1_data)
    line2.set_xdata(x_data)
    line2.set_ydata(y2_data)
    
    # Adjust limits if data goes out of view
    ax.relim()
    ax.autoscale_view()
    
    # Draw and flush events
    fig.canvas.draw()
    fig.canvas.flush_events()

    time.sleep(0.05) 

```
