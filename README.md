# Car data example model
Example code how to build simple in-memory function using Quix SDK on real-time data. The architecture is following:
 
[![](car-demo-model.svg)](car-demo-model.svg "Architecture") 

1) Car telemetry is streamed into the input topic.
2) Model is subscribed to input topic and is reading data realtime into memory.
3) Result of model function is streamed to output topic

## Code walkthrough 

Each instance of the model can process multiple streams. Quix SDK is designed to help you to react to each incoming stream in its scope. 
```python
# Callback called for each incoming stream
def read_stream(new_stream: StreamReader):
  # Here you react to each incoming stream individually.


# Hook up events before initiating read to avoid losing out on any data
input_topic.on_stream_received += read_stream
input_topic.start_reading()  # initiate read
```

In this case, we will create for each incoming stream one output stream with the result of our function. We also attach the new stream as a child of the input stream to persist data lineage. 

```python
# Create a new stream to output data
stream_writer = output_topic.create_stream(new_stream.stream_id + "-hard-braking")
    
stream_writer.properties.parents.append(new_stream.stream_id)
```

To read parameters from incoming stream, create a new buffer. In this example, we listen to data packets with **Brake** parameter.
```python
buffer = new_stream.parameters.create_buffer("Brake")
buffer.time_span_in_milliseconds = 100  # React to 100ms windows of data.

# Callback triggered for each new data frame
def on_parameter_data_handler(data: ParameterData):

    df = data.to_panda_frame()  # Input data frame
    output_df = pd.DataFrame()
    output_df["time"] = df["time"]

    output_df["TAG__LapNumber"] = df["TAG__LapNumber"]
    print(df)

    # If braking force applied is more than 50%, we send True.  
    output_df["HardBraking"] = df.apply(lambda row: "True" if row.Brake > 0.5 else "False", axis=1)  

    stream_writer.parameters.buffer.write(output_df)  # Send filtered data to output topic


# React to new data received from input topic.
buffer.on_read += on_parameter_data_handler
```

## Result
**If persistence is enabled**, car data and the result of the function is persisted. You can review the output of the function in the data catalogue.

[![](model-catalogue.png)](model-catalogue.png "Model in data catalogue")


[![](model-parameters.png)](model-parameters.png "Model parameters in parameter browser")

## What next
You can build a dashboard to visualize your data and share it with your others. See [car-data-dashboard](https://github.com/quixai/car-data-dashboard)
