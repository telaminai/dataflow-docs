---
title: DataFlow
layout: default
nav_order: 1
---
# Fluxtion DataFlow
---- 

* Java library that saves time developing and maintaining reactive applications
* Quick start development up and running in a couple of minutes
* Native Kafka connector for simple enterprise integration
{: .fs-6 }

## Example
-----
DataFlow stream api combines event feeds and user functions into a processing directed acyclic graph. Wiring and 
dispatch is automatically resolved by DataFlowBuilder. The returned DataFlow instance exposes a simple onEvent method 
for integration into a host application.

<div class="tab">
  <button class="tablinks2" onclick="openTab2(event, 'Windowing')" id="defaultExample">Windowing</button>
  <button class="tablinks2" onclick="openTab2(event, 'Multi feed join')" >Multi feed join</button>
  <button class="tablinks2" onclick="openTab2(event, 'TriggerOverride')" >Trigger override</button>
</div>

<div id="Windowing" class="tabcontent2">
<div markdown="1">
{% highlight java %}
public class WindowExample {
    record CarTracker(String id, double speed) {}
    public static void main(String[] args) {
        //calculate average speed, sliding window 5 buckets of 200 millis
        DataFlow averageCarSpeed = DataFlowBuilder.subscribe(CarTracker::speed)
                .slidingAggregate(Aggregates.doubleAverageFactory(), 200, 5)
                .sink("average car speed")
                .build();
        //register an output sink
        averageCarSpeed.addSink("average car speed", System.out::println);
        //send data from an unbounded real-time feed
        Executors.newSingleThreadScheduledExecutor().scheduleAtFixedRate(
                () -> averageCarSpeed.onEvent(new CarTracker("car-reg", new Random().nextDouble(100))),
                100, 100, TimeUnit.MILLISECONDS);
    }
}
{% endhighlight %}
</div>
</div>

<div id="Multi feed join" class="tabcontent2">
<div markdown="1">
{% highlight java %}
public class MultiFeedJoinExample {
    public static void main(String[] args) {
        //stream of realtime machine temperatures grouped by machineId
        DataFlow currentMachineTemp = DataFlowBuilder.groupBy(
                MachineReadingEvent::id, MachineReadingEvent::temp);
        //create a stream of averaged machine sliding temps,
        //with a 4-second window and 1 second buckets grouped by machine id
        DataFlow avgMachineTemp = DataFlowBuilder.subscribe(MachineReadingEvent.class)
                .groupBySliding(
                        MachineReadingEvent::id,
                        MachineReadingEvent::temp,
                        DoubleAverageFlowFunction::new,
                        1000,
                        4);
        //join machine profiles with contacts and then with readings.
        //Publish alarms with stateful user function
        DataFlow tempMonitor = DataFlowBuilder.groupBy(MachineProfileEvent::id)
                .mapValues(MachineState::new)
                .mapBi(
                        DataFlowBuilder.groupBy(SupportContactEvent::locationCode),
                        Helpers::addContact)
                .innerJoin(currentMachineTemp, MachineState::setCurrentTemperature)
                .innerJoin(avgMachineTemp, MachineState::setAvgTemperature)
                .publishTriggerOverride(FixedRateTrigger.atMillis(1_000))
                .filterValues(MachineState::outsideOperatingTemp)
                .map(GroupBy::toMap)
                .map(new AlarmDeltaFilter()::updateActiveAlarms)
                .filter(AlarmDeltaFilter::isChanged)
                .sink("alarmPublisher")
                .build();

        runSimulation(tempMonitor);
    }

