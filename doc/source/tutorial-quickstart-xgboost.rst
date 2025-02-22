.. _quickstart-xgboost:


Quickstart XGBoost
==================

.. meta::
   :description: Check out this Federated Learning quickstart tutorial for using Flower with XGBoost to train classification models on trees.

Federated XGBoost
-------------

EXtreme Gradient Boosting (**XGBoost**) is a robust and efficient implementation of gradient-boosted decision tree (**GBDT**), that maximises the computational boundaries for boosted tree methods.
It's primarily designed to enhance both the performance and computational speed of machine learning models.
In XGBoost, trees are constructed concurrently, unlike the sequential approach taken by GBDT.

Often, for tabular data on medium-sized datasets with fewer than 10k training examples, XGBoost surpasses the results of deep learning techniques.

Why federated XGBoost?
~~~~~~~~

Indeed, as the demand for data privacy and decentralized learning grows, there's an increasing requirement to implement federated XGBoost systems for specialised applications, like survival analysis and financial fraud detection.

Federated learning ensures that raw data remains on the local device, making it an attractive approach for sensitive domains where data security and privacy are paramount.
Given the robustness and efficiency of XGBoost, combining it with federated learning offers a promising solution for these specific challenges.

In this tutorial we will learn how to train a federated XGBoost model on HIGGS dataset using Flower and :code:`xgboost` package.
We use a simple example (`full code xgboost-quickstart <https://github.com/adap/flower/tree/main/examples/xgboost-quickstart>`_) with two *clients* and one *server*
to demonstrate how federated XGBoost works,
and then we dive into a more complex example (`full code xgboost-comprehensive <https://github.com/adap/flower/tree/main/examples/xgboost-comprehensive>`_) to run various experiments.


Environment Setup
-------------

First of all, it is recommended to create a virtual environment and run everything within a `virtualenv <https://flower.dev/docs/recommended-env-setup.html>`_.

We first need to install Flower and Flower Datasets. You can do this by running :

.. code-block:: shell

  $ pip install flwr flwr-datasets

Since we want to use :code:`xgboost` package to build up XGBoost trees, let's go ahead and install :code:`xgboost`:

.. code-block:: shell

  $ pip install xgboost


Flower Client
-------------

*Clients* are responsible for generating individual weight-updates for the model based on their local datasets.
Now that we have all our dependencies installed, let's run a simple distributed training with two clients and one server.

In a file called :code:`client.py`, import xgboost, Flower, Flower Datasets and other related functions:

.. code-block:: python

    import argparse
    from typing import Union
    from logging import INFO
    from datasets import Dataset, DatasetDict
    import xgboost as xgb

    import flwr as fl
    from flwr_datasets import FederatedDataset
    from flwr.common.logger import log
    from flwr.common import (
        Code,
        EvaluateIns,
        EvaluateRes,
        FitIns,
        FitRes,
        GetParametersIns,
        GetParametersRes,
        Parameters,
        Status,
    )
    from flwr_datasets.partitioner import IidPartitioner

Dataset partition and hyper-parameter selection
~~~~~~~~

Prior to local training, we require loading the HIGGS dataset from Flower Datasets and conduct data partitioning for FL:

.. code-block:: python

    # Load (HIGGS) dataset and conduct partitioning
    # We use a small subset (num_partitions=30) of the dataset for demonstration to speed up the data loading process.
    partitioner = IidPartitioner(num_partitions=30)
    fds = FederatedDataset(dataset="jxie/higgs", partitioners={"train": partitioner})

    # Load the partition for this `node_id`
    partition = fds.load_partition(idx=args.node_id, split="train")
    partition.set_format("numpy")

In this example, we split the dataset into two partitions with uniform distribution (:code:`IidPartitioner(num_partitions=2)`).
Then, we load the partition for the given client based on :code:`node_id`:

.. code-block:: python

    # We first define arguments parser for user to specify the client/node ID.
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--node-id",
        default=0,
        type=int,
        help="Node ID used for the current client.",
    )
    args = parser.parse_args()

    # Load the partition for this `node_id`.
    partition = fds.load_partition(idx=args.node_id, split="train")
    partition.set_format("numpy")

