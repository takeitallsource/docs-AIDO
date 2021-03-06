# API for solution {#api-solution status=draft}


## Examples {#api-solution-examples}

We have several example repositories that show different aspects
of the API.

These are:

- [`challenge-aido1_luck`][challenge-aido1_luck]: the simplest possible challenge;
- [`challenge-aido1_log_processing`][challenge-aido1_log_processing]: a test for log processing;
- [`challenge-aido1_dummy_sim`][challenge-aido1_dummy_sim]: a test for the gym simulation environment;
- [`challenge-aido1_test_multistep`][challenge-aido1_test_multistep]: shows the multi-step logic.


[challenge-aido1_luck]: https://github.com/duckietown/challenge-aido1_luck
[challenge-aido1_log_processing]: https://github.com/duckietown/challenge-aido1_log_processing
[challenge-aido1_dummy_sim]: https://github.com/duckietown/challenge-aido1_dummy_sim
[challenge-aido1_test_multistep]: https://github.com/duckietown/challenge-aido1_test_multistep


## Installing the API

The API is implemented in the [`duckietown-challenges` repository][duckietown-challenges].

[duckietown-challenges]: https://github.com/duckietown/duckietown-challenges

The suggested way to install it is to specify it as a `requirements.txt` as follows,
with a pinned version/branch `v3`.

```
-e git://github.com/duckietown/duckietown-shell.git@v3#egg=duckietown-shell
-e git://github.com/duckietown/duckietown-challenges.git@v3#egg=duckietown-challenges
```

The tag `v3` specifies the current protocol of client and servers and makes sure that
we can have forward-compatibility.

In the Dockerfile you can then do the following:

```Dockerfile
COPY requirements.txt /project/requirements.txt
RUN pip install -r /project/requirements.txt
```

[See here](https://github.com/duckietown/challenge-aido1_luck/tree/v3/submission-random) 
for an example of this mechanism.


## Solution entry point 

Each submission container should have as the `CMD` a script with the following pattern:

```python
#!/usr/bin/env python

from duckietown_challenges import wrap_solution, ChallengeSolution, ChallengeInterfaceSolution

class MySolution(ChallengeSolution):
    def run(self, cis):
        assert isinstance(cis, ChallengeInterfaceSolution)
        
        # run your script here
        myscript()
    
        # make this call to ma    
        cis.set_solution_output_dict({})


if __name__ == '__main__':
    wrap_solution(MySolution())

```

This structure makes sure that behind the scenes lots of stuff that is supposed
to happen happens hidden to you. This includes things like coordinating between the evaluation and 
solution container and being able to save the resulting artefacts.

In particular, what each solution needs to do is:

1. Implement the interface `ChallengeSolution`, which requires to implement a method `run()`.
2. Call `wrap_solution()` to with an instance of the class. The class will be called with an instance of `cis` of `ChallengeInterfaceSolution`, which provides several utility functions. 
3. Make at least one call to `cis.set_solution_output_dict()`. The parameter is a dictionary that can be passed to evaluator; you can leave it empty for now.


In most cases, you can just copy the structure above and fill in `myscript()` with your solution logic.

Each challenge will have different interaction protocols, explained in the next part.


## `ChallengeInterfaceSolution` API {#api-solution-desc}

Each solution is passed a `ChallengeInterfaceSolution` object as parameter to `run()`.

This section describes the most important methods of the API.
The all API is described in [the source code here][source].

[source]: https://github.com/duckietown/duckietown-challenges/blob/v3/src/duckietown_challenges/solution_interface.py

### Logging  {#api-solution-logging}

Use the following methods to log messages:

```python
def run(self, cis):
    cis.info('Information')
    cis.debug('Debug message')
    cis.error('Error message')
```

### Temporary files {#api-solution-temp}

If you need a temporary dir, you can use the method `get_tmp_dir():`

```python
d = cis.get_tmp_dir()
fn = os.path.join(d, 'tmp')
with open(fn, 'w') as f:
    f.write(data)
```

### Declaring failure {#api-solution-failure}

There is no shame in declaring failure early:

```python
cis.declare_failure("I give up")
```
        
### Getting challenge parameters and files {#api-solution-parameters}

Certain challenges require that the solution has access to certain files and parameters.

The semantics of these will vary from challenge to challenge.

For parameters, the API allows the evaluator to pass a dictionary, which can then
be recovered using the `get_challenge_parameters()` function:

```python
# assuming that the parameter "param1" has been set by the evaluator
params_from_evaluator = cis.get_challenge_parameters()
param1 = params_from_evaluator['param1']
```

For files, the API has the function `get_challenge_file()`:

```python
# assuming that the file "log.bag" has been passed by the evaluator
full_path = cis.get_challenge_file("log.bag")
bag = Bag(full_path)
```    

### Producing output {#api-solution-output}

Symmetrically, the API allows the solution to produce output 
that can be read by the evaluator.

There is a function `set_solution_output_dict()` that allows the solution
to pass one dictionary back to the evaluator:

```python
response = {'guess': 42}
cis.set_solution_output_dict(response)
```

There is a function `set_solution_output_file(basename, path)` that allows 
the solution to create a file that can be read by the evaluator
as well as to the user:

```python

output_file = ... # produced output
cis.set_solution_output_file("output.bag", output_file)

```

The method `set_solution_output_file_from_data()` allows to pass
directly the contents of the file.



### (Advanced) Reading output of previous steps {#api-solution-output-previous}

Some challenges have more than one evaluation step.

In this case, there is an API part that allows to read the output of the
previous steps.

The function `get_current_step()` returns the name of the current step.

The function `get_completed_steps()` returns the names of the completed steps.

The function `get_completed_step_solution_file(step_name, basename)`
returns a file created in a previous step.

Suppose now that there is a challenge with steps `step1` and `step2`
in which we need to pass some data. For example,  `step1`
might be a learning step, and `step2` might be an evaluation step.

 Using the functions above all together, we can have the following logic, in which 
the second step reads the output of the first step.

```python

step_name = cis.get_current_step()

if step_name == 'step1':
    learned_model_filename = ... 
    cis.set_solution_output_file('model', learned_model_filename)

if step_name == 'step2':
    # we know that step1 must have been successful
    assert cis.get_completed_steps() == ['step1']
    learned_model_filename = cis.get_solution_output_file('model')
    
```

Note that, transparently to the user, the two steps might have run *on different machines*.

Behind the scenes, the evaluator for `step1` has saved the data to S3, the challenge
server has kept track of the artefacts, and the evaluator for `step2` has downloaded
the data from S3. All of this is transparent to the user.


[See here for an example of a multi-step challenge.][challenge-aido1_test_multistep]


