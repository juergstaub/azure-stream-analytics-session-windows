# azure-stream-analytics-session-windows

Little project to test the new session windows in Azure Stream Analytics: https://msdn.microsoft.com/en-us/azure/stream-analytics/reference/session-window-azure-stream-analytics

## What are we trying to solve
We have devices (mobile phones) which are using the GSM network to ingest telemetry data via the IoT Hub to our backend. We use tumbling and hopping windows for chuncking our streams into smaller segments which are then processed by downstream systems. Our devices are offloading data which can be up to one week old,because of that, we configure for late arrival. The side effect is that when the device does not send data anymore, we are not receiving the last window. It is possible to insert a synthetic event at the cost of loosing some data when the device offloads data with a timestamp before the synthetic event timestamp.

Receiving the last window is important because it is usually the end of a car trip and users do not want to miss this one. Also because some mathematical operations we execute, it is better to have longer time windows (as much as fits into 256KB, because of EventHub limitations), this makes the problem even worse.

We were hoping that the session window would help us for our case.

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

The datafile used is the TestDeviceData.json in the root dirctory.

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
|1eb38af5-d031-4f06-807f-73ddef6da513|2018-04-12T16:59:02.9200000Z |2018-04-12T17:30:00.0000000Z |**1858**     |      
|1eb38af5-d031-4f06-807f-73ddef6da513|2018-04-12T17:32:30.0560000Z |2018-04-12T17:40:41.0230000Z |491          |


What we don't understand is why the 2nd session has a duration of 1858 seconds, I would expect 900 seconds or 1800 max as the query seems to be executed in fixed intervals, in our case this would be 0, 15, 30, 45, 60, ...

Questions are:
- Is my assumption correct that the maximum window duration should be 2 * maxDuration?
- Are there any plans to execute the query in smaller intervals then maxDuration.
- What is the behavior with late arrival, will we always get the timeout of the last window?
- Any plans to have overlapping session windows (hopping windows with timeouts)?


Update:

I have deployed now our query to Azure and see a behavior I cannot explain. The query does not seem to get executed in the maxInterval and I do nt get any output at all.
