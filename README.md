# Arduino_Serial_To_RStudio
This is an update exercise to Saul Buentello's original post in "towards data science"
R and Arduino Serial Interface

Hello, since I’m waiting on a Sparkfun shipment to continue working on a Websockets project, I decided today would be a good day to learn about interfacing R and Arduino, using the Serial connection. This way, I’ll be familiar with translation issues between the platforms, and know how to transform data for each side to interpret and use when I get back into Websockets. 

I started with this tutorial, which was a fairly straightforward walkthrough on how to get a serial connection started. I (re)learned that, at least in my version of R, I have to select the entire section of code to run. That took about 10 minutes. 

Now that I’ve got R running my arduino through serial, it’s time to see if I can edit the code to add functionality. I’m starting with:



I’d like to see if I can get the Arduino to fill in this data with a potentiometer reading into the Serial. There’s other sections of this code where Serial is being read back into tibbles, so it should be a simple copy paste, with some table settings selected so that the serial keeps running until the table is full. 

I added a Potentiometer to the circuit board, and ran an example potentiometer code to make sure the serial was reading correctly. 

I had to add in a Serial.begin line, and Serial.println so that data from the potentiometer would read into serial in a nice column for R to read. 


  - Note: because most Arduinos have a built-in LED attached to pin 13 on the
    board, the LED is optional.

  created by David Cuartielles
  modified 30 Aug 2011
  By Tom Igoe

  This example code is in the public domain.

  https://www.arduino.cc/en/Tutorial/BuiltInExamples/AnalogInput
*/

int sensorPin = A3;    // select the input pin for the potentiometer
int ledPin = 13;      // select the pin for the LED
int sensorValue = 0;  // variable to store the value coming from the sensor

void setup() {
  // declare the ledPin as an OUTPUT:
  Serial.begin(9600);
  pinMode(ledPin, OUTPUT);
}

void loop() {
  // read the value from the sensor:
  sensorValue = analogRead(sensorPin);
  // turn the ledPin on
  digitalWrite(ledPin, HIGH);
  // stop the program for <500> milliseconds:
  delay(200);
  // turn the ledPin off:
  digitalWrite(ledPin, LOW);
  // stop the program for for <500> milliseconds:
  delay(200);
  Serial.println(sensorValue);
}


Next I need to move the core pieces of this Arduino code to main, with a serial port open to make sure the data is caching before RStudio pulls it to read. I want the Arduino to read a specific length of data in three rows, so that R can parse and use the table. I also want to attach correct R,G,B labels to each value to mimic the original inputs. Since I only wanted to produce this data once, I added it to the setup section in the Arduino IDE. 

# include <Servo.h>
// SETTING SERVO AND RGB LEDS 
 
int rLed = 0;
int gLed = 0;
int bLed = 0;
int serv = 0;
int redLed = 6; // RED LED ON PIN 6
int greenLed = 5; // GREEN LED ON PIN 5
int blueLed = 3; // BLUE LED ON PIN 3
int theServo = 9; // SERVO ON PIN 9
int sensorPin = A3;    // select the input pin for the potentiometer
int sensorValue = 0;  // variable to store the value coming from the sensor
String Potentiometer = "";
unsigned int lastStringLength = Potentiometer.length();     // previous length of the String

Servo myServo;


void setup() {
  Serial.begin(9600);  
  
  // SETTING PINS AS OUTPUT
  
  pinMode(redLed, OUTPUT);
  pinMode(greenLed, OUTPUT);
  pinMode(blueLed, OUTPUT);
// ATTACHING SERVO OBJECT
  
  myServo.attach(theServo);

  // add incoming characters to the R String:
  while (Serial.available() > 0, Potentiometer.length() < 120) {
        // read the value from the sensor:
    Potentiometer += analogRead(sensorPin);
    Potentiometer += "R, ";
    delay(100);
    lastStringLength = Potentiometer.length();
  }
  //Print row and clear string
     Serial.println(Potentiometer);
     Potentiometer = "";

       // add incoming characters to the G String:
  while (Serial.available() > 0, Potentiometer.length() < 120) {
        // read the value from the sensor:
    Potentiometer += analogRead(sensorPin);
    Potentiometer += "G, ";
    delay(100);
    lastStringLength = Potentiometer.length();
  }
     Serial.println(Potentiometer);
     Potentiometer = "";

            // add incoming characters to the B String:
  while (Serial.available() > 0, Potentiometer.length() < 120) {
        // read the value from the sensor:
    Potentiometer += analogRead(sensorPin);
    Potentiometer += "B, ";
    delay(100);
    lastStringLength = Potentiometer.length();
  }
     Serial.println(Potentiometer);
     Potentiometer = "";
}

