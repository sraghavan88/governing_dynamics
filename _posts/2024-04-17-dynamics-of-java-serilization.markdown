---
layout: post
title:  "Dynamics of Java Serialization How to Defend from attacks"
date:   2024-04-16 06:21:52 -0400
categories: security
---
### Prologue

Java Serialization  has the great promise of taking the state of full object graph of and save it externally and magically restore its state when we deserialize. This is a big promise as this replaced very error prone state saving custom code which was used prior to Java. It may be single most important reason for Java’s success and it magic.  We would find how magic becomes dangerous. 

### Mechanics

#### Java serialization  ( How it works )

We would take a first look at default Java serialization. Lets take a POJO 

{% highlight java %}
 public static class Range implements Serializable {
        private final int low;
        private final int high;
        
        public Range(int low, int high) {
            if (low > high) {
                throw new IllegalArgumentException("Bad data");
            }
            this.low = low;
            this.high = high;
        }

        public int getLow() {
            return low;
        }


        public int getHigh() {
            return high;
        }
    }
{% endhighlight %}

### Serialization Mechanics 

{% highlight java %}
 final var range = new Range(3, 4);
        try (final var fileOutputStream = new FileOutputStream("output.ser")) {
            final var objectOut = new ObjectOutputStream(fileOutputStream);
            objectOut.writeObject(range);
        }
{% endhighlight %}

When the object is passed objectOut.writeObject it is not going get the values by calling the getter accessors of the pojo. 

Instead it would walk through the object graph reflectively scraps the data from fields directly.

Which means the object cannot control its output form of its internal state. This breaks encapsulation as the code written inside get is not longer used 

This is an extralinguistic behavior as I cannot reason the working of the code by just reading it  

### Deserialization Mechanics( How it works )

When sending data out (Serialization)  one  can be responsible when the object gets constructed the invariance is checked. But while  Deserialization happens it becomes even more of a nightmare because one is consuming data from world where hackers are waiting to take over your system.

{% highlight java %}
  try (final var fileInputStream = new FileInputStream("output.ser")) {
            final var objectIn = new ObjectInputStream(fileInputStream);
            final var range = (Range) objectIn.readObject();
            System.out.println(range.getLow());
        }
{% endhighlight %}

When we read output.ser we not enforcing a checksum or anything. U can happily tamper the output.ser and send  to deserialize it would be happily accept as the input 

When the object is passed objectIn.readObject is not going fill up the value by calling the constructor 

{% highlight java %}
public Range(int low, int high) {
            if (low > high) {
                throw new IllegalArgumentException("Bad data");
            }to 
            this.low = low;
            this.high = high;
        }
{% endhighlight %}

Instead it would call phantom empty constructor creates the object 

The constructor and invariant check would be never performed

This breaks encapsulation as the code written inside get is not longer used .

Again this is an extralinguistic behavior as I cannot reason the working of the code by just reading it  

We have to write more defensive code to make this class work correctly 

{% highlight java %}
  private  void readObject(ObjectInputStream objectInputStream) throws IOException, ClassNotFoundException {
            objectInputStream.defaultReadObject();
            if (low > high) {
                throw new IllegalArgumentException("Bad data");
            }
        }
{% endhighlight %}


One can again this is private method which would called during the objectIn.readObject and would check the invariance. Without this defensive code we cannot make the Range class work as expected

### Malicious Version 

Lets look at the DateHolder  class what we have done in it 

Defensive copies . Creating a new Date instead taking external input 

Invariant check . Creating a new Date from Time and checking its validity 

{% highlight java %}
 public class DateHolder implements Serializable {
  private final Date date;
  public DateHolder(Date d) {
    date = new Date(d.getTime());
    if (isInvalidDate(date)) {
      throw new IllegalArgumentException();
    }
} } 
{% endhighlight %}


But still Malicious sub type can wreck havoc 

{% highlight java %}

class SonOfDate extends Date {
  public long getTime() {
    Runtime.getRuntime().exec("printenv")
    return new Random().nextInt();
  }
}
{% endhighlight %}

Now can introduce by putting this malicious version of the Date as back references  and Deserialization supports polymorphism it would happily read it and we can bingo we have an  RCE attack



### Key TakeAways 

- Java serialization/de-serialization makes use heavy use of reflection to scrape data from Object graphs

- The use of reflection breaks encapsulation and  makes cases for by passing constructors of objects which prevent checks before creating the object

- Java serialization/de-serialization is extralinguistic behavior as one cannot reason the working of the code by just reading it  

- And if one cannot reason the correctness of the code one cannot reason the security aspect of the code 

- Java de-serialization requires phantom methods like readObject to write defensive code to validate the object before we create 

- Java de-serialization supports  polymorphic subtypes which opens the door for malicious subtypes to attack 

- Changing the wire encoding from native serialization to JSON or YML doesn't make it more secure as the internal mechanics of reading and creating objects remain the same 

### Jackson 

Ok native serialization is bad and lets do JSON . Even if switch we the wire enconding the mechanism of getting and filling of object values remain the same as it is a  native Java construct 

Now lets parse the same POJO with Jackson 

{% highlight java %}
    final var range = new Range(3, 4);
        final var objectMapper = new ObjectMapper();
        System.out.println(objectMapper.writeValueAsString(range));
{% endhighlight %}

No complains I get the output 

{% highlight java %}
{"low":3,"high":4}
{% endhighlight %}

Now lets take the same json out and deserialize it 

{% highlight java %}
 final var data = """
                {"low":3,"high":4}
                """;

        final var objectMapper = new ObjectMapper();
        System.out.println(objectMapper.readValue(data,Range.class));
{% endhighlight %}

But this would fail the same POJO which got the serialized cannot be constructed back why 

{% highlight java %}
 com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Cannot construct
instance of `com.veracode.sca.Main$Range` (no Creators, like default construct, exist)
 cannot deserialize from Object value (no delegate- or property-based Creator)at
  [Source: (String)"{"low":3,"high":4}
{% endhighlight %}

In Jackson if u have something else than default constructor no arg  Jackson cannot to the magic reflective instance creation. I have an option now to introduce a default one 

{% highlight java %}
        private  int low;
        private  int high;

        public Range(){

        }
{% endhighlight %}
This makes my variables non final

Or instruct the Jackson what constructor I need to use with @JsonCreator 

{% highlight java %}
 @JsonCreator
        public Range(@JsonProperty("low") int low, @JsonProperty("high") int high) {
            if (low > high) {
                throw new IllegalArgumentException("Bad data");
            }
            this.low = low;
            this.high = high;
        }
{% endhighlight %}

Jackson writes the defensive code for us so we specify the constructor to call making it slightly safer 



### Defend from Java De-Serialization attacks

- Be extra care full with untrusted data from the internet.

- Don't not create complex Objects like Maps in your DTO objects which face the internet that can open the doors for attacks 

- Always do code review of DTOs facing the internet to reason its security aspects 

- Always use final classes as DTOs  and field variables  and disable polymorphic subtype parsing in the parsing library 

- Try using Java Records which restricts things you can do with classes as DTO and it forces parsing libraries to call the constructor 

- Run SCA scanner to find out CVEs and update to the fixed versions

- The fixed versions of  parsing libraries has the defensive code and filters to protect from attacks so don't skip version upgrades