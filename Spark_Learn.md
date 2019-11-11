# Spark源码 #
2019/10/19 14:28:18  
版本: 2.1.1

## 初识Spark ##
1. spark-shell依赖Scala的Iloop类, 重写了loadFiles()方法  
 ```
class SparkILoop(in0: Option[BufferedReader], out: JPrintWriter) extends ILoop(in0, out)
...
/**
   * We override `loadFiles` because we need to initialize Spark *before* the REPL
   * sees any files, so that the Spark context is visible in those files. This is a bit of a
   * hack, but there isn't another hook available to us at this point.
   */
  override def loadFiles(settings: Settings): Unit = {
    initializeSpark()
    super.loadFiles(settings)
  }
 ```
2. 调用**initializeSpark()**方法, 因此能在shell中使用sc和spark
 ```
  def initializeSpark() {
    intp.beQuietDuring {
      processLine("""
        @transient val spark = if (org.apache.spark.repl.Main.sparkSession != null) {
            org.apache.spark.repl.Main.sparkSession
          } else {
            org.apache.spark.repl.Main.createSparkSession()
          }
        @transient val sc = {
          val _sc = spark.sparkContext
          if (_sc.getConf.getBoolean("spark.ui.reverseProxy", false)) {
            val proxyUrl = _sc.getConf.get("spark.ui.reverseProxyUrl", null)
            if (proxyUrl != null) {
              println(s"Spark Context Web UI is available at ${proxyUrl}/proxy/${_sc.applicationId}")
            } else {
              println(s"Spark Context Web UI is available at Spark Master Public URL")
            }
          } else {
            _sc.uiWebUrl.foreach {
              webUrl => println(s"Spark context Web UI available at ${webUrl}")
            }
          }
          println("Spark context available as 'sc' " +
            s"(master = ${_sc.master}, app id = ${_sc.applicationId}).")
          println("Spark session available as 'spark'.")
          _sc
        }
        """)
      processLine("import org.apache.spark.SparkContext._")
      processLine("import spark.implicits._")
      processLine("import spark.sql")
      processLine("import org.apache.spark.sql.functions._")
      replayCommandStack = Nil // remove above commands from session history.
    }
  }
 ```
3. **WordCount**
 ```
sc.textFile("file:///app/software/spark/README.md")
	.flatMap(_.split(" "))
	.map((_,1))
	.reduceByKey(_+_)
	.sortBy(_._2,false)
	.foreach(println)
 ```

## Spark基础设施 ##
### SparkConf ###
1. 底层实现
 ```
  private val settings = new ConcurrentHashMap[String, String]()
 ```
2. 获取配置的3种方式
 - 系统属性中的配置
 ```
  private[spark] def loadFromSystemProperties(silent: Boolean): SparkConf = {
    // Load any spark.* system properties
    for ((key, value) <- Utils.getSystemProperties if key.startsWith("spark.")) {
      set(key, value, silent)
    }
    this
  }
 ```
 - 使用SparkConf的API
 ```
  /** Set a configuration variable. */
  def set(key: String, value: String): SparkConf = {
    set(key, value, false)
  }
...
  private[spark] def set(key: String, value: String, silent: Boolean): SparkConf = {
    if (key == null) {
      throw new NullPointerException("null key")
    }
    if (value == null) {
      throw new NullPointerException("null value for " + key)
    }
    if (!silent) {
      logDeprecationWarning(key)
    }
    settings.put(key, value)
    this
  }
 ```
 - 克隆SparkConf
 ```
class SparkConf(loadDefaults: Boolean) extends Cloneable
...
  /** Create a SparkConf that loads defaults from system properties and the classpath */
  def this() = this(true)
...
  /** Copy this object */
  override def clone: SparkConf = {
    val cloned = new SparkConf(false)
    settings.entrySet().asScala.foreach { e =>
      cloned.set(e.getKey(), e.getValue(), true)
    }
    cloned
  }
 ```

### Spark内置RPC框架 ###
1. 图解
![](https://i.imgur.com/o9wNQzq.png)
2. 