The output looked like this:

0R, 0R, 0R, 0R, 0R, 0R, 124R, 321R, 333R, 332R, 302R, 225R, 190R, 186R, 186R, 186R, 182R, 99R, 0R, 0R, 29R, 221R, 339R, 
400G, 400G, 401G, 427G, 499G, 531G, 562G, 572G, 558G, 461G, 281G, 138G, 3G, 0G, 0G, 145G, 221G, 203G, 16G, 38G, 254G, 408G, 
426B, 256B, 148B, 300B, 315B, 245B, 409B, 486B, 486B, 278B, 0B, 0B, 132B, 234B, 136B, 22B, 144B, 373B, 467B, 463B, 152B, 


Now it’s time to see if R can grab this data and use it in a table for controlling the LED’s. Running the R code without modification, I did find a table in the R console that matched the Arduino, with parsing errors after that…so…success! Now it’s time to let the R code know that the Arduino has its own set of data to bounce back to the LED’s. 

At first I tried just copying the tibble read call from further down, and switching “arduinoInput” to “dataFromArduino” in the next few steps, so that it would deposit the reading in the correct location, but it seems that the tibble doesn’t autodetect separate entries, so I’m going to have to use a parse function to get the values separated. I tried using str_split after capturing the tibble from the Serial, to mixed success. It did separate the observations, but it did not write back to a tibble. I might need to go back and look at that again, and see if splitting the string back into a new tibble would work.  Next, I looked into the cat function in

dataFromArduino <- tibble(
  capture.output(cat(read.serialConnection(myArduino,n=0), ", ")))

And found that including a value for the “sep” call in “cat” got the table to manage writing to three separate variable columns, but only one observation per variable. 
Next I need to separate the observations, and transpose the table. I tried:

ArduinoPot <- tibble(
  capture.output(cat(read.serialConnection(myArduino,n=0), ", ")))
colnames(ArduinoPot)[1]<-"column"
arduinoInput<-separate(ArduinoPot, col = "column" , into = c("1","2","3","4","5","6","7","8","9","10","11","12","13","14","15","16","17","18","19","20","21","22","23","24","25","26","27","28","29","30"), sep = ", ", remove = TRUE)
transposedInput<-transpose(arduinoInput)

Which almost did it, but the transposedInput table didn’t handle using row names as variable names. So instead, I changed the tibble into a matrix to transpose, and then turned it back into a tibble for further wrangling. Let’s see how that went. 

#WRANGLING THE DATA
colnames(ArduinoPot)[1]<-"column"
PotInput<-separate(ArduinoPot, col = "column" , into = c("1","2","3","4","5","6","7","8","9","10","11","12","13","14","15","16","17","18","19","20","21","22","23","24","25","26","27","28","29","30"), sep = ", ", remove = TRUE, convert = FALSE, extra = "warn", fill = "right")
PotInput<-as.matrix(PotInput[-c(27,28,29,30)])
transposedInput<-tibble(t(PotInput))
colnames(transposedInput)[1]<-"blank"
transposedInput %>% fill("blank")
transposedInput<-separate(transposedInput, col = "blank", into = c("blank","r","g","b"), sep = c())
transposedInput<-select(transposedInput, c("blank[,2]","blank[,3]","blank[,4]"))
dim(transposedInput)

And the transpose worked, as well as filling in the missing values with dummy values. The dummy values are needed because dim(transposedInput) shows this is now just one column, even though the table shows like:



So I went back and changed transposedInput<-tibble(t(PotInput), NULL, "unique")  to
transposedInput<-as_tibble(t(PotInput), NULL, "unique")
And that seemed to fix the column-disappearing problem. Now I just need to relabel the columns to match, and trim the tibble to include only the valid columns. 

