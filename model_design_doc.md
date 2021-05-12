# Feature Store API design doc

## Motivations

From a user prospective we currently have a rather confusing API to interact with the feature store, especially when it comes to joining feature groups together. Moreover, the behavior differs between the 3 different clients we maintain.

## Proposed design:

### Connect

```python
import hopsworks

connection = hopsworks.connection(host="my.hopsworks", project="project_name")

# There are optional arguments and arguments can also be set directly
connection = hopsworks.connection(host="my.hopsworks",
                                  project="project_name",
                                  secrets_store="parameterstore")
connection.hostname_verification = True

# You can get shared feature stores by passing the name. Defaulting to the project feature store
fs = connection.get_feature_store(name=None)

# The feature store can be closed for cleanup explicitly or with a with statement. Not required.
connection.close()

with hopsworks.connection(host="my.hopsworks", project="project_name") as connection:
	fs = connection.get_feature_store()
	print(fs.featuregroups())

```

### Create 

At the moment we have several methods to create a feature group:
-  `create_featuregroup`: It creates online and offline feature groups. Some of the parameters are only valid in the online case (eg. `online_types`) and some make sense only in the offline case (eg. `partition_by`).
- `create_on_demand_featuregroup`: Creates on demand feature. No statistics available here - it is reasonable to have a job that computes those statistics every X amount of time.
There is also an issue with the schema of on demand feature groups, the schema is based on parsing (REGEX) the SQL query, which is not very reliable. We need to store the schema in Hopsworks. 
- `import_featuregroup_s3`: Overlapping parameters as for the `create_featuregroup` however in this case we need to specify the connector and the path within the bucket.
- `import_featuregroup_redshift`: Almost the same as for the `create_featuregroup` however in this case we need to specify the connector and the SQL query to read the data from redshift. 

```python
# Offline feature store
fg0 = fs.create_feature_group(name=name,
                          description=description,
                          version=version,
                          online_enabled=online_enabled,
                          statistics_conf=None)
fg0.schema = schema
fg0.primary_key = []
# A partition key can be specified, if specified the dataframe will have to be reordered to have partition
# columns in the end. Inserts will compare schemas and reorder columns if necessary accordingly.
fg1.partition_by = []

fg0.save(dataframe)

# You can update the the configuration of the feature group
fg0.statistics_conf = stat  # Set statistics configuration for the specific feature group
fg0.update()

# Drop the feature group
fg0.drop()

# On-demand feature store
fg2 = fs.create_ondemand_feature_group(,
                          name=name,
                          description=description,
                          version=version)
# fg2.query = query
fg2.save(query)

# Retrieve metadata about the feature store
fg0.partition_keys # Return partition keys for the specific feature group
fg0.statistics # Return computed statistics object for the specific feature group
fg0.statistics_conf # Get statistics configuration for the specific feature group

```

### Amend schema

We should allow non breaking changes to the schema (add a new column) without having to create a new feature group.

```python 
fg0 = fs.feature_group(name, version)
fg0.schema.add("ft100", type, default_value) # Default value is enforced for schema compatibility
```

### Insert

We already have a JIRA to have a "global" statistics configuration for a feature group, meaning that the settings is specificed during the creation and or modified explicitly (see above) 

The insert should be then delegated to only append or overwrite data in the feature group. The statistics will be recomputed only if they were enabled at creation time.

Currently we support inserting only though a dataframe. We can't do appends from external sources (s3). The new API design should address that.
Other use cases it would be nice to support:
- Write directly though Hive (e.g. Assuming you want to save a Pandas dataframe from SageMaker notebook)
- Write external tables using HopsFS connector form another Hadoop cluster.

```python
fg = fs.get_feature_group(name, version)

fg.insert(df)
fg.upsert(df)
fg.overwrite(df)

# A connector can be passed instead of a data frame
fg.insert(connector)
fg.upsert(connector)
fg.overwrite(connector)
```

### Statistics

We should have a way of triggering the recompute of the statistics separately from the insertion into the feature store. The use cases are the following:

- Enable statistics after the feature group has been created
- Compute statistics for "On-demand feature groups" on a timely basis
- Compute statistics for clients that write directly into the feature store (HopsFS client)

