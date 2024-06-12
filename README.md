The best way to connect the RS485 is the 4 pin JST SM connector under the seat that connects the internals and the external battery plug. There, the two middle connectors are the RS485 interface.

### JST Connector pinning
Super Soco (at least TC Max), uses a JST SM 4 pin connector under the seat. This connector can be disconnected if the motorcycle is not charged through the external charger port. Instead of the connection to the external charger, a JST SM 4 pin plug can be connected and attached to a RS485 converter. The pinning of the connector is as following:

| Pin | Signal  |
| --- | :-----: |
| 1   |    ?    |
| 2   | RX- / B |
| 3   | RX+ / A |
| 4   |    ?    |

The signals on pin 1 and 4 are currently unknown.

# RS485 Protocol
The following information is taken from the [Dashboard Android App](https://github.com/Xmanu12/SuSoDevs) project and is sumarized here. All the information has been reverse engineered and can therefor hold errors and unknown data.

The communication is using **9600 Baud**, with 8 bit data and 1 stop bit. It is based on Read requests and Read responses and data is transmitted in telegrams.
## Generic information and structure

Each telegram starts with two bytes specifying a request or a response, followed by one byte source (id) and one byte destination (id). After that, the length of the user data in Bytes and the data itself is transmitted. Lastly, a one byte checksum and the end tag 0x0D terminates the telegram.

| Byte |   0   |   1   |   2    |   3   |    4    |  4 + 1  |  4 + 2  |  ...  |   4 + n   | 4 + n + 1 | 4 + n + 2 |
| :--- | :---: | :---: | :----: | :---: | :-----: | :-----: | :-----: | :---: | :-------: | :-------: | :-------: |
|      | type1 | type2 | source | dest  | len (n) | data[0] | data[1] |  ...  | data[n-1] | checksum  |   0x0D    |

There are two known combinations for the type:
| Byte1 | Byte2 | Type          |
| ----: | :---- | ------------- |
|  0xC5 | 0x5C  | Read request  |
|  0xB6 | 0x6B  | Read response |

## Checksum calculation
The checksum byte is calculated by making an XOR calculation of the databytes and the length byte. The following C# example shall deepen the understanding how the checksum is calculated.

```c#
// dataLen holds the value of the length byte
// rawData is a byte[] holding the raw data between the length byte and the checksum byte

byte calcCheck = dataLen;
foreach (byte b in rawData)
{
    calcCheck ^= b;
}

if (calcCheck == checksum)
{
    // Data is valid
}

```

## Decoded telegrams
The following telegrams and packages of read responses are already decoded.

### BMS Status (Read Response 0xAA5A)

| Byte (len=10) |    0    |   1   |   2   |   3    |   4    |   5    |   6   |   7   |    8     |    9     |
| ------------- | :-----: | :---: | :---: | :----: | :----: | :----: | :---: | :---: | :------: | :------: |
|               | Voltage |  SoC  | Temp  | Charge | CycleH | CycleL |   ?   |   ?   | VBreaker | Charging |

#### Description of the variables
| Variable   | Description                                  | Unit                                                                                              | Data Type     |
| ---------- | -------------------------------------------- | ------------------------------------------------------------------------------------------------- | ------------- |
| Voltage    | current Voltage of the Battery in V          | Volts[V]                                                                                          | unsigned byte |
| SoC        | State of Charge in %                         | Percent [%]                                                                                       | unsigned byte |
| Temp       | current temperatur of the BMS                | Degree C [°C]                                                                                     | signed byte   |
| Charge     | current charging or discharging current in A | Ampere [A]                                                                                        | signed byte   |
| Cycle[H/L] | Number of loading cycles                     |                                                                                                   | unsigned word |
| VBreaker   |                                              | 0 = OK<br>1 = bms stopped charge<br>2 = too high charge current<br>4 = too high discharge current | unsigned byte |
| Charging   | Battery is currently charging                | 1 = charge<br>4 = discharge                                                                       | unsigned byte |

### ECU Status (Read Response 0xAADA)
| Byte (len=10) |   0   |    1     |    2     |   3    |   4    |    5     |   6   |   7   |    8    |   9   |
| ------------- | :---: | :------: | :------: | :----: | :----: | :------: | :---: | :---: | :-----: | :---: |
|               | Mode  | CurrentH | CurrentL | SpeedH | SpeedL | ECU Temp |   ?   |   ?   | Parking |   ?   |


#### Description of the variables
| Variable      | Description         | Unit                | Data Type     |
| ------------- | ------------------- | ------------------- | ------------- |
| Mode          | Speed Mode          | 1 - 3               | unsigned byte |
| Current [H/L] | Current consumption | [mA] ?              | unsigned word |
| Speed [H/L]   | Current speed       | [km/h] ?            | unsigned word |
| ECU Temp      | Temperature of ECU  | Degree Celcius [°C] | signed byte   |
| Parking       | Parking mode        | 2 = on<br>1 = off   | unsigned byte |


### GSM (Read Request 0xBAAA)
| Byte (len=14) |   0   |   1   |   2   |   3   |   4   |   5    |   6   |   7   |   8   |   9   |  10   |  11   |  12   |  13   |
| ------------- | :---: | :---: | :---: | :---: | :---: | :----: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
|               |   ?   |   ?   |   ?   |   ?   | Hour  | Minute |   ?   |   ?   |   ?   |   ?   |   ?   |   ?   |   ?   |   ?   |

#### Description of the variables
| Variable | Description                 | Unit |
| -------- | --------------------------- | ---- |
| Hour     | current hour in localtime   |      |
| Minute   | current minute in localtime |      |