    private static void runSimulation(DataFlow tempMonitor) {
        //any java.util.Consumer can be used as sink
        tempMonitor.addSink("alarmPublisher", Helpers::prettyPrintAlarms);

        //set up machine locations
        tempMonitor.onEvent(new MachineProfileEvent("server_GOOG", LocationCode.USA_EAST_1, 70, 48));
        tempMonitor.onEvent(new MachineProfileEvent("server_AMZN", LocationCode.USA_EAST_1, 99.999, 65));
        tempMonitor.onEvent(new MachineProfileEvent("server_MSFT", LocationCode.USA_EAST_2,92, 49.99));
        tempMonitor.onEvent(new MachineProfileEvent("server_TKM", LocationCode.USA_EAST_2,102, 50.0001));

        //set up support contacts
        tempMonitor.onEvent(new SupportContactEvent("Jean", LocationCode.USA_EAST_1, "jean@fluxtion.com"));
        tempMonitor.onEvent(new SupportContactEvent("Tandy", LocationCode.USA_EAST_2, "tandy@fluxtion.com"));

        //Send random MachineReadingEvent using `DataFlow.onEvent` 
        Random random = new Random();
        final String[] MACHINE_IDS = new String[]{"server_GOOG", "server_AMZN", "server_MSFT", "server_TKM"};
        Executors.newSingleThreadScheduledExecutor().scheduleAtFixedRate(() -> {
                    String machineId = MACHINE_IDS[random.nextInt(MACHINE_IDS.length)];
                    double temperatureReading = random.nextDouble() * 100;
                    tempMonitor.onEvent(new MachineReadingEvent(machineId, temperatureReading));
                },
                10_000, 1, TimeUnit.MICROSECONDS);

        System.out.println("Simulation started - wait four seconds for first machine readings\n");
    }
}
{% endhighlight %}
</div>
</div>


<div id="TriggerOverride" class="tabcontent2">
<div markdown="1">
{% highlight java %}
public class TriggerExample {
    public static void main(String[] args) {
        DataFlow sumDataFlow = DataFlowBuilder.subscribe(Integer.class)
                .aggregate(Aggregates.intSumFactory())
                .resetTrigger(DataFlowBuilder.subscribeToSignal("resetTrigger"))
                .filter(i -> i != 0)
                .publishTriggerOverride(DataFlowBuilder.subscribeToSignal("publishSumTrigger"))
                .console("Current sun:{}")
                .build();

        sumDataFlow.onEvent(10);
        sumDataFlow.onEvent(50);
        sumDataFlow.onEvent(32);
        //publish
        sumDataFlow.publishSignal("publishSumTrigger");

        //reset sum
        sumDataFlow.publishSignal("resetTrigger");

        //new sum
        sumDataFlow.onEvent(8);
        sumDataFlow.onEvent(17);
        //publish
        sumDataFlow.publishSignal("publishSumTrigger");
    }
}
{% endhighlight %}
</div>
</div>

## Latest release
----
Open source on [GitHub]({{site.fluxtion_src}}), artifacts published to maven central.

<div class="tab">
  <button class="tablinks" onclick="openTab(event, 'Maven')">Maven</button>
  <button class="tablinks" onclick="openTab(event, 'Gradle')" id="defaultOpen">Gradle</button>
</div>
<div id="Maven" class="tabcontent">
<div markdown="1">
{% highlight xml %}
    <dependencies>
        <dependency>
            <groupId>com.fluxtion</groupId>
            <artifactId>runtime</artifactId>
            <version>{{site.fluxtion_version}}</version>
        </dependency>
        <dependency>
            <groupId>com.fluxtion</groupId>
            <artifactId>compiler</artifactId>
            <version>{{site.fluxtion_version}}</version>
        </dependency>
    </dependencies>
{% endhighlight %}
</div>
</div>
<div id="Gradle" class="tabcontent">
<div markdown="1">
{% highlight groovy %}
implementation 'com.fluxtion:runtime:{{site.fluxtion_version}}'
implementation 'com.fluxtion:compiler:{{site.fluxtion_version}}'
{% endhighlight %}
</div>
</div>

<script>
document.getElementById("defaultOpen").click();
document.getElementById("defaultExample").click();
</script>