```python
fg1 = fs.feature_group(name, version)
fg1.compute_statistics()
fg1.visualize_featuregroup_correlations()
```

### Explore

Pandas style API.  

```python 
fg = fs.get_featuregroup(name, version)
fg.read() # Returns a dataframe object

fg.head(10) # Show a sample of 10 rows in the feature group (all features)

fg.select_all() # Select all the features in the feature group

fg.select(["ft1", "ft2", "ft3"]).head(10) # Show a sample of 10 rows, selected features, in the feature group

fg.select(["ft1", "ft2", "ft3"]).read() # Returns a dataframe object 
fg.select(["ft1", "ft2", "ft3"]).head(10)
```

Retrieve feature groups by time-travel timestamps:
```python
fs.get_feature_group("fg_name", fg_version, commit_time=None, commit_begin=None, commit_end=None)
# or (tbd)
fs.get_feature_group("fg_name", fg_version, wallclock_time=None,
                     wallclock_time_start=None,
                     wallclock_time_end=None)
```

Example drop highly correlated features.

```python
fg = fs.get_feature_group(name, version)

feature_correlation_statistics = fg.statistics.feature_correlation
feature_correlation_statistics["ft1"]["ft2"]

features = fg.statistics.not_correlated(threshold = 0.5)
fg.features(features).head(10)
```

How do we implement the filter? Fine in PySpark, but what about SageMaker?    
Is sample implemented in the query itself (meaning add `limit x`) or should we get the entire dataframe and then call the sample()? In the SageMaker case this might require transferring tons of data for nothing.

#### Filter

We want to provide an API to allow users to filter Feature Groups and Queries.

Applying a filter to a feature group will return a Query with all features selected and the applied filter. Similarly filters can be applied to any Query object.

In Python the implementation will be based on overloading the bitwise operators `&, |, ~` as well as logistic operators `==, >, <, >=, <=, !=` on the `Feature`-object. It will not be possible to use Python's binary `and`, `or` and `not` operators since these are short-circuited and it is not possible to overload them in Python.

Applying one of the logistic operators to a `Feature` returns a `Filter` object, containing the reference to the filtered feature, the value and the condition. The bitwise operators `&, |, ~` can then be used on these filter objects to construct conjuctions of filters.

In the future, this API can be extended by SQL functions as normal object methods, for example `Feature.like()`.

```python
fg = fs.get_featuregroup(name, version)

fg.filter(Feature("a") >= 10)
# returns Query with applied filter

fg.select(["abc", "def"]).filter(Feature("abc") >= 10 & Feature("def") <= 5)
```

This API is inspired by PySpark's filter API, which uses `col` instead of `Feature`: `col("a") > 10`

Filtering on a Query with joins. Here are a few cases concerned with identifying the origin of a feature.

1. Filter is applied after select: just take the feature group as feature origin on which the select and filter is applied (essentially the left_featuregroup in our Query representation of the Query the filter is applied on):
```python
joined_features = fg1.select(["ft1", "ft2", "ft3"]).filter(Feature("ft2") > 10)
                     .join(fg2.select(["ft1", "ff4", "ff3", "fz1"]), on=["ft1"], how="left")
```

2. Filter applied after join: in this case we have three possibilitie: (i) is to try and infer (in the backend) where the feature comes from, (ii) make the user input the feature group as argument to `Feature` or (iii) make an assumption and always assume it comes from left outer most, however this prevents the use of conjuctions as shown in case 4..
```python
joined_features = fg1.select(["ft1", "ft2", "ft3"])
                     .join(fg2.select(["ft1", "ff4", "ff3", "fz1"]), on=["ft1"], how="left")
                     .filter(Feature("ft2") > 10)
```

