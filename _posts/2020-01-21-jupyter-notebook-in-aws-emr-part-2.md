---
layout: post
title: "Jupyter Notebook in AWS EMR: Part 2"
tags: [ analytics, aws, spark ]
categories: analytics
published: true
---

Configure the Spark Session properly is essential as it is a key to be able to fully utilize EMR Notebook functionality. Here, I will share some possible configuration and also use case when using EMR Notebook for EDA or testing our ETL script .

<!--more-->
### Configure Spark from Jupyter Notebook
Using Apache Livy, we can configure the spark session from Jupyter without restarting the EMR cluster, including the Python version.
The possible field for %%configure magic can be found here. Below is the example script for configuring spark:

#### Default configuration
<pre><code class="language-py">%%configure -f
{
    "conf": {
        "spark.pyspark.python": "python3",
        "spark.pyspark.virtualenv.enabled": "true",
        "spark.pyspark.virtualenv.type": "native",
        "spark.pyspark.virtualenv.bin.path": "/usr/bin/virtualenv"
    }
}</code></pre>
#### Modify Python version

In EMR notebook, the default Python version is Python3. If we already install some package during EMR Bootstrapping in Python2,
we can configure the Jupyter Notebook to use it by adding the following conf field and execute the %%configure magic.
<pre><code class="language-py">%%configure -f
{
    "conf": {
        "spark.pyspark.python": "python",
        "spark.pyspark.virtualenv.enabled": "true",
        "spark.pyspark.virtualenv.type": "native",
        "spark.pyspark.virtualenv.bin.path": "/usr/bin/virtualenv"
    }
}</code></pre>

#### Modify driver and executor
<pre><code class="language-py">%%configure -f
{
    "executorMemory": "36G",
    "executorCores": 5,
    "driverMemory": "36G",
    "driverCores": 5,
    "numExecutors": 11,
    "conf": {
        "spark.pyspark.python": "python",
        "spark.pyspark.virtualenv.enabled": "true",
        "spark.pyspark.virtualenv.type": "native",
        "spark.pyspark.virtualenv.bin.path": "/usr/bin/virtualenv"
    }
}</code></pre>

If you decide to modify the number of slave instances, you should also modify the "numExecutors" in the script above to (#slave_instances x 3) - 1.

#### Use external (Maven) package
We can configure the spark session to import some useful packages from the Maven.
First find the Maven repository of the package that we want to import, in this case, Redis client for Java, jedis.
For this example, you may need to select different versions for the appropriate Scala or Spark version in your cluster.

Then, concatenate the three values highlighted above, separated by a colon, and use it in the %%configure magic like this.
<pre><code class="language-py">%%configure -f
{
    "conf": {
        "spark.pyspark.python": "python",
        "spark.pyspark.virtualenv.enabled": "true",
        "spark.pyspark.virtualenv.type": "native",
        "spark.pyspark.virtualenv.bin.path": "/usr/bin/virtualenv",
        "spark.jars.packages": "redis.clients:jedis:3.2.0"
    }
}</code></pre>

### Install Python Package
We can install additional Python packages from the Notebook without updating the bootstrapping script and restart the EMR cluster.
Once we initialize our Pyspark session, we can check the installed Python package using the following script in the Jupyter Notebook cell.
<pre><code class="language-py">sc.list_packages()</code></pre>
It will return the installed packages in the current spark session, similar to the following text.



You can also install Python packages, such as pandas and matplotlib using the following commands.
<pre><code class="language-py">sc.install_pypi_package("pandas==0.25.1") #Install pandas version 0.25.1 
sc.install_pypi_package("matplotlib", "https://pypi.org/simple") #Install matplotlib from given PyPI repository
sc.install_pypi_package("seaborn") #Install latest version of seaborn</code></pre>

### Data Visualization

We can visualize the data frame we created in Pyspark by converting it into Pandas data frame using toPandas() method.
For that, we need to configure the spark.driver.maxResultSize into unlimited as the default is only 1GB.

<pre><code class="language-py">%%configure -f
{
    "executorMemory": "36G",
    "executorCores": 5,
    "driverMemory": "36G",
    "driverCores": 5,
    "numExecutors": 5,
    "conf": {
        "spark.pyspark.python": "python",
        "spark.pyspark.virtualenv.enabled": "true",
        "spark.pyspark.virtualenv.type": "native",
        "spark.pyspark.virtualenv.bin.path": "/usr/bin/virtualenv",
        "spark.driver.maxResultSize": "0"
    }
}</code></pre>

Let say we have Pyspark data frame, df, as follows:

We can visualize the histogram of tenure and login frequency using seaborn using the following script:
<pre><code class="language-py">import seaborn as sns
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np

pd_df = df.toPandas()
ax = sns.pairplot(data=pd_df[['login_freq', 'log_tenure_in_months', 'downgraded']], hue="downgraded")
</code></pre>
Then, execute this to show the plot
<pre><code class="language-py">%matplot plt</code></pre>