colnames(transposedInput)[3]<-"r"
colnames(transposedInput)[4]<-"g"
colnames(transposedInput)[5]<-"b"
transposedInput %>% fill("blank")
transposedInput<-select(transposedInput, c("r","g","b"))
dim(transposedInput)

# A GLIMPSE OF ARDUINO INPUT
glimpse(transposedInput)
glimpse(ArduinoPot)

A quick glimpse and the table now looks identical to the original. Let’s see if it feeds correctly. 
The lights light up! That means that I’ve imported the correct data, formatted it so that R could understand it, and then send it back to the Arduino in a format that the Arduino could execute. Success!

Running the rest of the lines of data, it seems that I’ve trimmed the columns too closely to be combined with the old version of the data, so now I have a choice to make; do I use the residual table from the original version, or work back through the rest of the code to redirect the calls to the correct tables? Let’s use the new ones for internal consistency. 

When we use external data, it’s likely that the number of rows will vary, or that cleaning the data will cause some blanks. The original code used cbind to merge the two tables, but that won’t work with variable height tibbles. I’ll change it to merge so it fills in with NA’s, so that the Arduino can interpret the blank data. 

So, trying it with the full run-through exposed that the NA’s are a problem, especially when there’s pasting across the entire table going on, and some other formatting for Arduino to read. So, I went through and added replace_na to several places to make sure the program was running mainly integers throughout aside from the pasting, and now at another full run through, I’m running into a problem where the entire table has a problem computing a mutate. 

Checking through the tables line by line (which was a very good move to make new tables at each step instead of rewriting the same table every step for process history!) it turns out that the arduino Serial was buffering and concatenating old data, and needed to be flushed. So, hitting the reset button on the Arduino, and running from the very beginning of the R code, it worked! It ran all the way through to the ggplot, and here’s how it looks!



And that, ladies and gents, is probably not the right way to do all that…but hey, not bad for my first time back in R Studio since COVID shutdowns…

Thanks!

Oh, and here’s all the code for both:

library(tidyverse)
library(magrittr)
library(plotly)
library(serial)
library(ggplot2)
library(dplyr)

listPorts()

# SERIAL CONNECTION
myArduino <-  serialConnection(
  port = "COM3",
  mode = "9600,n,8,1" ,
  buffering = "none",
  newline = TRUE,
  eof = "",
  translation = "cr",
  handshake = "none",
  buffersize = 4096
)

# OPEN AND TESTING THE CONNECTION
open(myArduino)
isOpen(myArduino)

# GIVING TIME FOR THE BOARD TO COLLECT POTENTIOMETER DATA ONCE THE SERIAL INTERFACE IS INITIATED
Sys.sleep(15)

# CAPTURING RGB DATA FROM POTENTIOMETER
ArduinoPot <- tibble(
  capture.output(cat(read.serialConnection(myArduino,n=0))))

#WRANGLING THE DATA
colnames(ArduinoPot)[1]<-"column"
PotInput<-separate(ArduinoPot, col = "column" , into = c("1","2","3","4","5","6","7","8","9","10","11","12","13","14","15","16","17","18","19","20","21","22","23","24","25","26","27","28","29","30"), sep = ", ", remove = TRUE, convert = FALSE, extra = "warn", fill = "right")
PotInput<-as.matrix(PotInput)
transposedInput<-as_tibble(t(PotInput), NULL, "unique")
colnames(transposedInput)[2]<-"r"
colnames(transposedInput)[3]<-"g"
colnames(transposedInput)[4]<-"b"
transposedInput <- transposedInput %>% replace_na(list(r = "0R", g = "0G", b = "0B"))
transposedInput %>% fill("V1")
transposedInput<-select(transposedInput, c("r","g","b"))
dim(transposedInput)

# A GLIMPSE OF ARDUINO INPUT
glimpse(transposedInput)
glimpse(ArduinoPot)

# CLOSE THE OPEN CONNECTION AGAIN (BEST PRACTICE)
close(myArduino)
open(myArduino)

