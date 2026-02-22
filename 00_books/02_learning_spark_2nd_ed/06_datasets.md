
## Chapter 6: Datasets

### Creating Datasets with Case Classes (Scala)

```scala
// Define case class
case class Bloggers(
  id: Int, 
  first: String, 
  last: String, 
  url: String, 
  date: String, 
  hits: Int, 
  campaigns: Array[String]
)

// Read and convert to Dataset
val bloggersDS = spark.read
  .format("json")
  .load("/path/to/bloggers.json")
  .as[Bloggers]
```

### Creating Datasets with JavaBeans (Java)

```java
import org.apache.spark.sql.Encoders;
import java.io.Serializable;

public class Bloggers implements Serializable {
    private int id;
    private String first;
    private String last;
    private String url;
    private String date;
    private int hits;
    private String[] campaigns;
    
    // Getters and setters...
    
    public int getId() { return id; }
    public void setId(int id) { this.id = id; }
    // ...
}

// Create Encoder
Encoder<Bloggers> bloggerEncoder = Encoders.bean(Bloggers.class);

// Create Dataset
Dataset<Bloggers> bloggersDS = spark.read()
    .format("json")
    .load("/path/to/bloggers.json")
    .as(bloggerEncoder);
```

### Generating Sample Data

```scala
import scala.util.Random

// Case class
case class Usage(uid: Int, name: String, usage: Int)

// Generate data
val r = new scala.util.Random(42)
val data = for (i <- 0 to 1000) yield 
  Usage(i, "user-" + r.alphanumeric.take(5).mkString(""), r.nextInt(1000))

// Create Dataset
val dsUsage = spark.createDataset(data)
dsUsage.show(10)
```

### Dataset Operations

**filter() with lambda:**
```scala
// Using lambda
dsUsage.filter(d => d.usage > 900).orderBy(desc("usage")).show(5)

// Using function
def filterWithUsage(u: Usage) = u.usage > 900
dsUsage.filter(filterWithUsage(_)).orderBy(desc("usage")).show(5)
```

**map() with lambda:**
```scala
// Simple map
dsUsage.map(u => { if (u.usage > 750) u.usage * 0.15 else u.usage * 0.50 })
       .show(5)

// Map with function
def computeCostUsage(usage: Int): Double = {
  if (usage > 750) usage * 0.15 else usage * 0.50
}
dsUsage.map(u => computeCostUsage(u.usage)).show(5)
```

**Returning new objects:**
```scala
// New case class with cost
case class UsageCost(uid: Int, uname: String, usage: Int, cost: Double)

def computeUserCostUsage(u: Usage): UsageCost = {
  val v = if (u.usage > 750) u.usage * 0.15 else u.usage * 0.50
  UsageCost(u.uid, u.name, u.usage, v)
}

dsUsage.map(u => computeUserCostUsage(u)).show(5)
```

### Converting DataFrame to Dataset

```scala
// Read as DataFrame
val bloggersDF = spark.read
  .format("json")
  .load("/path/to/bloggers.json")

// Convert to Dataset
val bloggersDS = bloggersDF.as[Bloggers]
```

### Encoders and Memory Management

**Spark's Internal Format vs. Java Objects:**

```
Java Object (heap):
┌──────────────────┐
│ Object Header    │  16-24 bytes
│ hashcode         │
│ GC info          │
├──────────────────┤
│ id (int)         │  4 bytes
│ first (String)   │  48 bytes ("Jules")
│ last (String)    │  48 bytes ("Damji")
│ ...              │
└──────────────────┘
Total: ~200+ bytes

Spark Tungsten Format (off-heap):
┌──────────────────┐
│ id: 4 bytes      │
│ first offset     │
│ last offset      │
│ data ...         │
└──────────────────┘
Contiguous memory, pointer-based
```

### Costs of Using Datasets

**Inefficient chaining:**
```scala
// Each lambda → DSL transition causes serialization/deserialization
personDS
  .filter(x => x.birthDate.split("-")(0).toInt > earliestYear)  // lambda
  .filter($"salary" > 80000)                                    // DSL
  .filter(x => x.lastName.startsWith("J"))                      // lambda
  .filter($"firstName".startsWith("D"))                         // DSL
  .count()
```

**Efficient chaining:**
```scala
// All DSL - no serialization overhead
personDS
  .filter(year($"birthDate") > earliestYear)
  .filter($"salary" > 80000)
  .filter($"lastName".startsWith("J"))
  .filter($"firstName".startsWith("D"))
  .count()
```

---
