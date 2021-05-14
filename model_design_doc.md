# Model Registry API design doc

## Motivations

A Redesign of the model registry API to be more object-oriented and similar to hsfs.

## Proposed design:

### Connect

```python

# (H)pswork(s) (M)odel (R)egistry
import hsmr
# Create a connection
connection = hsmr.connection()
# Get the model registry handle for a project's model registry
mr = connection.get_model_registry()



```

### Model attributes

  String id;

  String name;

  int version;
  
  String projectName;

  String userFullName;

  Long created;
  
  Signature signature;

  ModelResult metrics;

  String description;

  String[] environment;

  String program;
  
  String experimentId;
  
  String experimentProjectName;

  String kernelId;

  String jobName;

### Create 

Instead of calling `model.export`, users create a model by invoking the `create_model` function.

```python

model_meta = mr.create_model(name="mnist",
                             version=1,
			     signature=mr.infer(train_df_minus_target, model.predict(train_df_minus_target)),
			     overwrite=False,
			     metrics=None,
			     description=None,
			     await_registration=300)
			     
			    	     
model_meta.save("/dir_with_model")

```

Note - currently in the registry the most crucial feature we are missing is the input shape for the training dataset, and the shape of the predictions.
We should add that in the saving of a model and show it in the UI.

mlflow provides an *infer* function

https://www.mlflow.org/docs/latest/python_api/mlflow.models.html#mlflow.models.ModelSignature


![Screenshot from 2021-05-14 10-24-04](https://user-images.githubusercontent.com/9936580/118243298-930f0680-b49e-11eb-9d05-dfd86bf4bd7e.png)


As a reference, when a model is saved in mlflow, a great deal of informaton is saved to describe the model depending on the machine learning library. For example for TensorFlow.

https://www.mlflow.org/docs/latest/python_api/mlflow.tensorflow.html#mlflow.tensorflow.save_model

*tf_meta_graph_tags* – A list of tags identifying the model’s metagraph within the serialized SavedModel object. For more information, see the tags parameter of the tf.saved_model.builder.savedmodelbuilder method.

*signature* – (Experimental) ModelSignature describes model input and output Schema. The model signature can be inferred from datasets with valid model input (e.g. the training dataset with target column omitted) and valid model output (e.g. model predictions generated on the training dataset).

*input_example* – (Experimental) Input example provides one or several instances of valid model input. The example can be used as a hint of what data to feed the model. The given example can be a Pandas DataFrame where the given example will be serialized to json using the Pandas split-oriented format, or a numpy array where the example will be serialized to json by converting it to a list. Bytes are base64-encoded.

Keep in mind those are specific to TensorFlow. There is also support for a large amount of other models to provide custom information, such as.
https://www.mlflow.org/docs/latest/models.html#model-api

- Python Function (python_function)
- R Function (crate)
- H2O (h2o)
- Keras (keras)
- MLeap (mleap)
- PyTorch (pytorch)
- Scikit-learn (sklearn)
- Spark MLlib (spark)
- TensorFlow (tensorflow)
- ONNX (onnx)
- MXNet Gluon (gluon)
- XGBoost (xgboost)
- LightGBM (lightgbm)
- CatBoost (catboost)
- Spacy(spaCy)
- Fastai(fastai)
- Statsmodels (statsmodels)

### Get meta object

An existing model can be found by invoking the `get_model` function.

```python

model_meta = mr.get_model(name="mnist",
                          version=1)			  
			  
```


### Get model provenance



```python

model_meta = mr.get_model(name="mnist",
                          version=1)
			  
model_meta.get_experiment().get_td()
			  
```


### Delete

An existing model can be deleted by invoking the `delete` function on the meta object.

```python

model_meta = mr.get_model(name="mnist",
                          version=1)

model_meta.delete()

```

### Finding best model version


The best performing version of a model family can be found by invoking the `get_models` function

```python

model_meta_list = mr.get_models(name="mnist",
                                sort_key="accuracy",
				order="desc")
				
best_model_meta = model_meta_list[0]

```



