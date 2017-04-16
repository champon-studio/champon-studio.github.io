---

title: "Java StAX XML parser logic separation example"
author: ferrisky
tags: Java, OOP, OOAD, Design Pattern
 
---
The way to use StAX in more flexiable and smarter.

# Introduction

StAX is the built-in XML parser in Java, it is ease to use, simple, and memory efficient. Following example is the typical way of parsing XML output from SOAP msg from webservice.

```java
public String getDataStateValue(XMLStreamReader xmlStreamReader, String targetedDevice) throws XMLStreamException {
        String deviceStatus = null;
        String stateValue = null;
        String rfZid = null;
        String featureName = null;

        while (xmlStreamReader.hasNext()) {
            int eventType = xmlStreamReader.next();

            switch (eventType) {
                case XMLStreamReader.START_ELEMENT:
                    deviceStatus = xmlStreamReader.getLocalName();
                    break;

                case XMLStreamReader.CHARACTERS:
                    if (deviceStatus.equals(DataConstants.DATA_RF_ZID)) {
                        rfZid = xmlStreamReader.getText();
                    }

                    if (deviceStatus.equals(DataConstants.DATA_FEATURE_NAME) && rfZid.equals(targetedDevice)) {
                        featureName = xmlStreamReader.getText();
                    }

                    if (deviceStatus.equals(DataConstants.DATA_VALUE) && rfZid.equals(targetedDevice) &&
                            featureName.equals(CmsConstants.VALUE_DATA_FEATURE_NAME)) {
                        stateValue = xmlStreamReader.getText();
                    }
                    break;
            }
        }
        return stateValue;
}
```
Furthermore, StAX can be used to collect some specific target, here is another example, and obvious it has the same access pattern as previous.

```java
public List<String> getEventSeverity(XMLStreamReader xmlStreamReader) throws XMLStreamException {
        String deviceStatus = null;
        List<String> listEventSeverity = new ArrayList<>();

        while (xmlStreamReader.hasNext()) {
            int eventType = xmlStreamReader.next();

            switch (eventType) {
                case XMLStreamReader.START_ELEMENT:
                    deviceStatus = xmlStreamReader.getLocalName();
                    break;

                case XMLStreamReader.CHARACTERS:
                    assert deviceStatus != null;
                    if (deviceStatus.equals(DataConstants.DATA_SEVERITY)) {
                        listEventSeverity.add(xmlStreamReader.getText());
                    }

                    break;
            }
        }
        return listEventSeverity;
  }
```
  
As the result, programmer are going to create lots of methods for collecting different targets, and also create methods for different collecting types; so on so forth. With the growth of the programming, the parsing methods/classes will be too complex to be maintained.

# Refactoring
To address this problem, the logic of data identification and collection must separated Therefore, we can focus on management of data collecting algorhtms/logics. To do this, we create abstract data collector class with Java Generics, which is used to encapsulate data collecting algorhtms/logics:

```java
public abstract class XMLDataCollector<T>{
        public abstract T getResult();
        public abstract void collect(String tag, String value);
}
```

And, having an engine to pump XML tags and value to our collector, the engine takes the responsiblity to   pump XML tag/value into data collector, which is the real place to identify and collect the XML data.

```java
public class SimpleStAXEngine{
    public static void pump(XMLStreamReader xmlStreamReader, XMLDataCollector collector) throws XMLStreamException {
        String deviceStatus = null;
        while (xmlStreamReader.hasNext()) {
            int eventType = xmlStreamReader.next();

            switch (eventType) {
                case XMLStreamReader.START_ELEMENT:
                    deviceStatus = xmlStreamReader.getLocalName();
                    break;

                case XMLStreamReader.CHARACTERS:
                    assert deviceStatus != null;
                    collector.collect(deviceStatus,xmlStreamReader.getText());
                    break;
            }
        }
    }
}
```

# Usage
Now, let's start using our simple StAX engine. First, declare the collector by java anonymous class

```java
XMLDataCollector<List<String>> eventSeverityCollector = new XMLDataCollector<List<String>>(){
    List<String> listEventSeverity = new ArrayList<>();
    public List<String> getResult(){
        return listEventSeverity;
    }
    public void collect(String tag, String value){
        if (tag.equals(DataConstants.DATA_SEVERITY)) {
            listEventSeverity.add(value);
        }
    }
};
```
Java Generic was used to declare the type of collecting target, and collecting logic are encapsulted in implemented anonymous object. After that, we can use this collector by calling `pump()`.

```java
SimpleStAXEngine.pump(xmlStreamReader,eventSeverityCollector);
```

After that, the result can be retrived in `getResult()`

```java
List<String> genericResult = eventSeverityCollector.getResult();
```

# Conclusion
By using this method, the logics/algorithms of data collecting were splitted from parsing method, be managed isolated with designed interface, and attachable to other XML parser with the same generalized tag/value interface.

Class inherientance also be benifited from in this case. For example, allow collecting target to be changed and collect it to a list:

```java
public class BasicXMLValueListCollector extends XMLDataCollector<List<String>>{
	List<String> resultList = new ArrayList<>();
	private String tag = null;
	
	public BasicXMLValueCollector(String tag){
		mTag = tag	
	}
	public List<String> getResult(){
		return listEventSeverity;
	}
	
	public void collect(String tag, String value){
		if (tag.equals(mtag)) {
			resultList.add(value);
       }
   }
}
```

In genernal, the solution used here is not a complicated one, it combines Java Generics, Strategy pattern, Delegation template, and anonymous class in Java. Actually, the solution could be better, but the real-world of softare system is always a trade-off between project resources and design materials. This solution is good enough for me, and old style methods can be easy converted to new one.
<center>
Little effort, huge gains
</center>
