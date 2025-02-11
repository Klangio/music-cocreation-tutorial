## Interactive music co-creation with PyTorch and TensorFlow.js

This tutorial is a start-to-finish demonstration ([click here for result](https://chrisdonahue.com/music-cocreation-tutorial)) of building an interactive music co-creation system in two parts:

1. **[Training a generative model of music in Python](#part-1-training-in-python)** (via PyTorch)
2. **[Deploying it in JavaScript for interaction](#part-2-deploying-in-javascript)** (via TensorFlow.js)

This demonstration was prepared by [Chris Donahue](https://chrisdonahue.com) as part of an [ISMIR 2021 tutorial](https://ismir2021.ismir.net/tutorials/) on *Designing generative models for interactive co-creation*, co-organized by [Anna Huang](https://research.google/people/105787/) and [Jon Gillick](https://www.jongillick.com).

The example generative model we will train and deploy is [Piano Genie](https://magenta.tensorflow.org/pianogenie) (Donahue et al. 2019). Piano Genie allows anyone to improvise on the piano by mapping performances on a miniature 8-button keyboard to realistic performances on a full 88-key piano in real time. We train Piano Genie by autoencoding expert piano performances: an encoder maps 88-key piano performances into 8-button "button performances", and a decoder attempts to reconstruct the piano performance from the button performance. At interaction time, we replace the encoder with a user performing on the buttons. At a low-level, the decoder is an LSTM that operates on symbolic music data (i.e., MIDI), and is lightweight enough for real-time performance on mobile CPUs.

<p align="center"><img src="part-1-py-training/figures/overview.png" width=600px/></p>

### Part 1: Training in Python

<a href="https://colab.research.google.com/drive/124pk1yehPx1y-K3hBG6-SoUSVqQ-RWnM?usp=sharing" target="_blank"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

This part of the tutorial involves training a music generative model (Piano Genie) from scratch in PyTorch, which comes in the form of a self-contained [Google Colab notebook](https://colab.research.google.com/drive/124pk1yehPx1y-K3hBG6-SoUSVqQ-RWnM?usp=sharing). The instructions for this part are embedded in the Colab, and the model takes about an hour to train on Colab's free GPUs. The outputs of this part are: (1) a [model checkpoint](part-2-js-interaction/pretrained), and (2) [serialized inputs and outputs for a test case](part-2-js-interaction/test/test.json), which we will use to check correctness of our JavaScript port in the next part.

### Part 2: Deploying in JavaScript

<a href="https://glitch.com/edit/#!/music-cocreation-tutorial" target="_blank"><img src="https://cdn.glitch.com/2703baf2-b643-4da7-ab91-7ee2a2d00b5b%2Fremix-button.svg" alt="Remix on Glitch"/></a>

This part of the tutorial involves porting the trained generative model from PyTorch to TensorFlow.js, and hooking the model up to a simple UI to allow users to interact with the model. The final result is this [simple web demo](https://chrisdonahue.com/music-cocreation-tutorial). The static files for this demo are located in the [`part-2-js-interaction`](part-2-js-interaction) directory.

We use JavaScript as the target language for interaction because, unlike Python, it allows for straightforward prototyping and sharing of interactive UIs. However, note that JavaScript is likely not the best target when building tools that musicians can integrate into their workflows. For this, you probably want to integrate with digital audio workstations through C++ plugin frameworks like [VSTs](https://en.wikipedia.org/wiki/Virtual_Studio_Technology) or visual programming environments like [Max/MSP](https://en.wikipedia.org/wiki/Max_(software)).

At time of writing, [TensorFlow.js](https://www.tensorflow.org/js) is the most mature JavaScript framework for client-side inference of machine learning models. If your Python model is written in TensorFlow, porting it to TensorFlow.js will be much easier, or perhaps even [automatic](https://www.tensorflow.org/js/tutorials/conversion/import_saved_model). However, if you prefer to use a different Python framework like PyTorch or JAX, porting to TensorFlow.js can be a bit tricky, but not insurmountable!

#### Porting weights from PyTorch to TensorFlow.js

At the end of Part 1, we exported our model parameters from PyTorch to a format that TensorFlow.js can recognize. For convenience, we've also included a relevant snippet here:

```py
import pathlib
from tensorflowjs.write_weights import write_weights
import torch

# Load saved model params
d = torch.load("model.pt", map_location=torch.device("cpu"))

# Convert to simple dictionary of named numpy arrays
d = {k: v.numpy() for k, v in d.items()}

# Save in TensorFlow.js format
pathlib.Path("output").mkdir(exist_ok=True)
write_weights([[{"name": k, "data": v} for k, v in d.items()]], "output")
```

This snippet will produce `output/group1-shard1of1.bin`, a binary file containing the model parameters, and `output/weights_manifest.json` a JSON spec which informs TensorFlow.js how to unpack the binary file into a JavaScript `Object`. Both of these files must be hosted in the same directory when loading the weights with TensorFlow.js

#### Creating a test case

At the end of Part 1, we also created a test case—a pair of raw inputs to and outputs from our trained model. For convenience, we've also included a relevant snippet here:

```py
import json
import torch

# Restore model from saved checkpoint
model = Model()
model.load_state_dict(torch.load("model.pt", map_location=device))
model.eval()

# Serialize a batch of inputs/outputs as JSON
with torch.no_grad():
    inputs = get_batch()
    outputs = model(inputs)
    test = {
        "inputs": inputs.cpu().numpy().tolist(),
        "outputs": outputs.cpu().numpy().tolist(),
    }
    with open("test.json", "w") as f:
        f.write(json.dumps(test))
```

This snippet will produce `test.json`, a JSON-encoded file containing serialized inputs and outputs for our model. We can use this later to check our ported model for correctness.

#### Redefining the model in TensorFlow.js

Next, we will write equivalent TensorFlow.js code to reimplement our PyTorch model. This is tricky, and will likely require unit testing (I have ported several models from Python to JavaScript and have yet to get it right on the first try) as well as a lot of poking around in the [documentation](https://js.tensorflow.org/api/latest/).

One tip is to try to make the APIs for the Python and JavaScript models as similar as possible. Here is a side-by-side comparison between the reference PyTorch model (from the [Colab notebook](part-1-py-training/Train.colab.ipynb)) and the TensorFlow.js equivalent (from [`part-2-js-interaction/modules.js`](part2-js-interaction/modules.js)):

<table>
    
<tr>
<th>PyTorch model</th>
<th>TensorFlow.js equivalent</th>
</tr>
    
<tr>
<td>

```py
class PianoGenieDecoder(nn.Module):
    def __init__(self, rnn_dim=128, rnn_num_layers=2):
        super().__init__()
        self.rnn_dim = rnn_dim
        self.rnn_num_layers = rnn_num_layers
        self.input = nn.Linear(PIANO_NUM_KEYS + 3, rnn_dim)
        self.lstm = nn.LSTM(
            rnn_dim,
            rnn_dim,
            rnn_num_layers,
            batch_first=True,
            bidirectional=False,
        )
        self.output = nn.Linear(rnn_dim, 88)
    
    def init_hidden(self, batch_size, device=None):
        h = torch.zeros(self.rnn_num_layers, batch_size, self.rnn_dim, device=device)
        c = torch.zeros(self.rnn_num_layers, batch_size, self.rnn_dim, device=device)
        return (h, c)

    def forward(self, k_m1, t, b, h_0=None):
        # Encode input
        inputs = [
            F.one_hot(k_m1, PIANO_NUM_KEYS + 1),
            t.unsqueeze(dim=2),
            b.unsqueeze(dim=2),
        ]
        x = torch.cat(inputs, dim=2)

        # Project encoded inputs
        x = self.input(x)

        # Run RNN
        if h_0 is None:
            h_0 = self.init_hidden(k.shape[0], device=k.device)
        x, h_N = self.lstm(x, h_0)

        # Compute logits
        hat_k = self.output(x)

        return hat_k, h_N
```
    
</td>
<td>
    
```js
class PianoGenieDecoder extends Module {
  constructor(rnnDim, rnnNumLayers) {
    super();
    this.rnnDim = rnnDim === undefined ? 128 : rnnDim;
    this.rnnNumLayers = rnnNumLayers === undefined ? 2 : rnnNumLayers;
  }

  initHidden(batchSize) {
    // NOTE: This allocates memory that must later be freed
    const c = [];
    const h = [];
    for (let i = 0; i < this.rnnNumLayers; ++i) {
      c.push(tf.zeros([batchSize, this.rnnDim], "float32"));
      h.push(tf.zeros([batchSize, this.rnnDim], "float32"));
    }
    return new LSTMHiddenState(c, h);
  }

  forward(kim1, ti, bi, him1) {
    // Encode input
    const inputs = [
      tf.oneHot(kim1, PIANO_NUM_KEYS + 1),
      tf.expandDims(ti, 1),
      tf.expandDims(bi, 1)
    ];
    let x = tf.concat(inputs, 1);

    // Project encoded inputs
    x = tf.add(
      tf.matMul(x, this._params["dec.input.weight"], false, true),
      this._params[`dec.input.bias`]
    );

    // Run RNN
    if (him1 === undefined || him1 === null) {
      him1 = this.initHidden(kim1.shape[0]);
    }
    const [hic, hih] = tf.multiRNNCell(this._cells, x, him1.c, him1.h);
    x = hih[this.rnnNumLayers - 1];
    const hi = new LSTMHiddenState(hic, hih);

    // Compute logits
    const hatki = tf.add(
      tf.matMul(x, this._params["dec.output.weight"], false, true),
      this._params[`dec.output.bias`]
    );

    return [hatki, hi];
  }
}
```

</td>
</tr>
</table>

While these models have similar APIs, note that the PyTorch model takes as input entire sequences, i.e., tensors of shape `[batch_size, seq_len]`, while the TensorFlow.js equivalent takes an individual timestep as input, i.e., tensors of shape `[batch_size]`. This is because in Python, we are passing as input full sequences of training data, while in JavaScript, we will be using the model in an on-demand fashion, passing in user inputs as they become available.

Note that this implementation makes use of several helpers, such as a `Module` class which mocks behavior in `torch.nn.Module`, and a factory method `pyTorchLSTMCellFactory` which handles subtle differences in the LSTM implementations between PyTorch and TensorFlow.js. See [`part-2-js-interaction/modules.js`](part-2-js-interaction/modules.js) for full implementation.

#### Testing for correctness

Now that we have our model redefined in JavaScript, it is critical that we test it to ensure the behavior is identical to that of the original Python version. The easiest way to do this is to serialize a batch of inputs to and outputs from your Python model as JSON. Then, you can run those inputs through your JavaScript model and check if the outputs are identical (modulo some inevitable numerical precision error). Such a test case might look like this (from [`part-2-js-interaction/modules.js`](part-2-js-interaction/modules.js)):

```js
async function testPianoGenieDecoder() {
  const numBytesBefore = tf.memory().numBytes;

  // Create model
  const quantizer = new IntegerQuantizer();
  const decoder = new PianoGenieDecoder();
  await decoder.init();

  // Fetch test case
  const t = await fetch(TEST_CASE_URI).then(r => r.json());

  // Run test
  let totalErr = 0;
  let him1 = null;
  for (let i = 0; i < 128; ++i) {
    him1 = tf.tidy(() => {
      const kim1 = tf.tensor(t["input_keys"][i], [1], "int32");
      const ti = tf.tensor(t["input_dts"][i], [1], "float32");
      let bi = tf.tensor(t["input_buttons"][i], [1], "float32");
      bi = quantizer.discreteToReal(bi);
      const [khati, hi] = decoder.forward(kim1, ti, bi, him1);

      const expectedLogits = tf.tensor(
        t["output_logits"][i],
        [1, 88],
        "float32"
      );
      const err = tf.sum(tf.abs(tf.sub(khati, expectedLogits))).arraySync();
      totalErr += err;

      if (him1 !== null) him1.dispose();
      return hi;
    });
  }

  // Check equivalence to expected outputs
  if (isNaN(totalErr) || totalErr > 0.015) {
    console.log(totalErr);
    throw "Failed test";
  }

  // Check for memory leaks
  him1.dispose();
  decoder.dispose();
  if (tf.memory().numBytes !== numBytesBefore) {
    throw "Memory leak";
  }
  quantizer.dispose();

  console.log("Passed test");
}
```

Note that this function makes use of the [`tf.tidy`](https://js.tensorflow.org/api/latest/#tidy) wrapper. TensorFlow.js is [unable to automatically manage memory](https://www.tensorflow.org/js/guide/tensors_operations#memory) when running on GPUs via WebGL, so best practices for writing TensorFlow.js code involve some amount of manual memory management. The `tf.tidy` wrapper makes this easy—any tensor that is allocated during (but not returned by) the wrapped function will be automatically freed when the wrapped function finishes. However, in this case we have two sets of tensors that must persist across model calls: the model's parameters and the RNN memory. Unfortunately, we will have to carefully and manually dispose of these to prevent memory leaks.

#### Hooking model up to simple UI

Now that we have ported and tested our model, we can finally have some fun and start building out the interactive elements! Our demo includes a [simple HTML UI](part-2-js-interaction/index.html) with 8 buttons, and a [script](part-2-js-interaction/script.js) which hooks the frontend up to the model. Our script makes use of the wonderful [Tone.js](https://tonejs.github.io/) library to quickly build out a polyphonic FM synthesizer. We also [build a higher-level API](part-2-js-interaction/piano-genie.js) around our low-level model port, to handle things like keeping track of state and sampling from a model's distribution. 

While this is the stopping point of the tutorial, I would encourage you to experiment further. You're now at the point where the benefits of porting the model to JavaScript are clear: JavaScript makes it fairly straightforward to add additional functionality to enrich the interaction. For example, you could add an piano keyboard display to visualize Piano Genie's outputs, bind the space bar to act as a sustain pedal to Piano Genie, or you could use the WebMIDI API to output notes to a hardware synthesizer (hint: all of this functionality is built into [Monica Dinculescu](https://meowni.ca/)'s official [Piano Genie demo](https://www.w3.org/TR/webmidi/)). The possibilities are endless!

### End matter

#### Licensing info

This tutorial uses the [MAESTRO dataset](https://magenta.tensorflow.org/datasets/maestro) (Hawthorne et al. 2018), which is distributed under a [CC BY-NC-SA 4.0 license](https://creativecommons.org/licenses/by-nc-sa/4.0/). Because of the ShareAlike clause, the material in this tutorial is also distributed under that same license.

#### Acknowledgements

Thanks to Ian Simon and Sander Dieleman, co-authors on Piano Genie, and to Monica Dinculescu for creating the original [Piano Genie demo](https://piano-genie.glitch.me).

#### Attribution

If this tutorial was useful to you, please consider citing this repository.

```
@software{donahue2021tutorial,
  author={Donahue, Chris and Huang, Cheng-Zhi Anna and Gillick, Jon},
  title={Interactive music co-creation with {PyTorch} and {TensorFlow.js}},
  url={https://github.com/chrisdonahue/music-cocreation-tutorial},
  year={2021}
}
```

If Piano Genie was useful to you, please consider citing our original paper.

```
@inproceedings{donahue2019genie,
  title={Piano Genie},
  author={Donahue, Chris and Simon, Ian and Dieleman, Sander},
  booktitle={Proceedings of the 24th ACM Conference on Intelligent User Interfaces},
  year={2019}
}
```
