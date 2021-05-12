# Model Registry API design doc

## Motivations

A Redesign of the model registry API to be more object-oriented and similar to hsfs.

## Proposed design:

### Connect

```python

# (Ho)psworks (M)od(e)l (R)egistry
import homer
# Create a connection
connection = homer.connection()
# Get the model registry handle for the project's model registry
mr = connection.get_model_registry()



```

### Create 

Instead of calling model.export, users create a model by invoking the `create_model` function.

```python

model_meta = mr.create_model(name="mnist",
                             version=1,
			     model_path="/dir_with_model"
			     overwrite=False,
			     metrics=None,
			     description=None,
			     await_registration=300,
			     project=None)
			     
# alternatively, call save on this meta object and upload model to hsfs		     
model_meta.save("/dir_with_model")

```

### Get

An existing model can be found by invoking the `get_model` function.

```python

model_meta = mr.get_model(name="mnist",
                          version=1,
			  project=None)
```

### Finding best model version


The best performing version of a model family can be found by invoking the `get_models` function

```python

model_meta_list = mr.get_models(name="mnist",
                                sort_key=None,
				project=None,
				direction=None)
				
best_model_meta = model_meta_list[0]
```



