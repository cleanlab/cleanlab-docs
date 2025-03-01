.. _issue_manager_creating_your_own:

Creating Your Own Issues Manager
================================



This guide walks through the process of creating creating your own 
:py:class:`IssueManager <cleanlab.datalab.issue_manager.issue_manager.IssueManager>`
to detect a custom-defined type of issue alongside the pre-defined issue types in 
:py:class:`Datalab <cleanlab.datalab.datalab.Datalab>`.

.. seealso::

    - :py:meth:`register <cleanlab.datalab.factory.register>`:
        You can either use this function at runtime to register a new issue manager:

        .. code-block:: python

            from cleanlab.datalab.factory import register
            register(MyIssueManager)

        or add as a decorator to the class definition:

        .. code-block:: python

            @register
            class MyIssueManager(IssueManager):
                ...

Prerequisites
-------------

As a starting point for this guide, we'll import the necessary things for the next section and create a dummy dataset.

.. note::

    .. include:: ../optional_dependencies.rst

.. code-block:: python


    import numpy as np
    import pandas as pd
    from cleanlab import IssueManager

    # Create a dummy dataset
    N = 20
    data = pd.DataFrame(
        {
            "text": [f"example {i}" for i in range(N)],
            "label": np.random.randint(0, 2, N),
        },
    )


Implementing IssueManagers
--------------------------

.. _basic_issue_manager:

Basic Issue Check
~~~~~~~~~~~~~~~~~


To create a basic issue manager, inherit from the
:py:class:`IssueManager <cleanlab.datalab.issue_manager.issue_manager.IssueManager>` class,
assign a name to the class as the class-variable, `issue_name`, and implement the
:py:meth:`find_issues <cleanlab.datalab.issue_manager.issue_manager.IssueManager.find_issues>` method.

The :py:meth:`find_issues <cleanlab.datalab.issue_manager.issue_manager.IssueManager.find_issues>`
method should mark each example in the dataset as an issue or not with a boolean array.
It should also provide a score for each example in the dataset that quantifies the quality of the example
with regards to the issue.

.. code-block:: python

    class Basic(IssueManager):
        # Assign a name to the issue
        issue_name = "basic"
        def find_issues(self, **kwargs) -> None:
            # Compute scores for each example
            scores = np.random.rand(len(self.datalab.data))

            # Construct a dataframe where examples are marked for issues
            # and the score for each example is included. 
            self.issues = pd.DataFrame(
                {
                    f"is_{self.issue_name}_issue" : scores < 0.1,
                    self.issue_score_key : scores,
                },
            )

            # Score the dataset as a whole based on this issue type
            self.summary = self.make_summary(score = scores.mean())


.. _intermediate_issue_manager:

Intermediate Issue Check
~~~~~~~~~~~~~~~~~~~~~~~~


To create an intermediate issue:

- Perform the same steps as in the :ref:`basic issue check <basic_issue_manager>` section.
- Populate the `info` attribute with a dictionary of information about the identified issues.

The information can be included in a report generated by :py:class:`Datalab <cleanlab.datalab.datalab.Datalab>`,
if you add any of the keys to the `verbosity_levels` class-attribute.
Optionally, you can also add a description of the type of issue this issue manager handles to the `description` class-attribute.

.. code-block:: python

    class Intermediate(IssueManager):
        issue_name = "intermediate"
        # Add a dictionary of information to include in the report
        verbosity_levels = {
            0: [],
            1: ["std"],
            2: ["raw_scores"],
        }
        # Add a description of the issue
        description = "Intermediate issues are a bit more involved than basic issues."
        def find_issues(self, *, intermediate_arg: int, **kwargs) -> None:
            N = len(self.datalab.data)
            raw_scores = np.random.rand(N)
            std = raw_scores.std()
            threshold = min(0, raw_scores.mean() - std)
            sin_filter = np.sin(intermediate_arg * np.arange(N) / N)
            kernel = sin_filter ** 2
            scores = kernel * raw_scores
            self.issues = pd.DataFrame(
                {
                    f"is_{self.issue_name}_issue" : scores < threshold,
                    self.issue_score_key : scores,
                },
            )
            self.summary = self.make_summary(score = scores.mean())

            # Useful information that will be available in the Datalab instance
            self.info = {
                "std": std,
                "raw_scores": raw_scores,
                "kernel": kernel,
            }

Advanced Issue Check
~~~~~~~~~~~~~~~~~~~~

.. note::

    WIP: This section is a work in progress.



Use with Datalab
----------------

We can create a
:py:class:`Datalab <cleanlab.datalab.datalab.Datalab>`
instance and run issue checks with the custom issue managers we created like so:


.. code-block:: python

    from cleanlab.datalab.factory import register
    from cleanlab import Datalab


    # Register the issue manager
    for issue_manager in [Basic, Intermediate]:
        register(issue_manager)

    # Instantiate a datalab instance
    datalab = Datalab(data, label_name="label")

    # Run the issue check
    issue_types = {"basic": {}, "intermediate": {"intermediate_arg": 2}}
    datalab.find_issues(issue_types=issue_types)

    # Print report
    datalab.report(verbosity=0)


The report will look something like this:

.. code-block:: text

    Here is a summary of the different kinds of issues found in the data:

      issue_type     score  num_issues
           basic  0.477762           2
    intermediate  0.286455           0

    (Note: A lower score indicates a more severe issue across all examples in the dataset.)


    ------------------------------------------- basic issues -------------------------------------------

    Number of examples with this issue: 2
    Overall dataset quality in terms of this issue: 0.4778

    Examples representing most severe instances of this issue:
        is_basic_issue  basic_score
    13            True     0.003042
    8             True     0.058117
    11           False     0.121908
    15           False     0.169312
    17           False     0.229044


    --------------------------------------- intermediate issues ----------------------------------------

    About this issue:
    	Intermediate issues are a bit more involved than basic issues.

    Number of examples with this issue: 0
    Overall dataset quality in terms of this issue: 0.2865

    Examples representing most severe instances of this issue:
        is_intermediate_issue  intermediate_score    kernel
    0                   False            0.000000       0.0
    1                   False            0.007059  0.009967
    3                   False            0.010995  0.087332
    2                   False            0.016296   0.03947
    11                  False            0.019459  0.794251
