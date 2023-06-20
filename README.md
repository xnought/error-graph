# `ErrorGraph`

Can I find patterns where a ML model fails?

`ErrorGraph` takes embeddings from the neural net, then constructs a graph where the incorrect data is connected to the nearest neigbors. The result is below.

Data -> Model Embeddings -> 2D -> Error Connections -> Force Graph Visualization


<img width="75%" alt="Screenshot 2023-06-20 at 7 09 42 PM" src="https://github.com/xnought/error-graph/assets/65095341/e520e43f-38b4-43fc-a96b-40e19c642b52">


Plus, you can compute graph algorithms on top of the structure (pagerank, etc.) if you want.

If you want to make something of this, go ahead and claim that its yours. I'm just here to experiment.