After that, we do train/test splitting on the given partition (client's local data), and transform data format for :code:`xgboost` package.

.. code-block:: python

    # Train/test splitting
    train_data, valid_data, num_train, num_val = train_test_split(
        partition, test_fraction=0.2, seed=42
    )

    # Reformat data to DMatrix for xgboost
    train_dmatrix = transform_dataset_to_dmatrix(train_data)
    valid_dmatrix = transform_dataset_to_dmatrix(valid_data)

The functions of :code:`train_test_split` and :code:`transform_dataset_to_dmatrix` are defined as below:

.. code-block:: python

    # Define data partitioning related functions
    def train_test_split(partition: Dataset, test_fraction: float, seed: int):
        """Split the data into train and validation set given split rate."""
        train_test = partition.train_test_split(test_size=test_fraction, seed=seed)
        partition_train = train_test["train"]
        partition_test = train_test["test"]

        num_train = len(partition_train)
        num_test = len(partition_test)

        return partition_train, partition_test, num_train, num_test


    def transform_dataset_to_dmatrix(data: Union[Dataset, DatasetDict]) -> xgb.core.DMatrix:
        """Transform dataset to DMatrix format for xgboost."""
        x = data["inputs"]
        y = data["label"]
        new_data = xgb.DMatrix(x, label=y)
        return new_data

Finally, we define the hyper-parameters used for XGBoost training.

.. code-block:: python

    num_local_round = 1
    params = {
        "objective": "binary:logistic",
        "eta": 0.1,  # lr
        "max_depth": 8,
        "eval_metric": "auc",
        "nthread": 16,
        "num_parallel_tree": 1,
        "subsample": 1,
        "tree_method": "hist",
    }

The :code:`num_local_round` represents the number of iterations for local tree boost.
We use CPU for the training in default.
One can shift it to GPU by setting :code:`tree_method` to :code:`gpu_hist`.
We use AUC as evaluation metric.


Flower client definition for XGBoost
~~~~~~~~

After loading the dataset we define the Flower client.
We follow the general rule to define :code:`XgbClient` class inherited from :code:`fl.client.Client`.

.. code-block:: python

    class XgbClient(fl.client.Client):
        def __init__(self):
            self.bst = None
            self.config = None

The :code:`self.bst` is used to keep the Booster objects that remain consistent across rounds,
allowing them to store predictions from trees integrated in earlier rounds and maintain other essential data structures for training.

Then, we override :code:`get_parameters`, :code:`fit` and :code:`evaluate` methods insides :code:`XgbClient` class as follows.

.. code-block:: python

    def get_parameters(self, ins: GetParametersIns) -> GetParametersRes:
        _ = (self, ins)
        return GetParametersRes(
            status=Status(
                code=Code.OK,
                message="OK",
            ),
            parameters=Parameters(tensor_type="", tensors=[]),
        )

Unlike neural network training, XGBoost trees are not started from a specified random weights.
In this case, we do not use :code:`get_parameters` and :code:`set_parameters` to initialise model parameters for XGBoost.
As a result, let's return an empty tensor in :code:`get_parameters` when it is called by the server at the first round.

.. code-block:: python

    def fit(self, ins: FitIns) -> FitRes:
        if not self.bst:
            # First round local training
            log(INFO, "Start training at round 1")
            bst = xgb.train(
                params,
                train_dmatrix,
                num_boost_round=num_local_round,
                evals=[(valid_dmatrix, "validate"), (train_dmatrix, "train")],
            )
            self.config = bst.save_config()
            self.bst = bst
        else:
            for item in ins.parameters.tensors:
                global_model = bytearray(item)

            # Load global model into booster
            self.bst.load_model(global_model)
            self.bst.load_config(self.config)

            bst = self._local_boost()

        local_model = bst.save_raw("json")
        local_model_bytes = bytes(local_model)

        return FitRes(
            status=Status(
                code=Code.OK,
                message="OK",
            ),
            parameters=Parameters(tensor_type="", tensors=[local_model_bytes]),
            num_examples=num_train,
            metrics={},
        )

In :code:`fit`, at the first round, we call :code:`xgb.train()` to build up the first set of trees.
the returned Booster object and config are stored in :code:`self.bst` and :code:`self.config`, respectively.
From the second round, we load the global model sent from server to :code:`self.bst`,
and then update model weights on local training data with function :code:`local_boost` as follows:

.. code-block:: python

    def _local_boost(self):
        # Update trees based on local training data.
        for i in range(num_local_round):
            self.bst.update(train_dmatrix, self.bst.num_boosted_rounds())

        # Extract the last N=num_local_round trees for sever aggregation
        bst = self.bst[
            self.bst.num_boosted_rounds()
            - num_local_round : self.bst.num_boosted_rounds()
        ]

Given :code:`num_local_round`, we update trees by calling :code:`self.bst.update` method.
After training, the last :code:`N=num_local_round` trees will be extracted to send to the server.

.. code-block:: python

    def evaluate(self, ins: EvaluateIns) -> EvaluateRes:
        eval_results = self.bst.eval_set(
            evals=[(valid_dmatrix, "valid")],
            iteration=self.bst.num_boosted_rounds() - 1,
        )
        auc = round(float(eval_results.split("\t")[1].split(":")[1]), 4)

        return EvaluateRes(
            status=Status(
                code=Code.OK,
                message="OK",
            ),
            loss=0.0,
            num_examples=num_val,
            metrics={"AUC": auc},
        )

In :code:`evaluate`, we call :code:`self.bst.eval_set` function to conduct evaluation on valid set.
The AUC value will be returned.

Now, we can create an instance of our class :code:`XgbClient` and add one line to actually run this client:

.. code-block:: python

    fl.client.start_client(server_address="127.0.0.1:8080", client=XgbClient())

That's it for the client. We only have to implement :code:`Client`and call :code:`fl.client.start_client()`.
The string :code:`"[::]:8080"` tells the client which server to connect to.
In our case we can run the server and the client on the same machine, therefore we use
:code:`"[::]:8080"`. If we run a truly federated workload with the server and
clients running on different machines, all that needs to change is the
:code:`server_address` we point the client at.


Flower Server
-------------

These updates are then sent to the *server* which will aggregate them to produce a better model.
Finally, the *server* sends this improved version of the model back to each *client* to finish a complete FL round.

In a file named :code:`server.py`, import Flower and FedXgbBagging from :code:`flwr.server.strategy`.

We first define a strategy for XGBoost bagging aggregation.

.. code-block:: python

    # Define strategy
    strategy = FedXgbBagging(
        fraction_fit=1.0,
        min_fit_clients=2,
        min_available_clients=2,
        min_evaluate_clients=2,
        fraction_evaluate=1.0,
        evaluate_metrics_aggregation_fn=evaluate_metrics_aggregation,
    )

    def evaluate_metrics_aggregation(eval_metrics):
        """Return an aggregated metric (AUC) for evaluation."""
        total_num = sum([num for num, _ in eval_metrics])
        auc_aggregated = (
            sum([metrics["AUC"] * num for num, metrics in eval_metrics]) / total_num
        )
        metrics_aggregated = {"AUC": auc_aggregated}
        return metrics_aggregated

We use two clients for this example.
An :code:`evaluate_metrics_aggregation` function is defined to collect and wighted average the AUC values from clients.

Then, we start the server:

.. code-block:: python

    # Start Flower server
    fl.server.start_server(
        server_address="0.0.0.0:8080",
        config=fl.server.ServerConfig(num_rounds=num_rounds),
        strategy=strategy,
    )

Tree-based bagging aggregation
~~~~~~~~

You must be curious about how bagging aggregation works. Let's look into the details.

In file :code:`flwr.server.strategy.fedxgb_bagging.py`, we define :code:`FedXgbBagging` inherited from :code:`flwr.server.strategy.FedAvg`.
Then, we override the :code:`aggregate_fit`, :code:`aggregate_evaluate` and :code:`evaluate` methods as follows:

.. code-block:: python

    import json
    from logging import WARNING
    from typing import Any, Callable, Dict, List, Optional, Tuple, Union, cast

    from flwr.common import EvaluateRes, FitRes, Parameters, Scalar
    from flwr.common.logger import log
    from flwr.server.client_proxy import ClientProxy

    from .fedavg import FedAvg


    class FedXgbBagging(FedAvg):
        """Configurable FedXgbBagging strategy implementation."""

        def __init__(
            self,
            evaluate_function: Optional[
                Callable[
                    [int, Parameters, Dict[str, Scalar]],
                    Optional[Tuple[float, Dict[str, Scalar]]],
                ]
            ] = None,
            **kwargs: Any,
        ):
            self.evaluate_function = evaluate_function
            self.global_model: Optional[bytes] = None
            super().__init__(**kwargs)

        def aggregate_fit(
            self,
            server_round: int,
            results: List[Tuple[ClientProxy, FitRes]],
            failures: List[Union[Tuple[ClientProxy, FitRes], BaseException]],
        ) -> Tuple[Optional[Parameters], Dict[str, Scalar]]:
            """Aggregate fit results using bagging."""
            if not results:
                return None, {}
            # Do not aggregate if there are failures and failures are not accepted
            if not self.accept_failures and failures:
                return None, {}

            # Aggregate all the client trees
            global_model = self.global_model
            for _, fit_res in results:
                update = fit_res.parameters.tensors
                for bst in update:
                    global_model = aggregate(global_model, bst)

            self.global_model = global_model

            return (
                Parameters(tensor_type="", tensors=[cast(bytes, global_model)]),
                {},
            )

        def aggregate_evaluate(
            self,
            server_round: int,
            results: List[Tuple[ClientProxy, EvaluateRes]],
            failures: List[Union[Tuple[ClientProxy, EvaluateRes], BaseException]],
        ) -> Tuple[Optional[float], Dict[str, Scalar]]:
            """Aggregate evaluation metrics using average."""
            if not results:
                return None, {}
            # Do not aggregate if there are failures and failures are not accepted
            if not self.accept_failures and failures:
                return None, {}

            # Aggregate custom metrics if aggregation fn was provided
            metrics_aggregated = {}
            if self.evaluate_metrics_aggregation_fn:
                eval_metrics = [(res.num_examples, res.metrics) for _, res in results]
                metrics_aggregated = self.evaluate_metrics_aggregation_fn(eval_metrics)
            elif server_round == 1:  # Only log this warning once
                log(WARNING, "No evaluate_metrics_aggregation_fn provided")

            return 0, metrics_aggregated

        def evaluate(
            self, server_round: int, parameters: Parameters
        ) -> Optional[Tuple[float, Dict[str, Scalar]]]:
            """Evaluate model parameters using an evaluation function."""
            if self.evaluate_function is None:
                # No evaluation function provided
                return None
            eval_res = self.evaluate_function(server_round, parameters, {})
            if eval_res is None:
                return None
            loss, metrics = eval_res
            return loss, metrics

In :code:`aggregate_fit`, we sequentially aggregate the clients' XGBoost trees by calling :code:`aggregate()` function:

.. code-block:: python

    def aggregate(
        bst_prev_org: Optional[bytes],
        bst_curr_org: bytes,
    ) -> bytes:
        """Conduct bagging aggregation for given trees."""
        if not bst_prev_org:
            return bst_curr_org

        # Get the tree numbers
        tree_num_prev, _ = _get_tree_nums(bst_prev_org)
        _, paral_tree_num_curr = _get_tree_nums(bst_curr_org)

        bst_prev = json.loads(bytearray(bst_prev_org))
        bst_curr = json.loads(bytearray(bst_curr_org))

        bst_prev["learner"]["gradient_booster"]["model"]["gbtree_model_param"][
            "num_trees"
        ] = str(tree_num_prev + paral_tree_num_curr)
        iteration_indptr = bst_prev["learner"]["gradient_booster"]["model"][
            "iteration_indptr"
        ]
        bst_prev["learner"]["gradient_booster"]["model"]["iteration_indptr"].append(
            iteration_indptr[-1] + paral_tree_num_curr
        )

        # Aggregate new trees
        trees_curr = bst_curr["learner"]["gradient_booster"]["model"]["trees"]
        for tree_count in range(paral_tree_num_curr):
            trees_curr[tree_count]["id"] = tree_num_prev + tree_count
            bst_prev["learner"]["gradient_booster"]["model"]["trees"].append(
                trees_curr[tree_count]
            )
            bst_prev["learner"]["gradient_booster"]["model"]["tree_info"].append(0)

        bst_prev_bytes = bytes(json.dumps(bst_prev), "utf-8")

        return bst_prev_bytes


    def _get_tree_nums(xgb_model_org: bytes) -> Tuple[int, int]:
        xgb_model = json.loads(bytearray(xgb_model_org))
        # Get the number of trees
        tree_num = int(
            xgb_model["learner"]["gradient_booster"]["model"]["gbtree_model_param"][
                "num_trees"
            ]
        )
        # Get the number of parallel trees
        paral_tree_num = int(
            xgb_model["learner"]["gradient_booster"]["model"]["gbtree_model_param"][
                "num_parallel_tree"
            ]
        )
        return tree_num, paral_tree_num

In this function, we first fetch the number of trees and the number of parallel trees for the current and previous model
by calling :code:`_get_tree_nums`.
Then, the fetched information will be aggregated.
After that, the trees (containing model weights) are aggregated to generate a new tree model.

After traversal of all clients' models, a new global model is generated,
followed by the serialisation, and sending back to each client.


Launch Federated XGBoost!
---------------------------

With both client and server ready, we can now run everything and see federated
learning in action. FL systems usually have a server and multiple clients. We
therefore have to start the server first:

.. code-block:: shell

    $ python3 server.py

Once the server is running we can start the clients in different terminals.
Open a new terminal and start the first client:

.. code-block:: shell

    $ python3 client.py --node-id=0

Open another terminal and start the second client:

.. code-block:: shell

    $ python3 client.py --node-id=1

Each client will have its own dataset.
You should now see how the training does in the very first terminal (the one that started the server):

.. code-block:: shell

    INFO flwr 2023-11-20 11:21:56,454 | app.py:163 | Starting Flower server, config: ServerConfig(num_rounds=5, round_timeout=None)
    INFO flwr 2023-11-20 11:21:56,473 | app.py:176 | Flower ECE: gRPC server running (5 rounds), SSL is disabled
    INFO flwr 2023-11-20 11:21:56,473 | server.py:89 | Initializing global parameters
    INFO flwr 2023-11-20 11:21:56,473 | server.py:276 | Requesting initial parameters from one random client
    INFO flwr 2023-11-20 11:22:38,302 | server.py:280 | Received initial parameters from one random client
    INFO flwr 2023-11-20 11:22:38,302 | server.py:91 | Evaluating initial parameters
    INFO flwr 2023-11-20 11:22:38,302 | server.py:104 | FL starting
    DEBUG flwr 2023-11-20 11:22:38,302 | server.py:222 | fit_round 1: strategy sampled 2 clients (out of 2)
    DEBUG flwr 2023-11-20 11:22:38,636 | server.py:236 | fit_round 1 received 2 results and 0 failures
    DEBUG flwr 2023-11-20 11:22:38,643 | server.py:173 | evaluate_round 1: strategy sampled 2 clients (out of 2)
    DEBUG flwr 2023-11-20 11:22:38,653 | server.py:187 | evaluate_round 1 received 2 results and 0 failures
    DEBUG flwr 2023-11-20 11:22:38,653 | server.py:222 | fit_round 2: strategy sampled 2 clients (out of 2)
    DEBUG flwr 2023-11-20 11:22:38,721 | server.py:236 | fit_round 2 received 2 results and 0 failures
    DEBUG flwr 2023-11-20 11:22:38,745 | server.py:173 | evaluate_round 2: strategy sampled 2 clients (out of 2)
    DEBUG flwr 2023-11-20 11:22:38,756 | server.py:187 | evaluate_round 2 received 2 results and 0 failures
    DEBUG flwr 2023-11-20 11:22:38,756 | server.py:222 | fit_round 3: strategy sampled 2 clients (out of 2)
    DEBUG flwr 2023-11-20 11:22:38,831 | server.py:236 | fit_round 3 received 2 results and 0 failures
    DEBUG flwr 2023-11-20 11:22:38,868 | server.py:173 | evaluate_round 3: strategy sampled 2 clients (out of 2)
    DEBUG flwr 2023-11-20 11:22:38,881 | server.py:187 | evaluate_round 3 received 2 results and 0 failures
    DEBUG flwr 2023-11-20 11:22:38,881 | server.py:222 | fit_round 4: strategy sampled 2 clients (out of 2)
    DEBUG flwr 2023-11-20 11:22:38,960 | server.py:236 | fit_round 4 received 2 results and 0 failures
    DEBUG flwr 2023-11-20 11:22:39,012 | server.py:173 | evaluate_round 4: strategy sampled 2 clients (out of 2)
    DEBUG flwr 2023-11-20 11:22:39,026 | server.py:187 | evaluate_round 4 received 2 results and 0 failures
    DEBUG flwr 2023-11-20 11:22:39,026 | server.py:222 | fit_round 5: strategy sampled 2 clients (out of 2)
    DEBUG flwr 2023-11-20 11:22:39,111 | server.py:236 | fit_round 5 received 2 results and 0 failures
    DEBUG flwr 2023-11-20 11:22:39,177 | server.py:173 | evaluate_round 5: strategy sampled 2 clients (out of 2)
    DEBUG flwr 2023-11-20 11:22:39,193 | server.py:187 | evaluate_round 5 received 2 results and 0 failures
    INFO flwr 2023-11-20 11:22:39,193 | server.py:153 | FL finished in 0.8905023969999988
    INFO flwr 2023-11-20 11:22:39,193 | app.py:226 | app_fit: losses_distributed [(1, 0), (2, 0), (3, 0), (4, 0), (5, 0)]
    INFO flwr 2023-11-20 11:22:39,193 | app.py:227 | app_fit: metrics_distributed_fit {}
    INFO flwr 2023-11-20 11:22:39,193 | app.py:228 | app_fit: metrics_distributed {'AUC': [(1, 0.7572), (2, 0.7705), (3, 0.77595), (4, 0.78), (5, 0.78385)]}
    INFO flwr 2023-11-20 11:22:39,193 | app.py:229 | app_fit: losses_centralized []
    INFO flwr 2023-11-20 11:22:39,193 | app.py:230 | app_fit: metrics_centralized {}

Congratulations!
You've successfully built and run your first federated XGBoost system.
The AUC values can be checked in :code:`metrics_distributed`.
One can see that the average AUC increases over FL rounds.

The full `source code <https://github.com/adap/flower/blob/main/examples/xgboost-quickstart/>`_ for this example can be found in :code:`examples/xgboost-quickstart`.


Comprehensive Federated XGBoost
---------------------------

Now that you have known how federated XGBoost work with Flower, it's time to run some more comprehensive experiments by customising the experimental settings.
In the xgboost-comprehensive example (`full code <https://github.com/adap/flower/tree/main/examples/xgboost-comprehensive>`_),
we provide more options to define various experimental setups, including data partitioning and centralised/distributed evaluation.
Let's take a look!

Customised data partitioning
~~~~~~~~

In :code:`dataset.py`, we have a function :code:`instantiate_partitioner` to instantiate the data partitioner
based on the given :code:`num_partitions` and :code:`partitioner_type`.
Currently, we provide four supported partitioner type to simulate the uniformity/non-uniformity in data quantity (uniform, linear, square, exponential).

.. code-block:: python

    from flwr_datasets.partitioner import (
        IidPartitioner,
        LinearPartitioner,
        SquarePartitioner,
        ExponentialPartitioner,
    )

    CORRELATION_TO_PARTITIONER = {
        "uniform": IidPartitioner,
        "linear": LinearPartitioner,
        "square": SquarePartitioner,
        "exponential": ExponentialPartitioner,
    }


    def instantiate_partitioner(partitioner_type: str, num_partitions: int):
        """Initialise partitioner based on selected partitioner type and number of
        partitions."""
        partitioner = CORRELATION_TO_PARTITIONER[partitioner_type](
            num_partitions=num_partitions
        )
        return partitioner


Customised centralised/distributed evaluation
~~~~~~~~

To facilitate centralised evaluation, we define a function in :code:`server.py`:

.. code-block:: python

    def get_evaluate_fn(test_data):
        """Return a function for centralised evaluation."""

        def evaluate_fn(
            server_round: int, parameters: Parameters, config: Dict[str, Scalar]
        ):
            # If at the first round, skip the evaluation
            if server_round == 0:
                return 0, {}
            else:
                bst = xgb.Booster(params=params)
                for para in parameters.tensors:
                    para_b = bytearray(para)

                # Load global model
                bst.load_model(para_b)
                # Run evaluation
                eval_results = bst.eval_set(
                    evals=[(test_data, "valid")],
                    iteration=bst.num_boosted_rounds() - 1,
                )
                auc = round(float(eval_results.split("\t")[1].split(":")[1]), 4)
                log(INFO, f"AUC = {auc} at round {server_round}")

                return 0, {"AUC": auc}

        return evaluate_fn

This function returns a evaluation function which instantiates a :code:`Booster` object and loads the global model weights to it.
The evaluation is conducted by calling :code:`eval_set()` method, and the tested AUC value is reported.

As for distributed evaluation on the clients, it's same as the quick-start example by
overriding the :code:`evaluate()` method insides the :code:`XgbClient` class in :code:`client.py`.

Arguments parser
~~~~~~~~

In :code:`utils.py`, we define the arguments parsers for clients and server, allowing users to specify different experimental settings.
Let's first see the sever side:

.. code-block:: python

    import argparse


    def server_args_parser():
        """Parse arguments to define experimental settings on server side."""
        parser = argparse.ArgumentParser()

        parser.add_argument(
            "--pool-size", default=2, type=int, help="Number of total clients."
        )
        parser.add_argument(
            "--num-rounds", default=5, type=int, help="Number of FL rounds."
        )
        parser.add_argument(
            "--num-clients-per-round",
            default=2,
            type=int,
            help="Number of clients participate in training each round.",
        )
        parser.add_argument(
            "--num-evaluate-clients",
            default=2,
            type=int,
            help="Number of clients selected for evaluation.",
        )
        parser.add_argument(
            "--centralised-eval",
            action="store_true",
            help="Conduct centralised evaluation (True), or client evaluation on hold-out data (False).",
        )

        args = parser.parse_args()
        return args

This allows user to specify the number of total clients / FL rounds / participating clients / clients for evaluation,
and evaluation fashion. Note that with :code:`--centralised-eval`, the sever will do centralised evaluation
and all functionalities for client evaluation will be disabled.

Then, the argument parser on client side:

.. code-block:: python

    def client_args_parser():
        """Parse arguments to define experimental settings on client side."""
        parser = argparse.ArgumentParser()

        parser.add_argument(
            "--num-partitions", default=10, type=int, help="Number of partitions."
        )
        parser.add_argument(
            "--partitioner-type",
            default="uniform",
            type=str,
            choices=["uniform", "linear", "square", "exponential"],
            help="Partitioner types.",
        )
        parser.add_argument(
            "--node-id",
            default=0,
            type=int,
            help="Node ID used for the current client.",
        )
        parser.add_argument(
            "--seed", default=42, type=int, help="Seed used for train/test splitting."
        )
        parser.add_argument(
            "--test-fraction",
            default=0.2,
            type=float,
            help="Test fraction for train/test splitting.",
        )
        parser.add_argument(
            "--centralised-eval",
            action="store_true",
            help="Conduct centralised evaluation (True), or client evaluation on hold-out data (False).",
        )

        args = parser.parse_args()
        return args

This defines various options for client data partitioning.
Besides, clients also have a option to conduct evaluation on centralised test set by setting :code:`--centralised-eval`.

Example commands
~~~~~~~~

To run a centralised evaluated experiment on 5 clients with exponential distribution for 50 rounds,
we first start the server as below:

.. code-block:: shell

    $ python3 server.py --pool-size=5 --num-rounds=50 --num-clients-per-round=5 --centralised-eval

Then, on each client terminal, we start the clients:

.. code-block:: shell

    $ python3 clients.py --num-partitions=5 --partitioner-type=exponential --node-id=NODE_ID

The full `source code <https://github.com/adap/flower/blob/main/examples/xgboost-comprehensive/>`_ for this comprehensive example can be found in :code:`examples/xgboost-comprehensive`.
