# Step by step example how to debug accuracy with Neural Insights
1. [Introduction](#introduction)
2. [Preparation](#preparation)
3. [Running the quantization](#running-the-quantization)
4. [Analyzing the result of quantization](#-analyzing-the-result-of-quantization)

# Introduction
In this instruction accuracy issue will be debugged using Neural Insights. ONNX LayoutLMv3 model will be used as an example. It will be quantized and the results will be analyzed to find the cause of the accuracy loss.

# Preparation
## Requirements
First you need to install Intel® Neural Compressor and other requirements.
```shell
pip install neural-compressor 
pip install datasets transformers torch torchvision
pip install onnx onnxruntime onnxruntime-extensions
pip install accelerate seqeval tensorboard sentencepiece timm fvcore Pillow einops textdistance shapely protobuf setuptools optimum
```

## Model
Get the LayoutLMv3 model from Intel® Neural Compressor [LayoutLMv3 example](https://github.com/intel/neural-compressor/tree/master/examples/onnxrt/nlp/huggingface_model/token_classification/layoutlmv3/quantization/ptq_static).
```shell
optimum-cli export onnx --model HYPJUDY/layoutlmv3-base-finetuned-funsd layoutlmv3-base-finetuned-funsd-onnx/ --task=token-classification
```

# Running the quantization
Generate a quantized model.
```python
onnx_model = onnx.load(input_model)
calib_dataset = IncDataset(eval_dataset, onnx_model)
config = PostTrainingQuantConfig(approach='static', quant_format="QOperator")
q_model = quantization.fit(onnx_model, 
                           config,
                           calib_dataloader=DataLoader(framework='onnxruntime', dataset=calib_dataset))
```

Execute benchmark to get the F1 score of both FP32 and INT8 models and then compute the relative accuracy ratio.
The output results indicate that the quantized model's accuracy is noticeably poor.

```
fp32 f1 = 0.9049, int8 f1 = 0.2989, accuracy ratio = -66.9631%
```

# Analyzing the result of quantization
In this section, the diagnosis tool is used for debugging to achieve higher INT8 model accuracy.
We need to set `diagnosis` parameter to `True` as shown below.
```python
config = PostTrainingQuantConfig(approach="static", quant_format="QOperator", quant_level=1, diagnosis=True) # set 'diagnosis' to True
q_model = quantization.fit(onnx_model, 
                           config, 
                           eval_func=eval_func, 
                           calib_dataloader=DataLoader(framework='onnxruntime', dataset=calib_dataset))
```
The diagnosis tool will output `Activations summary` and `Weights summary` in terminal. 

For easy to check, here we reload them to .csv files as shown below.
```python
import glob
import pandas as pd
pd.set_option('display.max_rows',None)
pd.set_option('display.max_columns',None)

subfolders = glob.glob("./nc_workspace" + "/*/")
subfolders.sort(key=os.path.getmtime, reverse=True)
if subfolders:
    activations_table = os.path.join(subfolders[0], "activations_table.csv")
    weights_table = os.path.join(subfolders[0], "weights_table.csv")
    
    activations_table = pd.read_csv(activations_table)
    weights_table = pd.read_csv(weights_table)
    
    print("Activations summary")
    display(activations_table)
    
    print("\nWeights summary")
    display(weights_table)
```

## Weights summary
These are the top 10 rows from weights summary table:

![weights_summary_onnx](./imgs/weights_summary_onnx.jpg)

## Activations summary
These are the top 10 rows from activations summary table:

![activations_summary_onnx](./imgs/activations_summary_onnx.jpg)

In the Activations summary table, there are some nodes showing dispersed activation data range. Therefore, we calculate the `Min-Max data range` for activations data and sort the results in descending order.

```python
activations_table["Min-Max data range"] = activations_table["Activation max"] - activations_table["Activation min"]
sorted_data = activations_table.sort_values(by="Min-Max data range", ascending=False)
display(sorted_data)
```

The results should look like below:

![min-max](./imgs/min-max.jpg)

According to the results displayed above, it is evident that the nodes of type `/layoutlmv3/encoder/layer.\d+/output/Add` and `/layoutlmv3/encoder/layer.\d+/output/dense/MatMul` have significantly higher values for `Min-Max data range` compared to other node types. This indicates that they may have caused a loss of accuracy. Therefore, we can try to fallback these nodes.

Refer to [diagnosis.md](https://github.com/intel/neural-compressor/blob/master/docs/source/diagnosis.md) for more tips for diagnosis.

```python
from neural_compressor.utils.constant import FP32
config = PostTrainingQuantConfig(approach="static", 
                                 quant_format="QOperator",
                                 op_name_dict={"/layoutlmv3/encoder/layer.\d+/output/dense/MatMul":FP32,
                                               "/layoutlmv3/encoder/layer.\d+/output/Add":FP32})
q_model = quantization.fit(onnx_model, 
                           config,
                           calib_dataloader=DataLoader(framework='onnxruntime', dataset=calib_dataset))
q_model.save(output_model)
```

Execute benchmark on the new quantized model again and the accuracy ratio is improved to <1%.
```
fp32 f1 = 0.9049, int8 f1 = 0.8981, accuracy ratio = -0.7502%
```