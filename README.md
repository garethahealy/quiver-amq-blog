[![License](https://img.shields.io/hexpm/l/plug.svg?maxAge=2592000)]()

# Using Quiver with Red Hat AMQ on Red Hat OpenShift Container Platform
Unsure what Red Hat AMQ is and how it can help? Check out this [webinar.](https://youtu.be/mkqVxWZfGfI)

As part of the Red Hat [UKI Professional Services](https://www.redhat.com/en/services/consulting) team,
I have worked with several customers who are implementing [AMQ Broker](https://developers.redhat.com/products/amq/overview/) on
[OpenShift Container Platform (OCP)](https://developers.redhat.com/products/openshift/overview/). One question customers typically ask is:
"_How do we validate the AMQ configuration is correct for our scenario?_".

Previously, I would of suggested one of the following:
- [ActiveMQ Perf Maven Plugin](http://activemq.apache.org/activemq-performance-module-users-manual.html)
- [JMeter](http://activemq.apache.org/jmeter-performance-tests.html)
- [Gatling](https://gatling.io/docs/3.0/jms/)

These tools can give you indicators around:
- Is the broker up and running? i.e.: can receive/publish messages for this configuration?
- Can the broker handle a certain performance characteristic? i.e.: what is my minimum publish rate per second for this configuration?
- and many more, but...

The problem with the above tools is that you cannot choose the client technology; which could lead to real-world differences
and limited technology choices that might lead you down the wrong technology path. i.e.:
- do you get the same performance from JMeter vs the [AMQ clients](https://access.redhat.com/documentation/en-us/red_hat_amq/7.2/html/amq_clients_overview/components) you would use in production? comparing like for like, apples with apples

So what do I think is the answer? [Quiver](https://github.com/ssorj/quiver)[*[1]](#DISCLAIMER).

## What is Quiver?
    https://github.com/ssorj/quiver

Straight from the horse's mouth:

> Quiver implementations are native clients (and sometimes also servers) in various languages and APIs that either
> send or receive messages and write raw information about the transfers to standard output.
> They are deliberately simple.

> The quiver-arrow command runs a single implementation in send or receive mode and captures its output.
> It has options for defining the execution parameters, selecting the implementation, and reporting statistics.

> The quiver command launches a pair of quiver-arrow instances, one sender and one receiver, and produces a
> summary of the end-to-end transmission of messages.

In my own words;

> Quiver is a tool that starts both a publisher and receiver client with the aim of completing as fast as possible.
> Once the publisher and receiver have completed; a statistical output is generated which can give you insight into how the test has performed.

> The clients are based on the [Apache Qpid](https://qpid.apache.org/) project that offers; Java, C++, Python and Javascript
> implementations to name a few. The clients primarily target [JMS](https://download.oracle.com/otndocs/jcp/jms-2_0-fr-eval-spec/)
> and [AMQP](http://www.amqp.org/resources/download) users.

## Recorded demo
Feeling lazy and want to watch a [asciinema](https://asciinema.org/) recording?

The demo will show the following:
- AMQ Broker and AMQ Interconnect being deployed onto OKD running on minishift
- Send and receive messages via Quiver using the Core protocol to the broker
- Send and receive messages via Quiver using an JMS client over AMQP protocol to the broker
- Send and receive messages via Quiver using an JMS client over AMQP protocol to interconnect

[![asciicast](https://asciinema.org/a/amuJ4J5V9cey9aCKi33q0ybFp.png)](https://asciinema.org/a/amuJ4J5V9cey9aCKi33q0ybFp)

## Do it yourself demo
Want to play with the Quiver, AMQ and OCP yourself?

The demo uses an OCP template which requires two arguments:
- DOCKER_IMAGE; location of the fully qualified docker pull URL. This can be resolved via the imported image stream; oc get is quiver -o jsonpath='{.status.dockerImageRepository}'
- DOCKER_CMD; the quiver command, in JSON array format, you want to execute.

### Demo prerequisites
It is presumed that you have the following:
- Basic knowledge of the OCP CLI and web console
- A running [OpenShift Container Platform (OCP)](https://developers.redhat.com/products/openshift/overview/) or [minishift](https://github.com/minishift/minishift) environment
- AMQ Broker templates and image streams installed by following; ['2.1. Installing AMQ Broker on OpenShift Container Platform image streams and application templates'](https://access.redhat.com/documentation/en-us/red_hat_amq/7.2/html-single/deploying_amq_broker_on_openshift_container_platform/index#install-deploy-ocp-broker-ocp)
- AMQ Interconnect templates and image streams installed by following; ['2.1. Verifying the availability of AMQ Interconnect templates'](https://access.redhat.com/documentation/en-us/red_hat_amq/7.2/html/deploying_amq_interconnect_on_openshift_container_platform/preparing-to-deploy-router-ocp)

### Setup
1. Create project

        $ oc new-project quiver

2. Import the Quiver image

        $ oc import-image quiver:latest --from=docker.io/ssorj/quiver --confirm

3. Add the view role to the default service account

        $ oc policy add-role-to-user view system:serviceaccount:$(oc project -q):default

4. Deploy AMQ Broker

        $ oc new-app --template=amq-broker-72-basic \
           -e AMQ_PROTOCOL=openwire,amqp,stomp,mqtt \
           -e AMQ_QUEUES=quiver \
           -e AMQ_ADDRESSES=quiver \
           -e AMQ_USER=anonymous \
           -e ADMIN_PASSWORD=password

        $ oc get pods --watch

5. Deploy AMQ Interconnect

        $ oc new-app --template=amq-interconnect-1-basic

        $ oc get pods --watch

### Send messages to AMQ Broker
6. Send 1,000,000 messages to the broker via the [core protocol](https://access.redhat.com/documentation/en-us/red_hat_amq/7.2/html-single/using_the_amq_core_protocol_jms_client/)

        $ oc process -f https://raw.githubusercontent.com/ssorj/quiver/0.2.0/packaging/openshift/openshift-pod-template.yml \
                DOCKER_IMAGE=$(oc get is quiver -o jsonpath='{.status.dockerImageRepository}'):latest \
                DOCKER_CMD="[\"quiver\", \"//broker-amq-tcp:61616/activemq-artemis-jms\", \"--impl\", \"activemq-artemis-jms\", \"--verbose\"]" \
                | oc create -f -

7. Look at output for core protocol pod

        $ oc get pods --watch
        $ oc logs -f {insert value of new pod name}

            ---------------------- Sender -----------------------  --------------------- Receiver ----------------------  --------
            Time [s]      Count [m]  Rate [m/s]  CPU [%]  RSS [M]  Time [s]      Count [m]  Rate [m/s]  CPU [%]  RSS [M]  Lat [ms]
            -----------------------------------------------------  -----------------------------------------------------  --------
                 2.5              0           0       88     34.5       2.5              0           0       92     33.9         0
                 4.5              0           0      103     73.5       4.5              0           0      104     73.1         0
                 6.6            455         227       53     85.5       6.5            361         180       42     81.7        71
                 8.6          3,337       1,440       51    100.1       8.5          3,252       1,439       79    103.9       113
                10.6         22,907       9,765       89    116.7      10.5         23,612      10,144      103    117.1        18
                12.6         83,588      30,325       68    117.6      12.5         84,570      30,464       70    121.0         4
                14.6        167,632      41,980       63    117.7      14.5        171,309      43,305       60    122.6         3
                16.6        254,407      43,323       61    113.3      16.5        254,795      41,722       49    122.6         2
                18.6        355,138      50,315       61    113.7      18.5        357,074      51,114       51    124.2         2
                20.6        450,408      47,611       58    113.7      20.5        451,403      47,164       48    124.2         2
                22.6        540,368      44,935       63    113.7      22.6        540,310      44,409       54    124.4         3
                24.6        634,880      47,209       59    113.7      24.6        641,024      50,332       56    124.4         2
                26.6        735,611      50,290       59    113.8      26.6        735,593      47,261       47    124.4         2
                28.6        820,261      42,283       60    113.9      28.6        820,525      42,445       51    124.4         2
                30.6        927,971      53,828       65    113.9      30.6        937,502      58,459       63    124.6         3
                32.6        971,207      21,607       36    113.9      32.6        971,354      16,926       20    124.6         5
                34.6      1,000,000      14,382       22      0.0      34.6      1,000,000      14,316       19      0.0         3
            --------------------------------------------------------------------------------
            Subject: activemq-artemis-jms //broker-amq-tcp:61616/activemq-artemis-jms (/tmp/quiver-_1cg5ct3)
            Count:                                          1,000,000 messages
            Body size:                                            100 bytes
            Credit window:                                      1,000 messages
            Duration:                                            27.7 s
            Sender rate:                                       36,162 messages/s
            Receiver rate:                                     36,539 messages/s
            End-to-end rate:                                   36,152 messages/s
            Latencies by percentile:
                      0%:        0 ms                90.00%:        5 ms
                     25%:        2 ms                99.00%:       22 ms
                     50%:        3 ms                99.90%:      122 ms
                    100%:      449 ms                99.99%:      334 ms
            --------------------------------------------------------------------------------

8. Send 1,000,000 messages to the broker via the [JMS client over AMQP protocol](https://access.redhat.com/documentation/en-us/red_hat_amq/7.2/html-single/using_the_amq_jms_client/)

        $ oc process -f https://raw.githubusercontent.com/ssorj/quiver/0.2.0/packaging/openshift/openshift-pod-template.yml \
                DOCKER_IMAGE=$(oc get is quiver -o jsonpath='{.status.dockerImageRepository}'):latest \
                DOCKER_CMD="[\"quiver\", \"amqp://broker-amq-amqp:5672/qpid-jms-over-ampq\", \"--impl\", \"qpid-jms\", \"--verbose\"]" \
                | oc create -f -

9. Look at output for the AMQP client pod

        $ oc get pods --watch
        $ oc logs -f {insert value of new pod name}

            ---------------------- Sender -----------------------  --------------------- Receiver ----------------------  --------
            Time [s]      Count [m]  Rate [m/s]  CPU [%]  RSS [M]  Time [s]      Count [m]  Rate [m/s]  CPU [%]  RSS [M]  Lat [ms]
            -----------------------------------------------------  -----------------------------------------------------  --------
                 2.3              0           0       48     66.0       2.2              0           0       48     66.1         0
                 4.3          2,159       1,078       70    108.3       4.2          1,776         888       53     85.3       109
                 6.3         12,952       5,391      117    115.9       6.2         11,914       5,061       82    109.2        41
                 8.3         32,688       9,863      119    122.1       8.2         32,190      10,128       87    109.9        49
                10.3         64,338      15,825       86    122.5      10.2         63,422      15,608       78    110.6        14
                12.3        100,451      18,038       82    117.7      12.2        100,080      18,320       67    110.7         6
                14.3        133,952      16,750       79    117.8      14.2        133,556      16,738       64    110.8         4
                16.3        176,502      21,254       83    118.1      16.2        175,628      21,015       72    111.3        41
                18.3        207,925      15,680       75    118.1      18.2        207,385      15,863       61    111.3         8
                20.3        245,339      18,688       78    118.7      20.2        244,603      18,590       64    111.4        13
                22.3        282,387      18,515       82    118.7      22.2        280,911      18,145       64    111.4         6
                24.3        315,644      16,628       74    118.9      24.2        315,196      17,142       66    111.4         9
                26.3        352,936      18,618       79    118.9      26.2        350,189      17,488       66    111.4         7
                28.3        395,607      21,314       84    118.9      28.2        393,374      21,582       73    111.4        27
                30.3        426,664      15,528       77    118.9      30.3        426,040      16,325       63    111.4        15
                32.3        464,934      19,116       79    118.9      32.3        462,652      18,279       63    111.6         8
                34.3        497,946      16,490       77    118.9      34.3        496,532      16,932       63    111.6         8
                36.3        532,548      17,275       76    118.9      36.3        531,424      17,402       64    111.6         5
                38.3        592,338      29,865       84    118.9      38.3        582,295      25,410       78    111.6       190
                40.3        624,494      16,070       69    118.9      40.3        623,761      20,723       68    111.6       125
                42.3        673,035      24,246       83    118.9      42.3        666,845      21,531       70    111.6        20
                44.3        716,318      21,620       83    118.9      44.3        715,896      24,526       76    111.6        39
                46.3        746,396      15,031       70    118.9      46.3        745,731      14,903       56    111.6         6
                48.3        772,562      13,076       74    118.9      48.3        771,723      12,990       59    111.6         4
                50.3        811,688      19,563       80    118.9      50.3        809,851      19,045       68    111.6        12
                52.3        864,019      26,048       87    119.1      52.3        855,969      23,036       76    111.6        23
                54.3        899,599      17,772       76    119.1      54.3        899,457      21,722       69    111.6        62
                56.3        936,768      18,566       80    119.1      56.3        935,967      18,246       64    111.6         7
                58.3        972,349      17,782       77    119.1      58.3        972,073      18,035       66    111.6         9
                60.3      1,000,000      13,826       61      0.0      60.3      1,000,000      13,964       50      0.0         7
            --------------------------------------------------------------------------------
            Subject: qpid-jms amqp://broker-amq-amqp:5672/qpid-jms-over-ampq (/tmp/quiver-7i3qza0d)
            Count:                                          1,000,000 messages
            Body size:                                            100 bytes
            Credit window:                                      1,000 messages
            Duration:                                            56.8 s
            Sender rate:                                       17,603 messages/s
            Receiver rate:                                     17,628 messages/s
            End-to-end rate:                                   17,603 messages/s
            Latencies by percentile:
                      0%:        0 ms                90.00%:       87 ms
                     25%:        1 ms                99.00%:      417 ms
                     50%:        4 ms                99.90%:      509 ms
                    100%:      526 ms                99.99%:      521 ms
            --------------------------------------------------------------------------------

### Send messages to AMQ Interconnect
10. Send 1,000,000 messages to the interconnect via the JMS client over AMQP protocol

        $ oc process -f https://raw.githubusercontent.com/ssorj/quiver/0.2.0/packaging/openshift/openshift-pod-template.yml \
                DOCKER_IMAGE=$(oc get is quiver -o jsonpath='{.status.dockerImageRepository}'):latest \
                DOCKER_CMD="[\"quiver\", \"amqp://amq-interconnect:5672/qpid-jms-over-ampq-via-interconnect\", \"--impl\", \"qpid-jms\", \"--verbose\"]" \
                | oc create -f -

11. Look at output for AMQP client pod

        $ oc get pods --watch
        $ oc logs -f {insert value of new pod name}

            ---------------------- Sender -----------------------  --------------------- Receiver ----------------------  --------
            Time [s]      Count [m]  Rate [m/s]  CPU [%]  RSS [M]  Time [s]      Count [m]  Rate [m/s]  CPU [%]  RSS [M]  Lat [ms]
            -----------------------------------------------------  -----------------------------------------------------  --------
                 2.3          1,529         764      132    104.9       2.3          1,465         732      120     84.9        26
                 4.3         13,201       5,827      144    114.9       4.3         13,553       6,035      124    112.1        18
                 6.3         36,908      11,848      116    114.5       6.3         36,900      11,650      101    113.4         7
                 8.3         74,392      18,695       77    114.8       8.3         74,481      18,772       75    113.6         5
                10.3        113,411      19,500       80    115.9      10.3        113,632      19,566       68    113.8         5
                12.3        163,908      25,223       82    115.9      12.3        164,908      25,612       76    113.8         4
                14.3        203,768      19,890       86    116.5      14.3        202,733      18,903       58    114.1         5
                16.3        235,435      15,818       77    116.7      16.3        235,602      16,426       61    114.1         6
                18.3        279,697      22,109       74    116.7      18.3        279,697      22,036       65    114.1         4
                20.3        315,399      17,815       75    116.7      20.3        315,196      17,732       64    114.1         5
                22.3        352,813      18,660       82    116.8      22.3        352,919      18,843       72    114.2         5
                24.3        392,184      19,666       70    116.8      24.3        392,767      19,914       60    114.2         5
                26.3        433,144      20,460       73    116.8      26.3        433,524      20,368       62    114.2         5
                28.3        466,401      16,612       69    116.8      28.3        466,191      16,334       58    114.2         5
                30.3        496,724      15,146       57    116.8      30.3        496,532      15,155       46    114.2         7
                32.3        530,959      17,092       63    116.8      32.3        531,222      17,328       56    114.3         7
                34.3        558,102      13,538       57    116.8      34.3        558,124      13,444       46    114.4         7
                36.3        596,250      19,055       64    116.8      36.3        596,454      19,146       54    114.4         6
                38.3        626,084      14,850       54    116.8      38.3        625,885      14,693       46    114.4         7
                40.3        660,075      16,979       66    116.8      40.3        660,170      17,125       54    114.4         7
                42.3        691,742      15,802       58    116.8      42.3        691,320      15,559       47    114.4         6
                44.3        724,877      16,543       60    116.8      44.3        723,986      16,309       49    114.4         7
                46.3        757,889      16,498       59    116.8      46.3        758,373      17,176       51    114.4         7
                48.4        791,024      16,543       63    116.8      48.3        791,040      16,277       50    114.4         6
                50.4        826,115      17,519       62    116.8      50.3        825,931      17,446       51    114.4         6
                52.4        858,761      16,307       59    116.8      52.3        858,699      16,368       48    114.4         6
                54.4        898,621      19,910       66    116.8      54.3        898,446      19,854       54    114.4         5
                56.4        951,808      26,567       86    116.8      56.3        951,542      26,521       73    114.4         4
                58.4        976,017      12,074       60    116.8      58.3        975,410      11,922       53    114.4         8
                60.4      1,000,000      11,992       52      0.0      60.3      1,000,000      12,283       46      0.0         5
            --------------------------------------------------------------------------------
            Subject: qpid-jms amqp://amq-interconnect:5672/qpid-jms-over-ampq-via-interconnect (/tmp/quiver-iy_ln_bf)
            Count:                                          1,000,000 messages
            Body size:                                            100 bytes
            Credit window:                                      1,000 messages
            Duration:                                            58.6 s
            Sender rate:                                       17,056 messages/s
            Receiver rate:                                     17,070 messages/s
            End-to-end rate:                                   17,055 messages/s
            Latencies by percentile:
                      0%:        0 ms                90.00%:       11 ms
                     25%:        3 ms                99.00%:       22 ms
                     50%:        5 ms                99.90%:       56 ms
                    100%:      147 ms                99.99%:      142 ms
            --------------------------------------------------------------------------------

### Cleanup
12. Cleanup and delete all quiver pods

        $ oc delete pod -l app=quiver

## Does your client use another language?
If your language of choice was not shown in the demo; the good news is that Quiver supports several [implementations](https://github.com/ssorj/quiver#quiver-1).
The below are provided examples:

    $ oc process -f https://raw.githubusercontent.com/ssorj/quiver/0.2.0/packaging/openshift/openshift-pod-template.yml \
            DOCKER_IMAGE=$(oc get is quiver -o jsonpath='{.status.dockerImageRepository}'):latest \
            DOCKER_CMD="[\"quiver\", \"//broker-amq-tcp:61616/activemq-jms\", \"--impl\", \"activemq-jms\", \"--verbose\"]" \
            | oc create -f -

    $ oc process -f https://raw.githubusercontent.com/ssorj/quiver/0.2.0/packaging/openshift/openshift-pod-template.yml \
            DOCKER_IMAGE=$(oc get is quiver -o jsonpath='{.status.dockerImageRepository}'):latest \
            DOCKER_CMD="[\"quiver\", \"//broker-amq-tcp:61616/qpid-messaging-cpp\", \"--impl\", \"qpid-messaging-cpp\", \"--verbose\"]" \
            | oc create -f -

    $ oc process -f https://raw.githubusercontent.com/ssorj/quiver/0.2.0/packaging/openshift/openshift-pod-template.yml \
            DOCKER_IMAGE=$(oc get is quiver -o jsonpath='{.status.dockerImageRepository}'):latest \
            DOCKER_CMD="[\"quiver\", \"//broker-amq-tcp:61616/qpid-messaging-python\", \"--impl\", \"qpid-messaging-python\", \"--verbose\"]" \
            | oc create -f -

    $ oc process -f https://raw.githubusercontent.com/ssorj/quiver/0.2.0/packaging/openshift/openshift-pod-template.yml \
            DOCKER_IMAGE=$(oc get is quiver -o jsonpath='{.status.dockerImageRepository}'):latest \
            DOCKER_CMD="[\"quiver\", \"//broker-amq-tcp:61616/qpid-proton-c\", \"--impl\", \"qpid-proton-c\", \"--verbose\"]" \
            | oc create -f -

    $ oc process -f https://raw.githubusercontent.com/ssorj/quiver/0.2.0/packaging/openshift/openshift-pod-template.yml \
            DOCKER_IMAGE=$(oc get is quiver -o jsonpath='{.status.dockerImageRepository}'):latest \
            DOCKER_CMD="[\"quiver\", \"//broker-amq-tcp:61616/qpid-proton-cpp\", \"--impl\", \"qpid-proton-cpp\", \"--verbose\"]" \
            | oc create -f -

    $ oc process -f https://raw.githubusercontent.com/ssorj/quiver/0.2.0/packaging/openshift/openshift-pod-template.yml \
            DOCKER_IMAGE=$(oc get is quiver -o jsonpath='{.status.dockerImageRepository}'):latest \
            DOCKER_CMD="[\"quiver\", \"//broker-amq-tcp:61616/qpid-proton-python\", \"--impl\", \"qpid-proton-python\", \"--verbose\"]" \
            | oc create -f -

    $ oc process -f https://raw.githubusercontent.com/ssorj/quiver/0.2.0/packaging/openshift/openshift-pod-template.yml \
            DOCKER_IMAGE=$(oc get is quiver -o jsonpath='{.status.dockerImageRepository}'):latest \
            DOCKER_CMD="[\"quiver\", \"//broker-amq-tcp:61616/rhea\", \"--impl\", \"rhea\", \"--verbose\"]" \
            | oc create -f -

## Food for thought
Hopefully, the demo has shown how easy it is to use Quiver to interact with AMQ Broker and AMQ Interconnect on OCP.

It should have also raised questions around how Quiver could be integrated into your own systems. For example:
- Could Quiver be part of your mvn integration tests?
- Could Quiver be added as a stage within your CI/CD pipeline for smoke testing?
- Could the results of the smoke test pass/fail the deployment?
- Could Quiver be used as part of a heart-beat monitoring system to validate that the broker is alive?

These are several scenarios for which I see Quiver being highly useful and hopefully you've also thought of others.

## <a name="DISCLAIMER"></a>DISCLAIMER
[1] Although Quiver is developed by Red Hat employees; it is not supported under a Red Hat subscription and is strictly an upstream project.