# GIVING TIME FOR THE BOARD TO RESET ONCE THE SERIAL INTERFACE IS INITIATED
Sys.sleep(3)
for (r in seq_len(n)){
  Sys.sleep(0.25)
  write.serialConnection(myArduino, paste(transposedInput[r,], collapse = ''))
}

# READ MAPPED DATA SENT FROM MY ARDUINO
dataFromArduino <- tibble(
  capture.output(cat(read.serialConnection(myArduino,n=0)))
)

# SELECT FIRST NINE ROWS, ASSIGN VALUES TO THEIR LEDS AND RENAME FIRST COLUMN
dataFromArduino %>% 
  slice_head(n = 9)
dim(dataFromArduino)

# ASSIGN VALUES TO LEDS AND CHANGE COLUMN NAME
dataFromArduino %<>% 
  tibble(ledNames = rep_along(seq_len(nrow(dataFromArduino)), 
                              c('rMapped','gMapped','bMapped'))) %>%
  rename("ledVal" = 1) %>%
  group_by(ledNames) %>%
  
  # ADD IDENTIFIERS REQUIRED BY PIVOT_WIDER FUNCTION AND CREATE NEW COLUMNS WITH 'LEDVAL' VALUES
  
  mutate(row = row_number()) %>%
  pivot_wider(names_from = ledNames, values_from = ledVal) %>%
  
  # DROPPING 'ROW' COLUMN AND CONVERT ALL COLUMNS TO DATA TYPE INTEGER 
  
  select(-row) %>%
  mutate_all(as.integer)
dataFromArduino %>% fill(c("rMapped","gMapped","bMapped"), )
dataFromArduino %>% 
  slice_head(n = 10)
dim(dataFromArduino)
dim(transposedInput)

# MERGE THE TWO DATA SETS,  DROP NON NUMERICAL CHARACTERS (R,G,B), AND REORDER COLUMNS
combinedData <- as_tibble(merge(transposedInput, dataFromArduino)) %>%
  mutate(across(where(is.character), ~parse_number(.x)), across(where(is.double), as.integer)) %>% 
  select(c(1, 4, 2, 5, 3, 6))
combinedData <- combinedData %>% replace_na(list(r = 0, rMapped = 0, g = 0, gMapped = 0, b = 0, bMapped = 0))
combinedData %>%
  slice_head(n = 10)

# CREATING NEW DATA SET THAT SELECTS VALUES IN ORDER: MAXIMUM OF RECEIVED LED VALUES, THEN MINIMUM, AND SO IS REPEATED
rowMin <- tibble(inputMin = dataFromArduino %>% apply(1,min)) %>%
  
  # SELECT EVEN ROWS
  
  filter(row_number() %% 2 == 0)
servoInput <- tibble(servoIn = dataFromArduino %>% 
                       apply(1,max))
servoInput <- servoInput %>% replace_na(list(servoIn = 10))

# REPLACE EVEN ROWS WITH A MINIMUM VALUE, AND APPENDING A TERMINATING CHARACTER
servoInput[c(1:n)[c(F,T)],] <- rowMin
servoInput %<>% 
  mutate(servoIn = servoIn %>% 
           paste('S', sep = ''))

close(myArduino)
open(myArduino)
Sys.sleep(1)
for (r in seq_len(n)){
  Sys.sleep(1)
  write.serialConnection(myArduino, paste(servoInput[r,], collapse = ''))
}

# READ MAPPED ANGLES SENT FROM MY ARDUINO, RENAME FIRST COLUMN
angleFromServo <- tibble(
  capture.output(cat(read.serialConnection(myArduino,n=0)))) %>%
  rename("servoAnglesMapped" = 1) %>% 
  mutate_all(as.integer)
angleFromServo <- angleFromServo %>% replace_na(list(servoAnglesMapped = 0))

# SELECT FIRST TEN ROWS
angleFromServo %>% 
  slice_head(n = 10)


# WHAT WE SENT VS WHAT WE RECEIVED. MERGE THE TWO DATA SETS AND DROP NON NUMERIC CHARACTER 'S'
combinedAngles <- as_tibble(
  merge(servoInput, angleFromServo)) %>%
  mutate(across(where(is.character), ~parse_number(.x)),
         across(where(is.double), as.integer))
