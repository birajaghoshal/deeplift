DeepLIFT: Deep Learning Important FeaTures
===
[![Build Status](https://api.travis-ci.org/kundajelab/deeplift.svg?branch=dev-th)](https://travis-ci.org/kundajelab/deeplift)

**\*\*For a keras 2.0-compatible version developed using tensorflow 1.7, see the keras2compat branch.\*\***

Algorithms for computing importance scores in deep neural networks. Implements the methods in ["Learning Important Features Through Propagating Activation Differences"](https://arxiv.org/abs/1704.02685) by Shrikumar, Greenside & Kundaje, as well as other commonly-used methods such as gradients, [guided backprop](https://arxiv.org/abs/1412.6806) and [integrated gradients](https://arxiv.org/abs/1611.02639).

**Please be aware that figuring out optimal references is still an unsolved problem and we are actively working on a principled solution. Suggestions on good heuristics for different applications are welcome**

Please feel free to follow this repository to stay abreast of updates.

## Table of contents

  * [Installation](#installation)
  * [Quickstart](#quickstart)
  * [Examples](#examples)
  * [FAQ](#faq)
  * [Contact](#contact)
  * [Under The Hood](#under-the-hood)
    * [Blobs](#blobs)
    * [The Forward Pass](#the-forward-pass)
    * [The Backward Pass](#the-backward-pass)
  * [Coming Soon](#coming-soon)

## Installation

```unix
git clone https://github.com/kundajelab/deeplift.git #will clone the deeplift repository
pip install --editable deeplift/ #install deeplift from the cloned repository. The "editable" flag means changes to the code will be picked up automatically.
```

While DeepLIFT does not require your models to be trained with any particular library, we have provided autoconversion functions to convert models trained using Keras into the DeepLIFT format. If you used a different library to train your models, you can still use DeepLIFT if you recreate the model using DeepLIFT layers.

The theano implementation of DeepLIFT was tested with theano 0.8 and 0.9, and autoconversion with sequential models was tested using keras 0.2, 0.3 and 1.2. Graph model conversion was tested with keras 0.3. Functional model conversion was tested with keras 1.2

For a keras 2.0-compatible version developed using tensorflow 1.7, see the keras2compat branch.

The recommended way to obtain theano and numpy is through [anaconda](https://www.continuum.io/downloads).

## Quickstart

These examples show how to autoconvert a keras model and obtain importance scores. Non-keras models can be manually converted to the DeepLIFT format as well (contact avanti [dot] shrikumar@gmail.com if you are interested in this so I can prioritise doccumenting this accordingly)

```python
#Convert a keras sequential model
import deeplift
from deeplift.conversion import keras_conversion as kc
#NonlinearMxtsMode defines the method for computing importance scores.
#NonlinearMxtsMode.DeepLIFT_GenomicsDefault uses the RevealCancel rule on Dense layers
#and the Rescale rule on conv layers (see paper for rationale)
#Other supported values are:
#NonlinearMxtsMode.RevealCancel - DeepLIFT-RevealCancel at all layers (used for the MNIST example)
#NonlinearMxtsMode.Rescale - DeepLIFT-rescale at all layers
#NonlinearMxtsMode.Gradient - the 'multipliers' will be the same as the gradients
#NonlinearMxtsMode.GuidedBackprop - the 'multipliers' will be what you get from guided backprop
#Use deeplift.util.get_integrated_gradients_function to compute integrated gradients
#Feel free to email avanti [dot] shrikumar@gmail.com if anything is unclear
deeplift_model = kc.convert_sequential_model(
                    keras_model,
                    nonlinear_mxts_mode=deeplift.blobs.NonlinearMxtsMode.DeepLIFT_GenomicsDefault)

#Specify the index of the layer to compute the importance scores of.
#In the example below, we find scores for the input layer, which is idx 0 in deeplift_model.get_layers()
find_scores_layer_idx = 0

#Compile the function that computes the contribution scores
#For sigmoid or softmax outputs, target_layer_idx should be -2 (the default)
#(See "3.6 Choice of target layer" in https://arxiv.org/abs/1704.02685 for justification)
#For regression tasks with a linear output, target_layer_idx should be -1
#(which simply refers to the last layer)
#If you want the DeepLIFT multipliers instead of the contribution scores, you can use get_target_multipliers_func
deeplift_contribs_func = deeplift_model.get_target_contribs_func(
                            find_scores_layer_idx=find_scores_layer_idx,
                            target_layer_idx=-1)
#You can also provide an array of indices to find_scores_layer_idx to get scores for multiple layers at once

#compute scores on inputs
#input_data_list is a list containing the data for different input layers
#eg: for MNIST, there is one input layer with with dimensions 1 x 28 x 28
#In the example below, let X be an array with dimension n x 1 x 28 x 28 where n is the number of examples
#task_idx represents the index of the node in the output layer that we wish to compute scores.
#Eg: if the output is a 10-way softmax, and task_idx is 0, we will compute scores for the first softmax class
scores = np.array(deeplift_contribs_func(task_idx=0,
                                         input_data_list=[X],
                                         batch_size=10,
                                         progress_update=1000))
```

This will work for sequential models involving dense and/or conv2d layers and linear/relu/sigmoid/softmax or prelu activations. Please create a github issue or email avanti [dot] shrikumar@gmail.com readme if you are interested in support for other layer types.

The syntax for autoconverting graph models is similar:

```python
#Convert a keras graph model
import deeplift
from deeplift.conversion import keras_conversion as kc
deeplift_model = kc.convert_graph_model(
                    keras_model,
                    nonlinear_mxts_mode=deeplift.blobs.NonlinearMxtsMode.DeepLIFT_GenomicsDefault)
#For sigmoid or softmax outputs, this should be the name of the linear layer preceding the final nonlinearity
#(See "3.6 Choice of target layer" in https://arxiv.org/abs/1704.02685 for justification)
#For regression tasks with a linear output, this should simply be the name of the final layer
#You can find the name of the layers by inspecting the keys of deeplift_model.get_name_to_blob()
deeplift_contribs_func = deeplift_model.get_target_contribs_func(
    find_scores_layer_name="name_of_input_layer",
    pre_activation_target_layer_name="name_goes_here")
#You can also provide an array of names ot find_scores_layer_name to get
#scores for multiple layers at once
```

For autoconverting a functional model:
```python
deeplift_model = kc.convert_functional_model(
                    keras_model,
                    nonlinear_mxts_mode=deeplift.blobs.NonlinearMxtsMode.DeepLIFT_GenomicsDefault)
#The syntax below for obtaining scores is similar to that of a converted graph model
#See deeplift_model.get_name_to_blob().keys() to see all the layer names
#As before, you can provide an array of names to find_scores_layer_name
#to get the scores for multiple layers at once
deeplift_contribs_func = deeplift_model.get_target_contribs_func(
    find_scores_layer_name="name_of_input_layer",
    pre_activation_target_layer_name="name_goes_here")
```

## Examples
Please explore the examples folder in the main repository for ipython notebooks illustrating the use of DeepLIFT and other importance-scoring methods. Figures from the paper are reproduced here.

## FAQ

#### Keras 2?
For a keras 2-compatible version developed using tensorflow 1.7, see the keras2compat branch. You can also look at the implementation provided by the authors of [DeepExplain](https://github.com/marcoancona/DeepExplain) which has DeepLIFT with the Rescale rule (the authors found that “Integrated Gradients and DeepLIFT (with the Rescale rule) have very high correlation, suggesting that the latter is a good (and faster) approximation of the former in practice”).

#### Non-keras models?
If you are able to convert your model into the saved file format used by the Keras 2 API, then you can use the keras2compat branch to load it into the DeepLIFT format. The keras2compat branch works directly from keras saved files, without ever actually loading the models into keras.

#### What do negative scores mean?
A negative contribution score on an input means that the input contributed to moving the output below its reference value, where the reference value of the output is the value that it has when provided the reference input. A negative contribution does not mean that the input is "unimportant". If you want to find inputs that DeepLIFT considers "unimportant" (i.e. DeepLIFT thinks they don't influence the output of the model much), these would be the inputs that have contribution scores near 0.

#### How do I provide a reference argument?

Just as you supply `input_data_list` as an argument to the scoring function, you can also supply `input_references_list`. It would have the same dimensions as `input_data_list`, but would contain reference images for each input.

#### What should I use as my reference?

The choice of reference depends on the question you wish to ask of the data. Generally speaking, the reference should retain the properties you don't care about and scramble the properties you do care about. In the [supplement of the DeepLIFT paper](http://proceedings.mlr.press/v70/shrikumar17a/shrikumar17a-supp.pdf), Appendix L looks at the results on a CIFAR10 model with two different choices of the reference. You'll notice that when a blurred version of the input is used as a reference, the outlines of objects stand out. When a black reference is used, the results are more confusing, possibly because the net is also highlighting color. One thing to consider is using multiple different references to interpret a single image and averaging the results over all the different references. We use this approach in genomics; we generate a collection of references per input sequence by shuffling the sequence (this is demonstrated in the genomics example notebook).

#### Can I have multiple input modes?

Yes. Rather than providing a single numpy array to input_data_list, provide a list of numpy arrays containing the input to each mode. You can also provide a dictionary to input_data_list where the key is the mode name and the value is the numpy array. Each numpy array should have the first axis be the sample axis.

#### Can I get the contribution scores on multiple input layers at once?

Also yes. Just provide a list to `find_scores_layer_name` rather than a single argument.

#### How does DeepLIFT compare to integrated gradients?

As illustrated in the DeepLIFT paper, the RevealCancel rule of DeepLIFT can allow DeepLIFT to properly handle cases where integrated gradients may give misleading results. [Independent researchers](https://arxiv.org/pdf/1711.06104.pdf) have found that DeepLIFT with just the Rescale rule performs comparably to Integrated Gradients (they write: “Integrated Gradients and DeepLIFT have very high correlation, suggesting that the latter is a good (and faster) approximation of the former in practice”). Their finding was consistent with our own personal experience. The speed improvement of DeepLIFT relative to Integrated Gradients becomes particularly useful when using a collection of references (since having a collection of references per example increases runtime).

#### What's the license?
MIT License. While we had originally filed a patent on some of our interpretability work, we have since disbanded the patent as it appears this project has enough interest from the community to be best distributed in open-source format.

## Contact
Please email avanti [dot] shrikumar [at] gmail.com with questions, ideas, feature requests, etc. If I don't respond, keep emailing me until I feel guilty and respond. Also feel free to email my adviser (anshul [at] kundaje [dot] net), who can further guilt trip me into responding. I promise I do actually want to respond; I'm just busy with other things because the incentive structure of academia doesn't reward maintenance of projects.

Please consider joining the [google group](https://groups.google.com/forum/#!forum/deeplift) or following the repository to stay abreast of updates. It seems that the ML community favors posting github issues over using Google Groups.

## Under the hood
This section explains finer aspects of the deeplift implementation

### Blobs
The blob (`deeplift.blobs.core.Blob`) is the basic unit; it is the equivalent of a "layer", but was named a "blob" so as not to imply a sequential structure. `deeplift.blobs.core.Dense` and `deeplift.blobs.convolution.Conv2D` are both examples of blobs.

Blobs implement the following key methods:
#### get_activation_vars()
Returns symbolic variables representing the activations of the blob. For an understanding of symbolic variables, refer to the documentation of symbolic computation packages like theano or tensorflow.

#### get_pos_mxts() and get_neg_mxts()
Returns symbolic variables representing the positive/negative multipliers on this layer (for the selected output). See paper for details.

#### get_target_contrib_vars()
Returns symbolic variables representing the importance scores. This is a convenience function that returns `self.get_pos_mxts()*self._pos_contribs() + self.get_neg_mxts()*self._neg_contribs()`. See paper for details.

### The Forward Pass
Here are the steps necessary to implement a forward pass. If executed correctly, the results should be identical (within numerical precision) to a forward pass of your original model, so this is definitely worth doing as a sanity check. Note that if autoconversion (as described in the quickstart) is an option, you can skip steps (1) and (2).

1. Create a blob object for every layer in the network
2. Tell each blob what its inputs are via the `set_inputs` function. The argument to `set_inputs` depends on what the blob expects
  - If the blob has a single blob as its input (eg: Dense layers), then the argument is simply the blob that is the input
  - If the blob takes multiple blobs as its input, the argument depends on the specific implementation - for example, in the case of a Concat layer, the argument is a list of blobs 
3. Once every blob is linked to its inputs, you may compile the forward propagation function with `deeplift.backend.function([input_layer.get_activation_vars()...], output_layer.get_activation_vars())`
  - If you are working with a model produced by autoconversion, you can access individual blobs via `model.get_layers()` for sequential models (where this function would return a list of blobs) or `model.get_name_to_blob()` for Graph models (where this function would return a dictionary mapping blob names to blobs) 
  - The first argument is a list of symbolic tensors representing the inputs to the net. If the net has only one input blob, then this will be a list containing only one tensor
  - The second argument is the output of the function. In the example above, it is a single tensor, but it can also be a list of tensors if you want the outputs of more than one blob
4. Once the function is compiled, you can use `deeplift.util.run_function_in_batches(func, input_data_list)` to run the function in batches (which would be advisable if you want to call the function on a large number of inputs that wont fit in memory)
  - `func` is simply the compiled function returned by `deeplift.backend.function`
  - `input_data_list` is a list of numpy arrays containing data for the different input layers of the network. In the case of a network with one input, this will be a list containing one numpy array
  - Optional arguments to `run_function_in_batches` are `batch_size` and `progress_update`

### The Backward Pass
Here are the steps necessary to implement the backward pass, which is where the importance scores are calculated. Ideally, you should create a model through autoconversion (described in the quickstart) and then use `model.get_target_contribs_func` or `model.get_target_multipliers_func`. Howver, if that is not an option, read on (please also consider sending us a message to let us know, as if there is enough demand for a feature we will consider adding it). Note the instructions below assume you have done steps (1) and (2) under the forward pass section.

1. For the blob(s) that you wish to compute the importance scores for, call `reset_mxts_updated()`. This resets the symbolic variables for computing the multipliers. If this is the first time you are compiling the backward pass, this step is not strictly necessary.
2. For the output blob(s) containing the neuron(s) that the importance scores will be calculated with respect to, call `set_scoring_mode(deeplift.blobs.ScoringMode.OneAndZeros)`.
    - Briefly, this is the scoring mode that is used when we want to find scores with respect to a single target neuron. Other kinds of scoring modes may be added later (eg: differences between neurons).
    - A point of clarification: when we eventually compile the function, it will be a function which computes scores for only a single output neuron in a single layer every time it is called. The specific neuron and layer can be toggled later, at runtime. Right now, at this step, you should call `set_scoring_mode` on all the target layers that you might conceivably want to find the scores with respect to. This will save you from having to recompile the function to allow a different target layer later.
    - For Sigmoid/Softmax output layers, the output blob that you use should be the linear blob (usually a Dense layer) that comes before the final nonlinear activation. See "3.6 Choice of target layer" in the paper for justification. If there is no final nonlinearity (eg: in the case of many regression tasks), then the output blob should just be the last linear blob. 
    - For Softmax outputs, you should may want to subtract the average contribution to all softmax classes as described in "Adjustments for softmax layers" in the paper (section 3.6). If your number of softmax classes is very large and you don't want to calculate contributions to each class separately for each example, contact me (avanti [dot] shrikumar@gmail.com) and I can implement a more efficient way to do the calculation (there is a way but I haven't coded it up yet).
3. For the blob(s) that you wish to compute the importance scores for, call `update_mxts()`. This will create the symbolic variables that compute the multipliers with respect to the layer specified in step 2.
4. Compile the importance score computation function with

    ```python
    deeplift.backend.function([input_layer.get_activation_vars()...,
                               input_layer.get_reference_vars()...],
                              blob_to_find_scores_for.get_target_contrib_vars())
    ```
    - The first argument represents the inputs to the function and should be a list of one symbolic tensor for the activations of each input layer (as for the forward pass), followed by a list of one symbolic tensor for the references of each input layer
    - The second argument represents the output of the function. In the example above, it is a single tensor containing the importance scores of a single blob, but it can also be a list of tensors if you wish to compute the scores for multiple blobs at once.
    - Instead of `get_target_contrib_vars()` which returns the importance scores (in the case of `NonlinearMxtsMode.DeepLIFT`, these are called "contribution scores"), you can use `get_pos_mxts()` or `get_neg_mxts()` to get the multipliers.
5. Now you are ready to call the function to find the importance scores.
    - Select a specific output blob to compute importance scores with respect to by calling `set_active()` on the blob.
    - Select a specific target neuron within the blob by calling `update_task_index(task_idx)` on the blob. Here `task_idx` is the index of a neuron within the blob.
    - Call the function compiled in step 4 to find the importance scores for the target neuron. Refer to step 4 in the forward pass section for tips on using `deeplift.util.run_function_in_batches` to do this.
    - Deselect the output blob by calling `set_inactive()` on the blob. Don't forget this!
    - (Yes, I will bundle all of these into a single function at some point)

## Coming soon
The following is a list of some features in the works:
- Rigorous unit tests for batch norm for version 5 (this version)
- Improved strategies for handling references
- extensions to RNNs

If you would like early access to any of those features, please contact us.
