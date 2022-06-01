# SENSOR-COMMUNICATION-USING-PAINLESS-MESH-

Exchange Sensor Readings using ESP-MESH

Parts Required
Here’s the parts required for this example:

4x ESP boards (ESP32 or ESP8266)
4x MPU6050
Breadboard
Jumper wires

THE BELOW IMAGE IS DESIGNED WITH AN EXAMPLE OF BME 280 SENSOR BUT HERE IN THIS ARTICLE IAM EXPLAINING WITH MPU 650 AND THE BLOCK DIAGRAM FOR IT IS ALSO ATTACHED BELOW 
![image](https://user-images.githubusercontent.com/93335682/171458175-93b872b9-c466-4bdc-ac89-9b5d9935ee10.png)





BLOCK DIAGRAM OF INDIVIDUAL NODE 
![image](https://user-images.githubusercontent.com/93335682/171457296-b6cc8e99-2066-470a-93ed-9129f2e6bd66.png)



How the Code Works


Continue reading this section to learn how the code works.

Libraries
Start by including the required libraries: the Adafruit_Sensor and Adafruit_MPU6050 to interface with the BME280 sensor; the painlessMesh library to handle the mesh network and the Arduino_JSON to create and handle JSON strings easily.

#include "painlessMesh.h"
#include <Arduino_JSON.h>

#include "painlessMesh.h"
//#include <Arduino_JSON.h>

#include <Adafruit_Sensor.h>
#include <Adafruit_MPU6050.h>
#include <Wire.h>

Mesh details
Insert the mesh details in the following lines.

#define MESH_PREFIX    "SAIKISHORE" //name for your MESH
#define MESH_PASSWORD  "CITYOFKITCHENER" //password for your MESH
#define MESH_PORT      5555 //default port
The MESH_PREFIX refers to the name of the mesh. You can change it to whatever you like. The MESH_PASSWORD, as the name suggests is the mesh password. You can change it to whatever you like. All nodes in the mesh should use the same MESH_PREFIX and MESH_PASSWORD.

The MESH_PORT refers to the the TCP port that you want the mesh server to run on. The default is 5555.

mpu6050
Create an Adafruit_mpu6050object called bme on the default ESP32 or ESP8266 pins.



Adafruit_MPU6050 mpu;
In the nodeNumber variable insert the node number for your board. It must be a different number for each board.

int nodeNumber = 2;
The readings variable will be used to save the readings to be sent to the other boards.

String readings;
Scheduler
The following line creates a new Scheduler called userScheduler.

Scheduler userScheduler; // to control your personal task
painlessMesh
Create a painlessMesh object called mesh to handle the mesh network.

Create tasks
Create a task called taskSendMessage responsible for calling the sendMessage() function every five seconds as long as the program is running.

Task taskSendMessage(TASK_SECOND * 5 , TASK_FOREVER, &sendMessage);
getReadings()
The getReadings() function gets temperature, humidity and pressure readings from the BME280 sensor and concatenates all the information, including the node number on a JSON variable called jsonReadings.

 JSONVar jsonReadings;
  /* Get new sensor events with the readings */
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);

  jsonReadings["node"] = nodeNumber;
  jsonReadings["XAXIS"] = String(a.acceleration.x);
  jsonReadings["YAXIS"] = String(a.acceleration.y);
  jsonReadings["ZAXIS"] = String(a.acceleration.z);
  readings = JSON.stringify(jsonReadings);
  return readings;




The jsonReadings variable is then converted into a JSON string using the stringify() method and saved on the readings variable.

readings = JSON.stringify(jsonReadings);
This variable is then returned by the function.

return readings;
Send a Message to the Mesh
The sendMessage() function sends the JSON string with the readings and node number (getReadings()) to all nodes in the network (broadcast).

void sendMessage () {
  String msg = getReadings();
  mesh.sendBroadcast(msg);
}
Init mpu6050 sensor
The initmpu() function initializes the MPU6050 sensor.

//Init MPU6050
void initmpu(){
   if (!mpu.begin()) {
    Serial.println("Failed to find MPU6050 chip");
    while (1) {
      delay(10);
    }
  }
  Serial.println("MPU6050 Found!");

  mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
  Serial.print("Accelerometer range set to: ");
  switch (mpu.getAccelerometerRange()) {
  case MPU6050_RANGE_2_G:
    Serial.println("+-2G");
    break;
  case MPU6050_RANGE_4_G:
    Serial.println("+-4G");
    break;
  case MPU6050_RANGE_8_G:
    Serial.println("+-8G");
    break;
  case MPU6050_RANGE_16_G:
    Serial.println("+-16G");
    break;
  }
   Serial.println("");
  delay(100);
}

Mesh Callback Functions
Next, several callback functions are created that will be called when some event on the mesh happens.

The receivedCallback() function prints the message sender (from) and the content of the message (msg.c_str()).

void receivedCallback( uint32_t from, String &msg ) {
  Serial.printf("startHere: Received from %u msg=%s\n", from, msg.c_str());


The message comes in JSON format, so, we can access the variables as follows:

void receivedCallback( uint32_t from, String &msg ) {
  Serial.printf("Received from %u msg=%s\n", from, msg.c_str());
  JSONVar myObject = JSON.parse(msg.c_str());
  int node = myObject["node"];
  double XAXIS  = myObject["XAXIS"];
  double YAXIS = myObject["YAXIS"];
  double ZAXIS = myObject["ZAXIS"];
  Serial.print("Node: ");
  Serial.println(node);

The newConnectionCallback() function runs whenever a new node joins the network. This function simply prints the chip ID of the new node. You can modify the function to do any other task.

void newConnectionCallback(uint32_t nodeId) {
  Serial.printf("--> startHere: New Connection, nodeId = %u\n", nodeId);
}
The changedConnectionCallback() function runs whenever a connection changes on the network (when a node joins or leaves the network).

void changedConnectionCallback() {
  Serial.printf("Changed connections\n");
}
The nodeTimeAdjustedCallback() function runs when the network adjusts the time, so that all nodes are synchronized. It prints the offset.

void nodeTimeAdjustedCallback(int32_t offset) {
  Serial.printf("Adjusted time %u. Offset = %d\n", mesh.getNodeTime(),offset);
}


setup()
In the setup(), initialize the serial monitor.

void setup() {
  Serial.begin(115200);
Call the initBME() function to initialize the BME280 sensor.

initmpu();
Choose the desired debug message types:

//mesh.setDebugMsgTypes( ERROR | MESH_STATUS | CONNECTION | SYNC | COMMUNICATION | GENERAL | MSG_TYPES | REMOTE ); // all types on

mesh.setDebugMsgTypes( ERROR | STARTUP );  // set before init() so that you can see startup messages
Initialize the mesh with the details defined earlier.

mesh.init(MESH_PREFIX, MESH_PASSWORD, &userScheduler, MESH_PORT);
Assign all the callback functions to their corresponding events.

mesh.onReceive(&receivedCallback);
mesh.onNewConnection(&newConnectionCallback);
mesh.onChangedConnections(&changedConnectionCallback);
mesh.onNodeTimeAdjusted(&nodeTimeAdjustedCallback);
Finally, add the taskSendMessage function to the userScheduler. The scheduler is responsible for handling and running the tasks at the right time.

userScheduler.addTask(taskSendMessage);
Finally, enable the taskSendMessage, so that the program starts sending the messages to the mesh.

taskSendMessage.enable();
To keep the mesh running, add mesh.update() to the loop().



void loop() {
  // it will run the user scheduler as well
  mesh.update();
}
Demonstration
After uploading the code to all your boards (each board with a different node number), you should see that each board is receiving the other boards’ messages.

The following screenshot shows the messages received by node 1. It receives the sensor readings from node 2, 3 and 4.



ouput image 
![image](https://user-images.githubusercontent.com/93335682/171459518-c9064ab9-d2db-41d1-a9a3-ac516e6d63d4.png)



circuit connection image 

![image](https://user-images.githubusercontent.com/93335682/171459721-d6ee2eaa-d717-4d5c-bf77-97d233cf5f2a.png)