combinedAngles %>%
  slice_head(n = 10)

# PLOT VARIATION OF SERVO ANGLE
theme_set(theme_light())
myPlot <- angleFromServo %>%
  ggplot(mapping = aes(x = 1:nrow(angleFromServo), y = servoAnglesMapped)) +
  geom_line() +
  geom_smooth(se = F) +
  labs(x = "Count", y = "Servo angle",  title = "Servo angle variation at each count instance")+
  theme(plot.title = element_text(hjust = 0.5))
ggplotly(myPlot)



Now here’s the Arduino code:


# include <Servo.h>
// SETTING SERVO AND RGB LEDS 
 
int rLed = 0;
int gLed = 0;
int bLed = 0;
int serv = 0;
int redLed = 6; // RED LED ON PIN 6
int greenLed = 5; // GREEN LED ON PIN 5
int blueLed = 3; // BLUE LED ON PIN 3
int theServo = 9; // SERVO ON PIN 9
int sensorPin = A3;    // select the input pin for the potentiometer
int sensorValue = 0;  // variable to store the value coming from the sensor
String Potentiometer = "";
unsigned int lastStringLength = Potentiometer.length();     // previous length of the String

Servo myServo;


void setup() {
  Serial.begin(9600);  
  
  // SETTING PINS AS OUTPUT
  
  pinMode(redLed, OUTPUT);
  pinMode(greenLed, OUTPUT);
  pinMode(blueLed, OUTPUT);
// ATTACHING SERVO OBJECT
  
  myServo.attach(theServo);

  // add incoming characters to the R String:
  while (Serial.available() > 0, Potentiometer.length() < 120) {
        // read the value from the sensor:
    Potentiometer += analogRead(sensorPin);
    Potentiometer += "R, ";
    delay(100);
    lastStringLength = Potentiometer.length();
  }
  //Print row and clear string
   // Serial.print("Red, ");
     Serial.println(Potentiometer);
     Potentiometer = "";

       // add incoming characters to the G String:
  while (Serial.available() > 0, Potentiometer.length() < 120) {
        // read the value from the sensor:
    Potentiometer += analogRead(sensorPin);
    Potentiometer += "G, ";
    delay(100);
    lastStringLength = Potentiometer.length();
  }
   //  Serial.print("Green, ");
     Serial.println(Potentiometer);
     Potentiometer = "";

            // add incoming characters to the B String:
  while (Serial.available() > 0, Potentiometer.length() < 120) {
        // read the value from the sensor:
    Potentiometer += analogRead(sensorPin);
    Potentiometer += "B, ";
    delay(100);
    lastStringLength = Potentiometer.length();
  }
   //  Serial.print("Blue, ");
     Serial.println(Potentiometer);
     Potentiometer = "";
}


void loop() {
  if (Serial.available()){
    
   // MAKING VARIABLE VISIBLE TO ONLY 1 FUNCTION
   // CALL AND PRESERVE THEIR VALUE
   
   static int t = 0;
   
   char myvalue = Serial.read();
switch(myvalue){
    
    // MYVALUE IS A a VARIABLE WHOSE VALUE TO COMPARE WITH VARIOUS CASES
    
    case '0'...'9':
      t = t * 10 + myvalue - '0';
      break;   
        
    case 'R':
    {
      rLed = map(t, 0, 100, 0, 255);
      analogWrite(redLed, rLed);
      Serial.println(rLed);  
    }
    t = 0;
    break;
    
    case 'G':
    {
      gLed = map(t, 0, 100, 0, 255);
      analogWrite(greenLed, gLed);
      Serial.println(gLed);
    }
    t = 0;
    break;
case 'B':
    {
      bLed = map(t, 0, 100, 0, 255);
      analogWrite(blueLed, bLed);
      Serial.println(bLed);
    }
    t = 0;
    break;
case 'S':
    {
      
      // MAPPING ANALOGUE LED VALUE TO ANGLE FROM 0 TO 180 DEGREES
      
      serv = map(t, 0, 26, 0, 179);
      Serial.println(serv);
      delay(5);
      myServo.write (serv);
      delay(150);
    }
    t = 0;
    break;
   }
  }
}