3. User wants to filter both feature groups: filtering on the two select seperately is effectively like using an `&` filter conjunction on the joined result, however, we know where the features come from.
More difficult is the case where the conjunction is applied as a single filter at the end, since we don't know the origin of the features, again we have the three possibilities like in case 2.
```python
# effectively a &-conjunction
joined_features = fg1.select(["ft1", "ft2", "ft3"]).filter(Feature("ft2") < 5)
                     .join(fg2.select(["ft1", "ff4", "ff3", "fz1"])
                              .filter(Feature("ff3") > 10),
                          on=["ft1"],
                          how="left")

# requires feature origin inferral like in case 2
joined_features = fg1.select(["ft1", "ft2", "ft3"])
                     .join(fg2.select(["ft1", "ff4", "ff3", "fz1"]),
                          on=["ft1"],
                          how="left")
                     .filter(Feature("ft2") < 5 & Feature("ff3") > 10)
```
Note: in this case we will rely on Spark to optimize and push the filter before the join, because our query constructor will translate it to:

```sql
   select fg1.ft1, fg1.ft2, fg1.ft3, fg2.ft1, fg2.ff4, fg2.ff3, fg2.fz1
     from fg1
left join fg2
       on fg1.ft1 = fg2.ft1
    where fg1.ft2 < 5 and fg2.ff3 > 10
```

4. A `or`-condition: similar to case 2, we have the three mentioned possibility to infer origin. This filter will have to be applied at the end after all joins.
```python
joined_features = fg1.select(["ft1", "ft2", "ft3"])
                     .join(fg2.select(["ft1", "ff4", "ff3", "fz1"]),
                          on=["ft1"],
                          how="left")
                     .filter(Feature("ft1") > 10 | Feature("ff4") < 10)
```

Ideally we would like to make use of Pythons dynamic attribute setting and set all features as attributes of the feature group, which would then allow the following simplification:

```python
fg.filter(fg.f1 >= 10)
```

Allowing this usage would simplify inferring the origin of a feature. However, this will not be possible in the Java API.


### Join

As before, Pandas style API. I'd keep it simple, either you pass a list of features to be join on or we join on the primary keys (if they match in number and types) 

No more flat namespace. Too much metadata to fetch in the backend and too many corner cases to address in case of duplicated names. A feature is addressable by (feature group name, version, feature name).

Always enforce the version.

if you specify `on` the column should be present on both feature groups and it should have the same type.
if you specify `left_on` then you should also specify `right_on`. The 2 lists should have the same length and they should correspond to features present in the respective feature groups. Types should also match.


```python 
fg1 = fs.get_feature_group(name, version)
fg2 = fs.get_feature_group(name, version)
fg3 = fs.get_feature_group(name, version)

joined_features = fg1.select(["ft1", "ft2", "ft3"])
                     .join(fg2.select(["ft1", "ff4", "ff3", "fz1"]), on=["f1"], how="left")
                     .join(fg3.select(["fz1", "fz3"]), on=["f1", "fz1"], how="inner")
                     .filter(lambda row : row["ft1"] != "Fabio") # Under feasibility studies
joined_features.read()
joined_features.head(10)
```

We should still allow for the SQL option as it's handy for power users. `fs.sql` is just a wrapper that allows us to abstract the behavior beo
```python
fs.sql("SQL query to be fed into Hive/SparkSQL")
```

### Create training dataset

```python
td = fs.create_training_dataset(name=name,
                    version=version,
                    data_format=data_format,
                    data_format_args=None,
                    storage_connector=None,
                    statistics_conf=None)

# Pass directly a query object
td.save(query, write_options)

# Allow users to insert directly dataframes
td.save(dataframe)
```

We should also provide a way of splitting the dataset in Training/Validation

```python
connector = fs.connectors['S3']
connector.path = 'my/folder'

td = fs.create_training_dataset(name=name,
                    version=version,
                    features=joined_features,
                    data_format=data_format,
                    storage_connector=connector,
                    split=[0.8, 0.2])

# Pass directly a query object
td.save(query)

# Allow users to insert directly dataframes
td.save(dataframe)
```

You should be able to append/overwrite a training dataset, as long as the schema matches the original metadata. No schema modifications allowed.


```python
td.insert(query, overwrite=False)
td.insert(dataframe, overwrite=False)
```

### Get a training dataset

```python
td = fs.get_training_dataset("td_name", version)
path = fs.get_training_dataset_path("td_name", version, split)
```