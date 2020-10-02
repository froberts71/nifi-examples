# nifi-examples
Apache Nifi Examples by http://www.nifi.rocks
Developing A Custom Apache Nifi Processor (JSON)
Feb 7, 2015 • Phillip Grenier
https://www.nifi.rocks/developing-a-custom-apache-nifi-processor-json/

The list of available Apache Nifi processors is extensive, as documented in this post. There is still a need to develop your own; to pull data from a database, to process an uncommon file format, or many other unique situations. So to get you started, we will work through a basic processor that takes a json file as input and a json path as a parameter to place into the contents and an attribute. The full source is hosted on Github.

Setup
Start by creating a simple maven project in your favorite IDE. Then edit the pom.xml.

pom.xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>rocks.nifi</groupId>
    <artifactId>examples</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>nar</packaging>
    
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>1.7</maven.compiler.source>
        <maven.compiler.target>1.7</maven.compiler.target>
        <nifi.version>1.1.0</nifi.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.apache.nifi</groupId>
            <artifactId>nifi-api</artifactId>
            <version>${nifi.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.nifi</groupId>
            <artifactId>nifi-utils</artifactId>
            <version>${nifi.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.nifi</groupId>
            <artifactId>nifi-processor-utils</artifactId>
            <version>${nifi.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-io</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>com.jayway.jsonpath</groupId>
            <artifactId>json-path</artifactId>
            <version>1.2.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.nifi</groupId>
            <artifactId>nifi-mock</artifactId>
            <version>${nifi.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.nifi</groupId>
                <artifactId>nifi-nar-maven-plugin</artifactId>
                <version>1.0.0-incubating</version>
                <extensions>true</extensions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.15</version>
            </plugin>
        </plugins>
    </build>
</project>
This pom.xml includes a single plug-in for building a nifi nar, which is similar to a war for nifi, that bundles everything up in a way nifi can unpack. The nifi-api is the only other “required” dependency. The other nifi dependencies are really use full as you will see.

The next important piece is telling nifi which classes to load and register. This is done in a single file located at /src/main/resources/META-INF/services/org.apache.nifi.processor.Processor

rocks.nifi.examples.processors.JsonProcessor
The JSON Processor
Now that everything is defined and findable by Apache Nifi, lets build a processor. Define a simple java class as defined in the setup process (rocks.nifi.examples.processors.JsonProcessor).

Tags are useful for finding your processor in the list of processors in the GUI. So in this case in the search box you could just type ‘json’ and your processor will be found. The capability description is also displayed in the processor selection box. Nifi.rocks will make future posts on documenting your custom processors. Finally most processors will just extend the AbstractProcessor, for more complicated tasks it may be required to go a level deeper for the AbstractSessionFactoryProcessor.

JsonProcessor.java: Apache Nifi Processor Header
@SideEffectFree
@Tags({"JSON", "NIFI ROCKS"})
@CapabilityDescription("Fetch value from json path.")
public class JsonProcessor extends AbstractProcessor {
Not really interesting stuff here. Properties will hold all a list of all the available properties tha are exposed to the user. Relationships will hold the relationships the processor will use to direct the flow files. For more details on relationships, properties, and components of an Apache Nifi flow please read the offical developer guide. There is plenty of room to expand on custom validators, but there is a large selection of validators in nifi-processor-utils package.

JsonProcessor.java: Variable Declaration
private List<PropertyDescriptor> properties;
private Set<Relationship> relationships;

public static final String MATCH_ATTR = "match";

public static final PropertyDescriptor JSON_PATH = new PropertyDescriptor.Builder()
        .name("Json Path")
        .required(true)
        .addValidator(StandardValidators.NON_EMPTY_VALIDATOR)
        .build();

public static final Relationship SUCCESS = new Relationship.Builder()
        .name("SUCCESS")
        .description("Succes relationship")
        .build();
The init function is called at the start of Apache Nifi. Remember that this is a highly multi-threaded environment and be careful what you do in this space. This is why both the list of properties and the set of relationships are set with unmodifiable collections. I put the getters for the properties and relationships here as well.

JsonProcessor.java: Apache Nifi Init
@Override
public void init(final ProcessorInitializationContext context){
    List<PropertyDescriptor> properties = new ArrayList<>();
    properties.add(JSON_PATH);
    this.properties = Collections.unmodifiableList(properties);

    Set<Relationship> relationships = new HashSet<>();
    relationships.add(SUCCESS);
    this.relationships = Collections.unmodifiableSet(relationships);
}

@Override
public Set<Relationship> getRelationships(){
    return relationships;
}

@Override
public List<PropertyDescriptor> getSupportedPropertyDescriptors(){
    return properties;
}
The onTrigger method is called when ever a flow file is passed to the processor. For more details on the context and session variables please again refer to the official developer guide.

JsonProcessor.java: Apache Nifi OnTrigger
@Override
public void onTrigger(ProcessContext context, ProcessSession session) throws ProcessException {
    final AtomicReference<String> value = new AtomicReference<>();

    FlowFile flowfile = session.get();

    session.read(flowfile, new InputStreamCallback() {
        @Override
        public void process(InputStream in) throws IOException {
            try{
                String json = IOUtils.toString(in);
                String result = JsonPath.read(json, "$.hello");
                value.set(result);
            }catch(Exception ex){
                ex.printStackTrace();
                getLogger().error("Failed to read json string.");
            }
        }
    });

    // Write the results to an attribute
    String results = value.get();
    if(results != null && !results.isEmpty()){
        flowfile = session.putAttribute(flowfile, "match", results);
    }

    // To write the results back out ot flow file
    flowfile = session.write(flowfile, new OutputStreamCallback() {

        @Override
        public void process(OutputStream out) throws IOException {
            out.write(value.get().getBytes());
        }
    });

    session.transfer(flowfile, SUCCESS);
}
In general you pull the flow file out of session. Read and write to the flow files and add attributes where needed. To work on flow files nifi provides 3 callback interfaces.

InputStreamCallback: For reading the contents of the flow file through a input stream.

Using Apache Commons to read the input stream out to a string. Use JsonPath to attempt to read the json and set a value to the pass on. It would normally be best practice in the case of a exception to pass the original flow file to a Error relation point in the case of an exception.

JsonProcessor.java: Apache Nifi InputStreamCallback
session.read(flowfile, new InputStreamCallback() {
    @Override
    public void process(InputStream in) throws IOException {
        try{
            String json = IOUtils.toString(in);
            String result = JsonPath.read(json, "$.hello");
            value.set(result);
        }catch(Exception ex){
            ex.printStackTrace();
            getLogger().error("Failed to read json string.");
        }
    }
});  
OutputStreamCallback: For writing to a flowfile, this will over write not concatenate.

We simply write out the value we recieved in the InputStreamCallback

JsonProcessor.java: Apache Nifi OutputStreamCallback
flowfile = session.write(flowfile, new OutputStreamCallback() {
    @Override
    public void process(OutputStream out) throws IOException {
        out.write(value.get().getBytes());
    }
});
StreamCallback: This is for both reading and writing to the same flow file. With both the outputstreamcallback and streamcall back remember to assign it back to a flow file. This processor is not in use in the code and could have been. The choice was deliberate to show a way of moving data out of callbacks and back in.
Apache Nifi StreamCallback
flowfile = session.write(flowfile, new OutputStreamCallback() {

    @Override
     public void process(OutputStream out) throws IOException {
        out.write(value.get().getBytes());
    }
});
Flow files can also contain meta data in attributes to push between processors.

JsonProcessor.java: Setting low file attributes
// Write the results to an attribute
String results = value.get();
if(results != null && !results.isEmpty()){
    flowfile = session.putAttribute(flowfile, "match", results);
}
Finally every flow file that is generated needs to be deleted or transfered.

JsonProcessor.java: Session Transfer
session.transfer(flowfile, SUCCESS);
At this point you should be able to build with a simple

mvn clean install
Deployment
Copy the target/examples-1.0-SNAPSHOT.nar to $NIFI_HOME/lib
$NIFI_HOME/bin/nifi.sh stop
$NIFI_HOME/bin/nifi.sh start
After Nifi finishes starting you should be able to add it to your flow.

Nifi.rocks will follow up with how to generate unit tests and documentation for your custom processors soon.
