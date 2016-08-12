How to create study.
====================

Under what circumstances should we step off a path? When is it essential that we finish what we start? If I bought a bag of peanuts and had an allergic reaction, no one would fault me if I threw it out. If I ended a relationship with a woman who hit me, no one would say that I had a commitment problem. But if I walk away from a seemingly secure route because my soul has other ideas, I am a flake?  


{% highlight java %}
public static KafkaConsumer consumer;
static long records_processed = 0L;
private static boolean VERBOSE = false;

public static void main(String[] args) throws IOException {
    Logger.getRootLogger().setLevel(Level.OFF);
    if (args.length < 1) {
        System.err.println("ERROR: You must specify the stream:topic.");
        System.err.println("USAGE:\n" +
                "\tjava -cp `mapr classpath`:./nyse-taq-streaming-1.0-jar-with-dependencies.jar com.mapr.test.BasicConsumer stream:topic1 [stream:topic_n] [verbose]\n" +
                "EXAMPLE:\n" +
                "\tjava -cp `mapr classpath`:./nyse-taq-streaming-1.0-jar-with-dependencies.jar com.mapr.test.BasicConsumerRelay /user/mapr/taq:test01 /user/mapr/taq:test02 /user/mapr/taq:test03 verbose");
    }

    List<String> topics = new ArrayList<String>();
    System.out.println("Consuming from streams:");
    for (int i=0; i<args.length-1; i++) {
        String topic = args[i];
        topics.add(topic);
        System.out.println("\t" + topic);
    }
{% endhighlight %}


At the heart of the struggle are two very different ideas of success—survival-driven and soul-driven. For survivalists, success is security, pragmatism, power over others. Success is the absence of material suffering.

Appendix
--------

The nourishing of the soul be damned. It is an odd and ironic thing that most of the material power in our world often resides in the hands of younger souls. Still working in the egoic and *more*!

1. one
2. two
3. three


It is an essential failure if you are called to be a monastic this time around. If you need to explore and abandon ten careers in order to stretch your soul toward its innate image, then so be it. Flake it till you make it.
