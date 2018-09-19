# azure-stream-analytics-session-windows

Little project to test the new session windows in Azure Stream Analytics: https://msdn.microsoft.com/en-us/azure/stream-analytics/reference/session-window-azure-stream-analytics

## What are we trying to solve
We have devices (mobile phones) which are connected to the GSM network. Currently we use tumbling and hopping windows for chuncking our streams into smaller segments which are used by downstream processing systems. Our device are offloading data which can be up to one week old, this means that we configure for late arrival. The side effect is that when the device does not send data anymore, we are not receiving the last window. It is possible to insert a synthtic even at the cost of loosing some data when the device sends data beefore the synthetic event.


For comparison I have added a query that outputs tumbling windows, those look correct:

```
SELECT 
    deviceId,
    MIN(DATEADD(millisecond, [endTime], '1970-01-01T00:00:00Z')) AS windowStart,
    System.Timestamp AS windowEnd,
    DATEDIFF(second, MIN(DATEADD(millisecond, [endTime], '1970-01-01T00:00:00Z')), System.Timestamp) AS durationSeconds
INTO
   TumblingWindows
FROM 
    TestDeviceData 
TIMESTAMP BY 
    DATEADD(millisecond, [endTime], '1970-01-01T00:00:00Z')
GROUP BY 
   deviceId, TUMBLINGWINDOW(minute,15)
```

|DeviceId                            |Start                        |End                          |Duration(sec)
|------------------------------------|-----------------------------|-----------------------------|-------------|
|1eb38af5-d031-4f06-807f-73ddef6da513|2018-04-12T16:45:04.7530000Z |2018-04-12T17:00:00.0000000Z |896          |
|1eb38af5-d031-4f06-807f-73ddef6da513|2018-04-12T17:01:07.0560000Z |2018-04-12T17:15:00.0000000Z |833          |
|1eb38af5-d031-4f06-807f-73ddef6da513|2018-04-12T17:15:10.9540000Z |2018-04-12T17:30:00.0000000Z |890          |
|1eb38af5-d031-4f06-807f-73ddef6da513|2018-04-12T17:32:30.0560000Z |2018-04-12T17:45:00.0000000Z |750          |


I defined a maxDuration for the session windows and this does not seem to be respected.

```
SELECT 
    deviceId,
    MIN(DATEADD(millisecond, [endTime], '1970-01-01T00:00:00Z')) AS windowStart,
    System.Timestamp AS windowEnd,
    DATEDIFF(second, MIN(DATEADD(millisecond, [endTime], '1970-01-01T00:00:00Z')), System.Timestamp) AS durationSeconds
INTO
   SessionWindows
FROM 
   TestDeviceData 
TIMESTAMP BY 
   DATEADD(millisecond, [endTime], '1970-01-01T00:00:00Z')
GROUP BY 
   deviceId, SESSIONWINDOW(minute, 3, 15) OVER (PARTITION BY deviceId)
```


|DeviceId                            |Start                        |End                          |Duration(sec)
|------------------------------------|-----------------------------|-----------------------------|-------------|
|1eb38af5-d031-4f06-807f-73ddef6da513|2018-04-12T16:45:04.7530000Z |2018-04-12T16:54:11.7430000Z |547          |
|1eb38af5-d031-4f06-807f-73ddef6da513|2018-04-12T16:59:02.9200000Z |2018-04-12T17:30:00.0000000Z |**1858**       |      
|1eb38af5-d031-4f06-807f-73ddef6da513|2018-04-12T17:32:30.0560000Z |2018-04-12T17:40:41.0230000Z |491          |

