# HateSpeech_TIS
Hate speech detection service via NVIDIA triton inference server
## Guideline for deploying service
The guide will consist of the following steps:
1. [Converting model to onnx format](#convert)
2. [Organization of the service repository](#repo)
3. [Setting up configuration files](#setup)
4. [Local deployment and usage](#dev)
5. [Moving post and preprocessing to the backend](#ens)
6. [Using dynamic batching and instance group](#imp)
   
<a name="convert"><h3>1.Converting model to onnx format</h3></a>
Models written in different frameworks can be converted to the onnx format, there are no general instructions for conversion.

I will provide various links to examples:

[Github](https://github.com/triton-inference-server/tutorials/tree/main/Conceptual_Guide/Part_1-model_deployment)

[HuggingFace](https://huggingface.co/blog/convert-transformers-to-onnx)

[PyTorch](https://pytorch.org/tutorials/advanced/super_resolution_with_onnxruntime.html)

You can get ONNX model for this service here: [Download](https://snet-open-models.s3.amazonaws.com/hate-speech-detection/model.onnx) You should place this file inside hatespeech_repo/hate_speech_detection/1/ folder

<a name="repo"><h3>2.Organization of the service repository</h3></a>
The repository should look like:
```
hatespeech_repo/
└── hate_speech_detection
    ├── 1
    │   └── model.onnx
    └── config.pbtxt
```

Where:
1. `hatespeech_repo` - service root directory
2. `hate_speech_detection` - model directory
3. folders `1` - contain models under version 1
4. `config.pbtxt` - models configurations files

<a name="setup"><h3>3.Setting up configuration files</h3></a>
`config.pbtxt` for model should look like:
```
name: "hate_speech_detection"
backend: "onnxruntime"
max_batch_size : 0

input [
    {
        name: "input_ids"
        data_type: TYPE_INT64
        dims: [ 1 , -1 ]
    },
    {
        name: "attention_mask"
        data_type: TYPE_INT64
        dims: [ 1, -1 ]
    }
]
output [
    {
        name: "output"
        data_type: TYPE_FP32
        dims: [ -1, 4 ]
    }
]
```
Where:
1. `name` - service directory name 
2. `backend` - selected backend ([More info](https://github.com/triton-inference-server/backend))
3. `max_batch_size` - batch size limit
4. `input` and `output` blocks - blocks describing input and output tensors
5. Block fields:
   + `name` - name of tensor
   + `data_type` - tensor data type
   + `dims` - tensor dimensions

How to define dims:

Because we are working with a BERT-like model which has standard inputs input_ids and focus_mask. The shape of which is `[1, -1]` (One-dimensional tensor of length N).

The result is known in advance because the model returns a list of four elements (each element is the probability that the text belongs to the class at that index). Therefore, the output form will be `[-1, 4]`, `-1` since the model can accept N texts at a time.

From here you can also establish that the inputs are of type `INT` and the output is `FLOAT`.

<a name="dev"><h3>4.Local deployment and usage</h3></a>
Launching the server:
```
docker run --gpus='"device=0"' -it --shm-size=256m --rm -p8000:8000 -p8001:8001 -p8002:8002 -v ${PWD}/hatespeech_repo:/models nvcr.io/nvidia/tritonserver:23.12-py3
```
Inside container:
```
tritonserver --model-repository=/models
```
Building a client application:

<details>
  <summary>Full code</summary>

```python
import struct

import torch
from transformers import AutoTokenizer
import tritonclient.http as httpclient

from BERTweet.TweetNormalizer import normalizeTweet


# Sample text
text = 'Sample text'

# Tokenize and preprocess the text
tokenizer = AutoTokenizer.from_pretrained("vinai/bertweet-large")
normalized_text = normalizeTweet(text)
tokens = tokenizer(normalized_text, padding="longest", return_tensors="pt")
tokens = {k: torch.tensor(v, device='cpu') for k, v in tokens.items()}
# Extract input tensors
input_ids = tokens['input_ids']
attention_mask = tokens['attention_mask']

# Instantiate the Triton HTTP client
client = httpclient.InferenceServerClient(url="localhost:8000")

# Create InferInput objects for each input tensor
input_ids_input = httpclient.InferInput("input_ids", input_ids.shape, datatype="INT64")
input_ids_input.set_data_from_numpy(input_ids.cpu().numpy(), binary_data=True)

attention_mask_input = httpclient.InferInput("attention_mask", attention_mask.shape, datatype="INT64")
attention_mask_input.set_data_from_numpy(attention_mask.cpu().numpy(), binary_data=True)

# Perform inference
inputs = [input_ids_input, attention_mask_input]
response = client.infer(model_name="hate_speech_detection", inputs=inputs)

# Optionally, you can print the response
response_dict = vars(response)

# Assuming _buffer contains the binary data
binary_data = response_dict['_buffer']

# Decode binary data based on the datatype (FP32 in this case)
decoded_data = struct.unpack('f' * (len(binary_data) // 4), binary_data)

# Assuming the prediction is a list of probabilities
prediction = list(decoded_data)

# generate answer
res = []
res.append({"text": text,
            "hate": prediction[0],
            "abusing": prediction[1],
            "neutral": prediction[2],
            "spam": prediction[3]})
print(res)
```
</details>
Make call:

```
# pip install tritonclient[http]

python client.py
```

<a name="ens"><h3>5.Moving post and preprocessing to the backend</h3></a>
The client application from the last part turned out to be very large and not entirely correct. Let's move post and preprocessing to our backend to fix it.

Need to use [Model Ensembles](https://github.com/triton-inference-server/tutorials/tree/main/Conceptual_Guide/Part_5-Model_Ensembles)

The repository will look like this:
```
hatespeech_repo/
├── hate_speech_detection
│   ├── 1
│   │   └── model.onnx
│   └── config.pbtxt
└── ensemble_model
│   ├── 1
│   │   
│   └── config.pbtxt
└── preprocessing
│   ├── 1
│   │   └── model.py
│   └── config.pbtxt
└── postprocessing
    ├── 1
    │   │
    │   └──BERTweet
    │   │         └──TweetNormalizer.py
    │   └── model.py
    └── config.pbtxt
```
After write configs:

`preprocessing:`
```
name: "preprocessing"
backend: "python"
max_batch_size: 0
input [
    {
        name: "input_text"
        data_type: TYPE_STRING
        dims: [ -1 ]
    }
]
output [
    {
        name: "input_ids"
        data_type: TYPE_INT64
        dims: [ 1, -1 ]
    },
    {
    name: "attention_mask"
    data_type: TYPE_INT64
    dims: [ 1, -1 ]
    }
]
```

`postprocessing:`
```
name: "postprocessing"
backend: "python"
max_batch_size: 0
input [
    {
        name: "output"
        data_type: TYPE_FP32
        dims: [ -1, 4 ]
    }
]
output [
    {
        name: "answer"
        data_type: TYPE_STRING
        dims: [ -1 ]
    }
]
```
`ensemble_model:`

<details>
  <summary>Full code</summary>
   
```
name: "ensemble_model"
platform: "ensemble"
max_batch_size: 0
input [
  {
    name: "input_text"
    data_type: TYPE_STRING
    dims: [ -1 ]
  }
]
output [
  {
    name: "answer"
    data_type: TYPE_STRING
    dims: [ -1 ]
  }
]

ensemble_scheduling {
  step [
    {
      model_name: "preprocessing"
      model_version: -1
      input_map {
        key: "input_text"
        value: "input_text"
      }
      output_map {
        key: "input_ids"
        value: "input_ids"
      }
      output_map {
        key: "attention_mask"
        value: "attention_mask"
      }
    },
    {
      model_name: "hate_speech_detection"
      model_version: -1
      input_map {
        key: "input_ids"
        value: "input_ids"
      }
      input_map {
        key: "attention_mask"
        value: "attention_mask"
      }
      output_map {
        key: "output"
        value: "output"
      }
    },
    {
      model_name: "postprocessing"
      model_version: -1
      input_map {
        key: "output"
        value: "output"
      }
      output_map {
        key: "answer"
        value: "answer"
      }
    }
  ]
}
```
</details>

`hate_speech_detection` the same from previous parts.

Configs done, now write model.py files.

`preprocessing:`
<details>
  <summary>Full code</summary>

```python
import json

import torch
from transformers import AutoTokenizer
import triton_python_backend_utils as pb_utils

from BERTweet.TweetNormalizer import normalizeTweet


class TritonPythonModel:
    """Your Python model must use the same class name. Every Python model
    that is created must have "TritonPythonModel" as the class name.
    """

    def initialize(self, args):
        """`initialize` is called only once when the model is being loaded.
        Implementing `initialize` function is optional. This function allows
        the model to initialize any state associated with this model.
        Parameters
        ----------
        args : dict
          Both keys and values are strings. The dictionary keys and values are:
          * model_config: A JSON string containing the model configuration
          * model_instance_kind: A string containing model instance kind
          * model_instance_device_id: A string containing model instance device ID
          * model_repository: Model repository path
          * model_version: Model version
          * model_name: Model name
        """

        # You must parse model_config. JSON string is not parsed here
        model_config = json.loads(args["model_config"])

        # Get OUTPUT0 configuration
        output0_config = pb_utils.get_output_config_by_name(
            model_config, "input_ids"
        )

        # Convert Triton types to numpy types
        self.output0_dtype = pb_utils.triton_string_to_numpy(
            output0_config["data_type"]
        )

        # Get OUTPUT1 configuration
        output1_config = pb_utils.get_output_config_by_name(
            model_config, "attention_mask"
        )

        # Convert Triton types to numpy types
        self.output1_dtype = pb_utils.triton_string_to_numpy(
            output1_config["data_type"]
        )

    def execute(self, requests):
        """`execute` MUST be implemented in every Python model. `execute`
        function receives a list of pb_utils.InferenceRequest as the only
        argument. This function is called when an inference request is made
        for this model. Depending on the batching configuration (e.g. Dynamic
        Batching) used, `requests` may contain multiple requests. Every
        Python model, must create one pb_utils.InferenceResponse for every
        pb_utils.InferenceRequest in `requests`. If there is an error, you can
        set the error argument when creating a pb_utils.InferenceResponse
        Parameters
        ----------
        requests : list
          A list of pb_utils.InferenceRequest
        Returns
        -------
        list
          A list of pb_utils.InferenceResponse. The length of this list must
          be the same as `requests`
        """

        output0_dtype = self.output0_dtype
        output1_dtype = self.output1_dtype
        responses = []

        # Every Python backend must iterate over everyone of the requests
        # and create a pb_utils.InferenceResponse for each of them.
        for request in requests:
            # Get INPUT0
            in_1 = pb_utils.get_input_tensor_by_name(
                request, "input_text"
            )
            
            # Preprocessing inputs 
            tokenizer = AutoTokenizer.from_pretrained("vinai/bertweet-large")
            in_1 = in_1.as_numpy()
            in_1[0] = in_1[0].decode('utf-8')
            text = [normalizeTweet(in_1[0])]
            input_ids = tokenizer(text, return_tensors="pt")
            tokens = {k: torch.tensor(v, device='cpu') for k, v in input_ids.items()}

            # Extract input tensors
            input_ids = tokens['input_ids']
            attention_mask = tokens['attention_mask']

            
            out_tensor_0 = pb_utils.Tensor(
                "input_ids", input_ids.numpy().astype(output0_dtype)
            )

            out_tensor_1 = pb_utils.Tensor(
                "attention_mask", attention_mask.numpy().astype(output1_dtype)
            )

            # Create InferenceResponse. You can set an error here in case
            # there was a problem with handling this inference request.
            # Below is an example of how you can set errors in inference
            # response:
            #
            # pb_utils.InferenceResponse(
            #    output_tensors=..., TritonError("An error occurred"))
            inference_response = pb_utils.InferenceResponse(
                output_tensors=[out_tensor_0, out_tensor_1]
            )
            responses.append(inference_response)
        # You should return a list of pb_utils.InferenceResponse. Length
        # of this list must match the length of `requests` list.
        return responses

    def finalize(self):
        """`finalize` is called only once when the model is being unloaded.
        Implementing `finalize` function is OPTIONAL. This function allows
        the model to perform any necessary clean ups before exit.
        """
        print("Cleaning up...")
```
</details>

`postprocessing:`
<details>
  <summary>Full code</summary>

```python
import json

import numpy as np
import triton_python_backend_utils as pb_utils


class TritonPythonModel:
    """Your Python model must use the same class name. Every Python model
    that is created must have "TritonPythonModel" as the class name.
    """

    def initialize(self, args):
        """`initialize` is called only once when the model is being loaded.
        Implementing `initialize` function is optional. This function allows
        the model to initialize any state associated with this model.
        Parameters
        ----------
        args : dict
          Both keys and values are strings. The dictionary keys and values are:
          * model_config: A JSON string containing the model configuration
          * model_instance_kind: A string containing model instance kind
          * model_instance_device_id: A string containing model instance device ID
          * model_repository: Model repository path
          * model_version: Model version
          * model_name: Model name
        """

        # You must parse model_config. JSON string is not parsed here
        model_config = json.loads(args["model_config"])

        # Get OUTPUT0 configuration
        output0_config = pb_utils.get_output_config_by_name(
            model_config, "answer"
        )

        # Convert Triton types to numpy types
        self.output0_dtype = pb_utils.triton_string_to_numpy(
            output0_config["data_type"]
        )

    def execute(self, requests):
        """`execute` MUST be implemented in every Python model. `execute`
        function receives a list of pb_utils.InferenceRequest as the only
        argument. This function is called when an inference request is made
        for this model. Depending on the batching configuration (e.g. Dynamic
        Batching) used, `requests` may contain multiple requests. Every
        Python model, must create one pb_utils.InferenceResponse for every
        pb_utils.InferenceRequest in `requests`. If there is an error, you can
        set the error argument when creating a pb_utils.InferenceResponse
        Parameters
        ----------
        requests : list
          A list of pb_utils.InferenceRequest
        Returns
        -------
        list
          A list of pb_utils.InferenceResponse. The length of this list must
          be the same as `requests`
        """

        output0_dtype = self.output0_dtype
        responses = []

        # Every Python backend must iterate over everyone of the requests
        # and create a pb_utils.InferenceResponse for each of them.
        for request in requests:
            # Get INPUT0
            in_1 = pb_utils.get_input_tensor_by_name(
                request, "output"
            )
            in_1 = in_1.as_numpy()[0]

            # Assuming the prediction is a list of probabilities
            prediction = list(in_1)
            # generate answer
            answ = json.dumps({
                "hate": str(prediction[0]),
                "abusing": str(prediction[1]),
                "neutral": str(prediction[2]),
                "spam": str(prediction[3])
                })
            
            answ = np.array([answ], dtype="object")
            out_tensor_0 = pb_utils.Tensor(
                "answer", answ
            )

            # Create InferenceResponse. You can set an error here in case
            # there was a problem with handling this inference request.
            # Below is an example of how you can set errors in inference
            # response:
            #
            # pb_utils.InferenceResponse(
            #    output_tensors=..., TritonError("An error occurred"))
            inference_response = pb_utils.InferenceResponse(
                output_tensors=[out_tensor_0]
            )
            responses.append(inference_response)
        # You should return a list of pb_utils.InferenceResponse. Length
        # of this list must match the length of `requests` list.
        return responses

    def finalize(self):
        """`finalize` is called only once when the model is being unloaded.
        Implementing `finalize` function is OPTIONAL. This function allows
        the model to perform any necessary clean ups before exit.
        """
        print("Cleaning up...")
```
</details>

Everything is ready to start the server:
```
docker run --gpus='"device=0"' -it --shm-size=256m --rm -p8000:8000 -p8001:8001 -p8002:8002 -v ${PWD}/hatespeech_repo:/models nvcr.io/nvidia/tritonserver:23.12-py3
```
Inside container:
```
tritonserver --model-repository=/models
```
Building a client application:
```python
import time

import numpy as np
import tritonclient.grpc as grpcclient
from tritonclient.utils import np_to_triton_dtype


# define client
client = grpcclient.InferenceServerClient(url="localhost:8001")

# processing text
text_data = 'Sample text'
text_obj = np.array([text_data], dtype="object")

# convert input to tensor
input_tensors = [grpcclient.InferInput("input_text", text_obj.shape, np_to_triton_dtype(text_obj.dtype))]
input_tensors[0].set_data_from_numpy(text_obj)

# make call 
start_time = time.time()
results = client.infer(model_name="ensemble_model", inputs=input_tensors)
print("--- %s seconds ---" % (time.time() - start_time))

# convert answer to string
output_data = results.as_numpy("answer").astype(str)
print(output_data)
```
Then run client application:
```
# conda activate env_name
python client.py
# answer: ['{"hate": "0.0007111975", "abusing": "0.0004021231", "neutral": "0.0011137378", "spam": "0.9977728"}']
```
or
```
docker run -it --net=host -v ${PWD}:/workspace/ nvcr.io/nvidia/tritonserver:yy.mm-py3-sdk bash
# install libs
python3 client.py
# answer: ['{"hate": "0.0007111975", "abusing": "0.0004021231", "neutral": "0.0011137378", "spam": "0.9977728"}']
```
<a name="imp"><h3>6.Using dynamic batching and instance group</h3></a>
Dynamic Batching and Concurrent Model Execution are features of Triton that improve throughput ([More info](https://github.com/triton-inference-server/tutorials/tree/main/Conceptual_Guide/Part_2-improving_resource_utilization)).
To use them, just add the model configuration file:
```
name: "postprocessing"
backend: "python"
max_batch_size: 0
input [
    {
        name: "output"
        data_type: TYPE_FP32
        dims: [ -1, 4 ]
    }
]
output [
    {
        name: "answer"
        data_type: TYPE_STRING
        dims: [ -1 ]
    }
]

instance_group [{ kind: KIND_GPU }]
dynamic_batching { }
```
Also you can analyze you model using [Triton Analyzer](https://github.com/triton-inference-server/tutorials/tree/main/Conceptual_Guide/Part_2-improving_resource_utilization#measuring-performance)

<a name="ens"><h3>7. Additional files for publishing</h3></a>
Protobuf files for this service, as well as example of demo files for marketplace portal can be found in resources direcrory. 

For model file and where to put it check first section.
