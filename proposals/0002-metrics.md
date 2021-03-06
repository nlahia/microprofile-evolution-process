# Metrics via HTTP endpoints

* Proposal: [MP-0002](0002-metrics.md)
* Authors: [Heiko W. Rupp](https://github.com/pilhuhn)
* Status: **Awaiting review**

*During the review process, add the following fields as needed:*

* Decision Notes: [Discussion thread topic covering the  Rationale](https://groups.google.com/forum/#!forum/microprofile), [Discussion thread topic with additional Commentary](https://groups.google.com/forum/#!forum/microprofile)

## Introduction

To ensure reliable operation of the software it is necessary to monitor essential
system parameters. This enhancement proposes the addition of well-known monitoring
endpoints and metrics for each process adhering to the microprofile standard.

This proposal does not talk about health checks. There is a separate proposal
[MP-0003: Health Check Proposal](https://github.com/microprofile/microprofile-evolution-process/issues/14).

Mailinglist thread: [Monitoring and Healthcheck requirements](https://groups.google.com/forum/#!topic/microprofile/jIZAKiu76ys)

## Motivation

Reliable service of a platform needs monitoring. There is already JMX as
standard to expose metrics, but remote-JMX is not easy to deal with and
especially does not fit well in a polyglot environment where other services
are not running on the JVM.
To enable monitoring in an easy fashion it is necessary that all microprofile
implementations follow a certain standard with respect to (base) API path,
data types involved, always available metrics and return codes used.

### Difference to health checks

Health checks are primarily targeted at a quick yes/no response to the
question "Is my application still running ok?". Modern systems that
schedule the starting of applications (e.g. Kubernetes) use this
information to restart the application if the answer is 'no'.

Metrics on the other hand can help to determine the health. Beyond this
they serve to pinpoint issues, provide long term trend data for capacity
planning and pro-active discovery of issues (e.g. disk usage growing without bounds).
Metrics can also help those scheduling systems to make decisions when to scale the application
to run on more or less machines depending on the metrical values.

## Proposed solution

Data is exposed via REST over http under the `/metrics` base path

* Metrics will respond to GET requests and are exposed in a tree like fashion with sub-trees for various cases described below.
* A 'shadow tree' that responds to OPTIONS will provide the metadata
* All data is at least exposed as `application/json` data type.
* The api is discoverable. GETting a base object will list references to children

Data access must honour the http response codes, especially

* 200 for successful retrieval of an object
* 204 when retrieving a subtree that would exist, but has no content. E.g. when the application-specific subtree has no application specific metrics defined.
* 404 if an directly-addressed item does not exist. This may be a non-existing sub-tree or non-existing object
* 500 to indicate that a request failed due to "bad health". The body SHOULD contain details if possible { "details": <text> }

The Api MUST NOT return a 500 Internal Server Error code to represent a non-existing resource.

## Detailed design (draft)

### REST-Api Objects

Api-objects MAY include one or more metrics as in


    {
      "thread-count" : 33,
      "peak-thread-count" : 47,
      "total-started-thread-count" : 49,
      "current-thread-cpu-time" : 66855000,
      "current-thread-user-time" : 64003000
    }

or

    {
      "count": 45
    }

### Metadata

Metric values MUST have metadata to attach semantics to the values.
Metadata is returned as an array of objects with the following content:

* unit: a fixed set of string units from e.g. [1], [UoM] or [Metrics2.0]
* type:
     * counter: a monotonically increasing or decreasing numeric value (e.g. total number of requests received)
     * gauge: a numeric value that can arbitrarily go up and down (e.g. cpu or disk usage)
     * bool: a boolean value which can be `true` or `false`
     * string: a string
* description (optional): A human readable description of the metric
* displayName (optional): A human readable name of the metric for display purposes if the metric name is not
human readable. This could e.g. be the case when the metric name is a uuid.
* tags (optional): A list of key=value pairs, which are separated by comma.

Example:

if `GET /metrics/foo` exposes:

    {"fooVal": 12345}

then `OPTIONS /metrics/foo` will expose:

    [
      {"fooVal": {
          "unit": "ms",
          "type": "gauge",
          "description": "The size of foo after each request",
          "displayName": "Size of foo",
          "tags": "app=webshop"
         }
      }
    ]

If the endpoint exposes multiple values like this:

    {
      "fooVal": 12345,
      "barVal": 42
    }

then Metadata is returned as follows:

    [
      {"fooVal": {
          "unit": "ms",
          "type": "gauge",
          "description": "The duration of foo after each request",
          "displayName": "Duration of foo",
          "tags": "app=webshop"
         }
      },
      {"barVal": {
          "unit": "mbytes",
          "type": "gauge",
          "tags": "component=backend,app=webshop"
         }
      }
    ]


Metadata must not change over the lifetime of a process (i.e. it is not allowed
to return the units as seconds in one retrieval and as hours in a subsequent one).
The reason behind it is that e.g. a monitoring agent on Kubernetes may read the
metadata once it sees the new container and store it. It may not periodically
re-query the process for the metadata.

In fact that metadata should probably not change during the life-time of the
whole container image or an application, as all containers spawned from it
will be "the same" and form part of an app, where it would be confusing in
an overall view if the same metric has different metadata.

Metadata SHOULD support caching via cache control headers and SHOULD reply with a 304 Not Modified response accordingly.

## Required metrics

Required metrics is a list of metrics that all vendors need to implement. They are exposed under `/metrics/base`

The following is a list of required metrics if the application uses the data. E.g. if the application does not use any data source, then there will be no data sources listed.

* java.lang.* metrics from the MBeanServer (read-only metrics, no writing, no operations)
 Q: should this really include all values?
   * especially Garbage collector stuff is pretty convoluted
* List of datasources with connections in use (list can be empty)
* ...

Values from the MBean server are encoded with `MBean-Name/attribute[#field]` name to retrieve a single attribute.

E.g. `GET /metrics/base/java.lang:type=Memory/ObjectPendingFinalizationCount` to only get that count.
For MBeans attributes that are of type `CompositeData`, the `#field` will return a single item of this composite
data.

Q: should we expose current total memory usage (heap+non heap) in a separate item? I am in favour of that as other non-JVM
environments do may not be able to report fine grained values, but only a total.

Q: should current thread count be exposed in a separate item?


## Vendor specific data

It is possible for microprofile server implementors to supply their specific metrics data on top of the basic set listed above.
Vendor specific metrics are exposed under `/metrics/vendor`.

Examples for vendor specific data could be metrics like

* OSGi statistics if the Microprofile-enabled container internally runs on top of OSGi.
* Statistics of some internal caching modules

Vendor specific metrics are not supposed to be portable between different implementations
of Microprofile

## Application specific data

It is possible for applications to expose their own application metrics on top of the basic set listed above.
Application specific metrics are exposed under `/metrics/application`.

Application specific metrics are supposed to be portable to other implementations of
the Microprofile if the application can run there unmodified.

## Security

It must be possible to secure the endpoints via the usual means

Accessing `/metrics` without valid credentials must return a 401 Unauthorised header

Q: should we return 503 Service Unavailable if the server detects an internal bad health state when authorisation is required or stick to a 401 to not expose additional hints to attackers.

A server SHOULD implement TLS encryption by default

## Configuration

### Required + Vendor specific metrics

Q: Do we want to mandate this or can/should each vendor do as they like?

A sample configuration in YAML may look like this:

    base:
      - name: "thread-count"
        mbean: "java.lang:type=Threading/ThreadCount"
        description: "Number of currently deployed threads"
        unit: "none"
        type: "gauge"
        displayName: "Current Thread count"
      - name: "peak-thread-count"
        mbean: "java.lang:type=Threading/PeakThreadCount"
        description: "Max number of threads"
        unit: "none"
        type: "gauge"
      - name: "total-started-thread-count"
        mbean: "java.lang:type=Threading/TotalStartedThreadCount"
        description: "Number of threads started for this server"
        unit: "none"
        type: "counter"
      - name: "max-heap"
        mbean: "java.lang:type=Memory/HeapMemoryUsage#max"
        description: "Number of threads started for this server"
        unit: "bytes"
        type: "counter"
        tags: "kind=memory"

    vendor:
      - name: "msc-loaded-modules"
        mbean: "jboss.modules:type=ModuleLoader,name=BootModuleLoader-2/LoadedModuleCount"
        description: "Number of loaded modules"
        unit: "none"
        type: "gauge"

This configuration can be backed into the runtime. Application specific metrics don't show up here.

### Application metrics

To access application metrics and its metadata a class `ApplicationMetric` is made available which can be injected
via CDI

    @Inject
    ApplicationMetric applicationMetric;

This can then be used to register MetaData

    MetadataEntry demoEntry = new MetadataEntry("demo",  // Name, mandatory
            null,                                        // display name
            "Just a demo value",                         // description
            "gauge",                                     // type
            "none");                                     // unit
    demoEntry.setTags("app=demo");
    applicationMetric.registerMetric("demo", demoEntry);


Registration of a metric is mandatory before it can be used (and is published over the REST api)

Writing a value:

    applicationMetric.bumpValue("demo",1);  // Increment by 1

or

    applicationMetric.setValue("demo",42);   // set to absolute value 42


### Supplying of Tags

Tags can be supplied in two ways

* At the level of a metric shown in the above examples
* At the application server level by passing the list of tags in an environment variable `MP_METRICS_TAGS`

    `export MP_METRICS_TAGS=app=shop,tier=integration`

Global tags will be appended to the per-metric tags.

## Java API classes

### Metadata

    /**
     * Bean holding the metadata of one single metric
     */
    @SuppressWarnings("unused")
    public class MetadataEntry {

      /**
       * Name of the metric.
       * <p>Exposed over REST</p>
       */
      private String name;
      /**
       * Display name of the metric. If not set, the name is taken.
       * <p>Exposed over REST</p>
       */
      private String displayName;
      /**
       * The mbean info to retrieve the data from.
       * Format is objectname/attribute[#field], with field
       * being one field in a composite attribute.
       * E.g. java.lang:type=Memory/HeapMemoryUsage#max
       */
      @JsonIgnore
      private String mbean;
      /**
       * A human readable description.
       * <p>Exposed over REST</p>
       */
      private String description;
      /**
       * Type of the metric.
       * <p>Exposed over REST</p>
       */
      private MpMType type;
      /**
       * Unit of the metric.
       * <p>Exposed over REST</p>
       */
      private MpMUnit unit;
      /**
       * Tags of the metric. Augmented by global tags.
       * <p>Exposed over REST</p>
       */
      @JsonInclude(JsonInclude.Include.NON_NULL)
      private String tags;

      public MetadataEntry(String name, MpMType type, MpMUnit unit) {
        this.name = name;
        this.type = type;
        this.unit = unit;
      }


    [...]
    }

### Metric type

    public enum MpMType {
      /**
       * A Counter monotonically in-/decreases its values.
       * An example could be the number of Transactions committed.
        */
      COUNTER("counter"),
      /**
       * A Gauge has values that 'arbitrarily' go up/down at each
       * sampling. An example could be CPU load
       */
      GAUGE("gauge")
      ;

      /**
       * Convert the string representation in to an enum
       * @param in the String representation
       * @return the matching Enum
       * @throws IllegalArgumentException if in is not a valid enum value
       */
      public static MpMType from(String in) { [..] }

      [...]
    }


### Units

    public enum MpMUnit {
      /** Dummy to say that this has no unit */
      NONE ("none"),

      /** A single Bit. Not defined by SI, but by IEC 60027 */
      BIT("bit"),
      /** 1000 {@link #BIT} */
      KILOBIT("kilobit"),
      /** 1000 {@link #KIBIBIT} */
      MEGABIT("megabit"),
      /** 1000 {@link #MEGABIT} */
      GIGABIT("gigabit"),
      /** 1024 {@link #BIT} */
      KIBIBIT("kibibit"),
      /** 1024 {@link #KIBIBIT}  */
      MEBIBIT("mebibit"),
      /** 1024 {@link #MEBIBIT} */
      GIBIBIT("gibibit"), /* 1024 mebibit */

      /** 8 {@link #BIT} */
      BYTE ("byte"),
      /** 1024 {@link #BYTE} */
      KILO_BYTE ("kbyte"),
      /** 1024 {@link #KILO_BYTE} */
      MEGA_BYTE ("mbyte"),
      /** 1024 {@link #MEGA_BYTE} */
      GIGA_BYTE("gbyte"),

      NANOSECONDS("ns"),
      MICROSECONDS("us"),
      MILLISECOND("ms"),
      SECONDS("s"),
      MINUTES("m"),
      HOURS("h"),
      DAYS("d"),

      PERCENT("%")

      ;

      /**
       * Convert the string representation in to an enum
       * @param in the String representation
       * @return the matching Enum
       * @throws IllegalArgumentException if in is not a valid enum value
       */
      public static MpMUnit from(String in) { [..] }

      [...]
    }

### Application Metrics access

    public class ApplicationMetrics implements Serializable {
      /**
       * Register an application metric via its metadata.
       * It is required that each application metric has a unique name
       * set in its metadata.
       * If a metric is registered, but no value has been set yet, it will
       * return 0 - both via REST api and via #getValue
       * @param theData The metadata
       */
      public void registerMetric(MetadataEntry theData) { }

      /**
       * Store a value for key to be exposed by the rest-api
       * @param key the name of a metric
       * @param value the value
       * @throws IllegalArgumentException if the key was not registered.
       */
      public void storeValue(String key, Number value) { }

      /**
       * Retrieve the value of the key
       * @param key The name of the metric
       * @throws IllegalArgumentException if the key was not registered.
       * @return a numeric value
       */
      public Number getValue(String key) { }

      /**
       * Increase the value of a given metric by a certain delta
       * @param key The name of the metric
       * @param increment increment (could be negative to decrement)
       * @return The new value
       * @throws IllegalArgumentException if the key was not registered.
       */
      public Number bumpValue(String key, int increment) { }

    }
}

## References

[http return codes](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)

[1, RHQ Measurement Units](https://github.com/pilhuhn/rhq/blob/78eb557ae8f799b628769d76ccece61b6cb452a4/modules/core/domain/src/main/java/org/rhq/core/domain/measurement/MeasurementUnits.java#L43-79)

[UoM,JSR 363](https://github.com/unitsofmeasurement)

[Metrics2.0](http://metrics20.org/spec/)


## Impact on existing code (if applicable)

n/a

## Alternatives considered

There exists Jolokia as JMX-HTTP bridge. Using this for application specific metrics requires that those metrics
are exposed to JMX first, which are many users not familiar with.

