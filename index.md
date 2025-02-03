---
title: Home
layout: home
nav_order: 1
---
# DataFlow introduction
---- 

**Fluxtion DataFlow is a java library that saves time and effort when creating reactive applications. <br>
Designed for ease of use a developer can be up and running in a couple of minutes.**
{: .fs-5 }

A stream api combines event feeds and user functions into a processing graph. DataFlow takes care of all the wiring 
and presents a simple onEvent method for integration into a host application. 

## Example
-----
<div class="tab">
  <button class="tablinks2" onclick="openTab2(event, 'Windowing')" id="defaultExample">Windowing</button>
  <button class="tablinks2" onclick="openTab2(event, 'Filtering')">Filtering</button>
  <button class="tablinks2" onclick="openTab2(event, 'GroupBy')" >GroupBy and aggregate</button>
  <button class="tablinks2" onclick="openTab2(event, 'AnomalyDetection')" >Anomaly detection</button>
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

<div id="Filtering" class="tabcontent2">
<div markdown="1">
{% highlight java %}
//create a streaming event processor
var sep = Fluxtion.interpret(c ->
    //subscribe to "car speeds" messages
    DataFlow.subscribe(Message.class, "car speeds")
        //map function get speed
        .map(Message::speed)
        //filter and warn if speed > 100
        .filter(s -> s > 100)
        .map(s -> "warning speed > 100 detected")
        //publish to a sink output
        .sink("excess car speed"));

//init the streaming event processor and connect to your event flow source
sep.init();
connectEventFlow(sep);
{% endhighlight %}
</div>
</div>

<div id="GroupBy" class="tabcontent2">
<div markdown="1">
{% highlight java %}
//create a streaming event processor
var sep = Fluxtion.interpret(c -> {
    //reset signal for aggregate function
    var resetSignal = DataFlow.subscribe(ResetList.class).console("\n--- RESET ---");
    //subscribe to Integers
    DataFlow.subscribe(Integer.class)
        //group by odd/even and aggregate to a list
        .groupByToList(i -> i % 2 == 0 ? "evens" : "odds")
        //attaches the aggregate reset trigger to the resetSignal data flow
        .resetTrigger(resetSignal)
        .map(GroupBy::toMap)
        .console("ODD/EVEN map:{}"))
};

//init the streaming event processor and connect to your event flow source
sep.init();
sep.onEvent(1);
sep.onEvent(2);
sep.onEvent(5);
sep.onEvent(7);
sep.onEvent(2);
sep.onEvent(new ResetList());
{% endhighlight %}


{% highlight console %}
//OUTPUT
ODD/EVEN map:{odds=[1]}
ODD/EVEN map:{odds=[1], evens=[2]}
ODD/EVEN map:{odds=[1, 5], evens=[2]}
ODD/EVEN map:{odds=[1, 5, 7], evens=[2]}
ODD/EVEN map:{odds=[1, 5, 7], evens=[2, 2]}

--- RESET ---
ODD/EVEN map:{}
{% endhighlight %}
</div>
</div>

<div id="AnomalyDetection" class="tabcontent2">
<div markdown="1">
{% highlight java %}
//create a streaming event processor
var sep = Fluxtion.interpret(c -> {
    //embed a stateful user class to notify support teams
    var workScheduler = new WorkScheduler();
    //live machine locations
    DataFlow.subscribe(MachineLocation.class).push(workScheduler::setMachineLocation);
    //live support details
    DataFlow.subscribe(SupportContact.class).push(workScheduler::setSupportContact);

    //listen to machine temperature and groupBY id to a list of last 50 temperature readings
    DataFlow.subscribe(MachineReadings.class)
        .groupByFieldsGetAndAggregate(MachineReadings::temp, Collectors.listFactory(50), MachineReadings::id)
        //reset readings signal
        .resetTrigger(DataFlow.subscribeToSignal("reset"))
        //wait for at least 10 readings
        .filterValues(tempReadings -> tempReadings.size() > 10)
        //User static function convert a list of doubles to an average
        .mapValues(Mappers::listToAverage)
        //filter for avg temp > 48
        .filterValues(temp -> temp > 48)
        //push to user class that notifies support in the correct region
        .push(workScheduler::investigateMachine)
};

//init the streaming event processor and connect to your event flow source
sep.init();
connectEventFlow(sep);
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