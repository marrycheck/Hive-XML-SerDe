原始使用文档链接：[https://github.com/dvasilen/Hive-XML-SerDe/wiki/XML-data-sources](https://github.com/dvasilen/Hive-XML-SerDe/wiki/XML-data-sources)

# XML data sources

Cory Taylor edited this page on 5 Aug 2019 · [34 revisions](https://github.com/dvasilen/Hive-XML-SerDe/wiki/XML-data-sources/_history)

## Installation

Download the latest version of the [hivexmlserde.jar](http://search.maven.org/#search|ga|1|hivexmlserde) and copy it to HIVE_HOME/auxlib or add to hive session with command `hive --auxpath LIB_PATH/hivexmlserde-x.x.x.x.jar.`. Alternatively you can add it to your hive session with the following hive commands.

```
add jar localpath/hivexmlserde-x.x.x.x.jar;
```

**Apache Maven dependency**

```
<dependency>
    <groupId>com.ibm.spss.hive.serde2.xml</groupId>
    <artifactId>hivexmlserde</artifactId>
    <version>x.x.x.x</version>
</dependency>
```

The latest and previous versions are listed at [search.maven.org](http://search.maven.org/#search|ga|1|a%3A"hivexmlserde").

## Usage

The XML SerDe allows the user to map the XML schema to Hive data types through the Hive Data Definition Language (DDL), according to the following rules.

```
CREATE [EXTERNAL] TABLE <table_name> (<column_specifications>)
ROW FORMAT SERDE 'com.ibm.spss.hive.serde2.xml.XmlSerDe'
WITH SERDEPROPERTIES (
["xml.processor.class"="<xml_processor_class_name>",]
"column.xpath.<column_name>"="<xpath_query>",
... 
["xml.map.specification.<xml_element_name>"="<map_specification>"
...
]
)
STORED AS
INPUTFORMAT 'com.ibm.spss.hive.serde2.xml.XmlInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.IgnoreKeyTextOutputFormat'
[LOCATION "<data_location>"]
TBLPROPERTIES (
"xmlinput.start"="<start_tag ",
"xmlinput.end"="<end_tag>"
);
```

For example, the following XML...

```
<records>
	<record customer_id="0000-JTALA">
		<income>200000</income>		
		<demographics>
			<gender>F</gender>
			<agecat>1</agecat>
			<edcat>1</edcat>
			<jobcat>2</jobcat>
			<empcat>2</empcat>
			<retire>0</retire>
			<jobsat>1</jobsat>
			<marital>1</marital>
			<spousedcat>1</spousedcat>
			<residecat>4</residecat>
			<homeown>0</homeown>
			<hometype>2</hometype>
			<addresscat>2</addresscat>
		</demographics>
		<financial>
			<income>18</income>
			<creddebt>1.003392</creddebt>
			<othdebt>2.740608</othdebt>
			<default>0</default>
		</financial>
	</record>
	<record>
	.....
	</record>
</records>
```

...would be represented by the following Hive DDL.

```
CREATE TABLE xml_bank(customer_id STRING, income BIGINT, demographics map<string,string>, financial map<string,string>)
ROW FORMAT SERDE 'com.ibm.spss.hive.serde2.xml.XmlSerDe'
WITH SERDEPROPERTIES (
"column.xpath.customer_id"="/record/@customer_id",
"column.xpath.income"="/record/income/text()",
"column.xpath.demographics"="/record/demographics/*",
"column.xpath.financial"="/record/financial/*"
)
STORED AS
INPUTFORMAT 'com.ibm.spss.hive.serde2.xml.XmlInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.IgnoreKeyTextOutputFormat'
TBLPROPERTIES (
"xmlinput.start"="<record customer",
"xmlinput.end"="</record>"
);

select customer_id, income, demographics, financial from xml_bank limit 1; 
```

The result set of the query is `customer_id:string, income:string, demographics:map<string,string>, financial:map<string,string>`. For clarity, we have split the result of a single row over three lines.

```
"0000-JTALA"
"200000"
{"gender":"F", "agecat":"1", "edcat":"1", "jobcat":"2", "empcat":"2", "retire":"0", "jobsat":"1", "marital":"1", "spousedcat":"1", "residecat":"4", "homeown":"0", "hometype":"2", "addresscat":"2"}
{"income": "18", "creddebt": "1.003392", "othdebt": "2.740608", "default": "0"}
```

## Limitations

- The entire XML file should be a single line (i.e. no newlines in the XML). (A simple unix command to strip newlines is `tr '\n\r' ' ' < source.xml > processed.xml`.) [Dmitry Vasilenko: this is not required, The XML does not need to be formatted as a single line.]
- The XML is split into records using the delimiting text in `xmlinput.start` and `xmlinput.end` properties. Anything outside these tags is ignored. The implementation looks for the byte sequences for the start and end. The approach is similar to [Mahout's XMLInputFormat](https://github.com/apache/mahout/blob/ad84344e4055b1e6adff5779339a33fa29e1265d/examples/src/main/java/org/apache/mahout/classifier/bayes/XmlInputFormat.java)
- Only the XPath 1.0 specification is currently supported. The local part of the qualified names for the elements and attributes are used when handling Hive field names. The namespace prefixes are ignored.
- The field of String type in table DDL does not support the data type of array.Otherwise, data in ```<string>aa</string>```format will appear
- Fields of String type do not support data with a parent in the current hierarchy.



## XML to Hive Data Types Mapping

The data modeled in XML can be transformed to the Hive data types using the conventions documented below. If you are familiar with Java code, you can also examine the test cases in the built in [Hive XML Serde test scripts](https://github.com/dvasilen/Hive-XML-SerDe/blob/master/src/test/java/com/ibm/spss/hive/serde2/xml/processor/java/JavaXPathProcessorTest.java). Some of the more common use cases are documented here.

### 1. Primitive types

The XML element can be directly mapped to an XPath expression.

#### 1.1) Attribute

- **Hive DDL**

```
customer_id string,
....
....
"column.xpath.customer_id"="/record/@customer_id",
```

- **XML data**

```
<record customer_id="ID_00_CUSTOMER">03.06.2009</result>
```

- **Result**

```
select customer_id from xml_bank;

ID_00_CUSTOMER
```

#### 1.2) Content of an XML tag

- **Hive DDL**

```
income BIGINT,
....
....
"column.xpath.income"="/record/income/text()",
```

- **XML data**

```
<record customer_id="ID_00_CUSTOMER"><income>200000</income></result>
```

- **Result**

```
select income from xml_bank;

200000
```

### 2. Structures

The XML element can be directly mapped to the Hive structure type so that all the attributes become the data members. The content of the element becomes an additional member of primitive or complex type.

- **Hive DDL**

```
result STRUCT<name:STRING,result:STRING>,
....
....
"column.xpath.result"="/record/result",
```

- **XML data**

```
<record><result name="ID_DATUM">03.06.2009</result></record>
```

- **Hive DDL and raw data**

```
struct<name:string,result:string>
{"name":"ID_DATUM", "result":"0.3.06.2009"}
```

### 3. Arrays

The XML sequences of elements can be represented as Hive arrays of primitive or complex type. The following example shows how the user can define an array of strings using content of the XML `<result>` element.

- **Hive DDL**

```
result ARRAY<STRING>,
....
....
"column.xpath.result"="/record/result",
```

- **XML data**

```
<record>
<result>03.06.2009</result>
<result>03.06.2010</result>
<result>03.06.2011</result>
</record>
```

- **Hive DDL and raw data**

```
result array<string>    

["03.06.2009","03.06.2010",...]
```

### 4. Maps

The XML schema does not provide native support for maps. There are three common approaches to modeling maps in XML. To accommodate the different approaches we use the following syntax:

```
"xml.map.specification.<element_name>"="<key>-><value>"
```

where

- element_name - The name of the XML element to be considered as a map entry
- key - The map entry key XML node
- value - The map entry value XML node The map specification for the given XML element should be defined under the SERDEPROPERTIES section in the Hive table creation DDL. The keys and values can be defined using the following syntax:
- @attribute - The @attribute specification allows the user to use the value of the attribute as a key or value of the map.
- element - The element name can be used as a key or value.
- \#content - The content of the element can be used as a key or value. As the map keys can only be of primitive type the complex content will be converted to string.

The approaches to representing maps in XML, and their corresponding Hive DDL and raw data, is as follows.

#### 4.1) Element name to content

The name of the element is used as a key and the content as a value. This is one of the common techniques and is used by default when mapping XML to Hive map types. The obvious limitation with this approach is that the map key can be only of type string.

- **Hive DDL**

```
result MAP<STRING,STRING>,
....
"column.xpath.result"="/record/result",
```

- **XML data**

```
<record>
<result>
<entry1>value1</entry1>
<entry2>value2</entry2>
<entry3>value3</entry3>
</result>
</record>
```

- **Mapping, Hive DDL, and raw data**

In this case you do not need to specify a mapping because the name of the element is used as a key and the content as a value by default.

```
result map<string,string>
{"entry1": "value1", "entry2": "value2", "entry3": "value3"}
```

#### 4.2) Attribute to Element Content

Use an attribute value as a key and the element content as a value.

- **Hive DDL**

```
result MAP<STRING,STRING>,
....
"column.xpath.result"="/record/result",
"xml.map.specification.entry"="@name->#content"
```

- **XML data**

```
<record>
 <result>
  <entry name=”key1”>value1</entry>
  <entry name=”key2”>value2</entry>
  <entry name=”key3”>value3</entry>
 </result>
</record>
```

- **Mapping, Hive DDL, and raw data**

```
"xml.map.specification.entry"="@name->#content"
result map<string,string>

{"key1": "value1", "key2": "value2", "key3": "value3"}
```

#### 4.3) Attribute to Attribute

When the field name and the field values are available as attributes, use this syntax to extract the content.

- **Hive DDL**

```
result MAP<STRING,STRING>,
....
"column.xpath.result"="/record/result",
"xml.map.specification.entry"="@name->@value"
```

- **XML data**

```
<record>
 <result>
  <entry name=”key1” value=”value1”/>
  <entry name=”key2” value=”value2”/>
  <entry name=”key3” value=”value3”/>
 </result>
</record>
```

#### 4.4) Attribute to XML element name

When the field name is available as an attribute and you want to extract the element name.

- **Hive DDL**

```
result MAP<STRING,STRING>,
....
"column.xpath.result"="/record/result",
"xml.map.specification.entry1"="@name->entry1"
"xml.map.specification.entry2"="@name->entry2"
"xml.map.specification.entry3"="@name->entry3"
```

- **XML data**

```
<record>
 <result>
  <entry1 name=”key1”/>
  <entry2 name=”key2”/>
  <entry3 name=”key3”/>
 </result>
</record>
```

- **Mapping, Hive DDL, and raw data**

```
result map<string,string>
{"key1": "entry1", "key2": "entry2", "key3": "entry3"}
```

### 5. Complex Content

Complex content being used as a primitive type will be converted to a valid XML string by adding a root element called `<string>`. Consider the following XML:

- **Hive DDL**

```
rawxml STRING,
....
"column.xpath.rawxml"="/record/dataset/*",
```

- **XML**

```
<record>
  <dataset> <value>10</value> <value>20</value> <value>30</value> </dataset>
</record>
```

The XPath expression `/dataset/*` will result in a number of `<value>` XML nodes being returned. If the target field is of primitive type the implementation will transform the result of the query to the valid XML by adding the `<string>` root node.

```
<string> <value>10</value> <value>20</value> <value>30</value> </string>
```

*Note: The implementation will not add a root element If the result of the query is a single XML element.*

### 5. Text Content

The whitespace-only text content of an XML element is ignored.

### 6. References

[Dmitry Vasilenko, Mahesh Kurapati, Efficient Processing of XML Documents in Hadoop Map Reduce. International Journal on Computer Science and Engineering, Vol. 6, Issue 09, p. 329-333](http://www.enggjournals.com/ijcse/doc/IJCSE14-06-09-012.pdf)
