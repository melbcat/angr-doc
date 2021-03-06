# Analyses

Angr's goal is to make it easy to carry out useful analyses on binary programs.
These analyses might be complicated to run, so angr makes sure to save them when a project is saved and load them when it is loaded.
This section will discuss how to run and create these analyses.

## Built-in Analyses

Angr comes with several built-in analyses:

| Name | Description |
|------|-------------|
| CFG  | Constructs a *Control Flow Graph* of the program. The results are accessible via `b.analyses.CFG()`. |
| VFG  | Performs VSA on every function of the program, creating a *Value Flow Graph* and detecting stack variables. |
| DDG  | Calculates a data dependency graph, allowing one to determine what statements a given value depends on. |

### CFG

A basic analysis that one might carry out on a binary is a Control Flow Graph.
A CFG is a graph with (conceptually) basic blocks as nodes and jumps/calls/rets/etc as edges.

A CFG can be constructed by doing:

```python
# load your project
b = angr.Project('/path/to/bin')

# generate a CFG
cfg = b.analyses.CFG()
```

Of course, there are several options for customized CFGs.

| Option | Description |
|--------|-------------|
| context_sensitivity_level | This sets the context sensitivity level of the analysis. See the context sensitivity level section below for more information. This is 1 by default. |
| start, starts | An address, or, for `starts`, a list of addresses, to use as entry points into the analysis. |
| avoid_runs | A list of addresses to ignore in the analysis. |
| call_depth | Limit the depth of the analysis to some number calls. This is useful for checking which functions a specific function can directly jump to (by setting `call_depth` to 1).
| initial_state | An initial state can be provided to the CFG, which it will use throughout its analysis. |
| keep_input_state | To save memory, the state at each basic block is discarded by default. If `keep_input_state` is True, the state is saved in the CFGNode. |

### Context Sensitivity Level

Angr constructs a CFG by executing every basic block and seeing where it goes.
This introduces some challenges: a basic block can act differently in different *contexts*.
For example, if a block ends in a function return, the target of that return will be different, depending on different callers of the function containing that basic block.

The context sensitivity level is, conceptually, the number of such callers to keep on the callstack.
To explain this concept, let's look at the following code:

```c
void error(char *error)
{
	puts(error);
}

void alpha()
{
	puts("alpha");
	error("alpha!");
}

void beta()
{
	puts("beta");
	error("beta!");
}

void main()
{
	alpha();
	beta();
}
```

The above sample has four call chains: `main>alpha>puts`, `main>alpha>error>puts` and `main>beta>puts`, and `main>beta>error>puts`.
While, in this case, angr can probably execute both call chains, this becomes unfeasible for larger binaries.
Thus, angr executes the blocks with states limited by the context sensitivity level.
That is, each function is re-analyzed for each unique context that it is called in.

For example, the `puts()` function above will be analyzed with the following contexts, given different context sensitivity levels:

| Level | Meaning | Contexts |
|-------|---------|----------|
| 0 | Callee-only | `puts` |
| 1 | One caller, plus callee | `alpha>puts` `beta>puts` `error>puts` |
| 2 | Two callers, plus callee | `alpha>error>puts` `main>alpha>puts` `beta>error>puts` `main>beta>puts` |
| 3 | Three callers, plus callee | `main>alpha>error>puts` `main>alpha>puts` `main>beta>error>puts` `main>beta>puts` |

The upside of increasing the context sensitivity level is that more information can be gleamed from the CFG.
For example, with context sensitivity of 1, the CFG will show that, when called from `alpha`, `puts` returns to `alpha`, when called from `error`, `puts` returns to `error`, and so forth.
With context sensitivity of 0, the CFG simply shows that `puts` returns to `alpha`, `beta`, and `error`.
This, specifically, is the context sensitivity level used in IDA.
The downside of increasing the context sensitivity level is that it exponentially increases the analysis time.

### Using the CFG

The CFG, at its core, is a [NetworkX](https://networkx.github.io/) di-graph.
This means that all of the normal NetworkX APIs are available:

```python
print "This is the graph:", cfg.graph
print "It has %d nodes and %d edges" % (len(cfg.graph.nodes()), len(cfg.graph.edges()))
```

The nodes of the CFG graph are instances of class `CFGNode`.
Due to context sensitivity, a given basic block can have multiple nodes in the graph (for multiple contexts).

```python
# this grabs *any* node at a given location:
food_node = cfg.get_any_node(0xf00d)

# on the other hand, this grabs all of the nodes
print "There were %d contexts for the 0xf00d block" % len(cfg.get_all_nodes(0xf00d))

# if keep_input_states was given as True, we can also retrieve the actual SimIRSBs
print "A single SimIRSB at 0xf00d:", cfg.get_any_irsb(0xf00d)
print "All SimIRSBs at 0xf00d:", cfg.get_all_irsbs(0xf00d)

# we can also look up predecessors and successors
print "Predecessors of 0xf00d:" [ node.addr for node in cfg.get_predecessors(food_node) ]
print "Successors of 0xf00d:" [ node.addr for node in cfg.get_successors(food_node) ]
print "Successors (and type of jump) of 0xf00d:" [ jumpkind + " to " + str(node.addr) for node,jumpkind in cfg.get_successors_and_jumpkind(food_node) ]
```

## Function Manager

TODO

### VFG

TODO

### DDG

TODO

## Running Analyses

Now that you understand how to load binaries in angr, and have some idea of angr's internals, we can discuss how to carry out analyses!
Angr provides a standardized interface to perform analyses. Specifically, it is:


Results of analyses can be accessed by using the `Project.results` attribute.
If an analysis that hasn't been run is accessed, it will be automatically run with default options.
For example:

```python
cfg = b.analyses.CFG()
assert cfg is b.results.CFG

print "About to run the VSA analysis and create a VFG!"
print b.results.VFG
```

### Resilience

Analyses can be written to be resilient, and catch and log basically any error.
These errors, depending on how they're caught, are logged to the `errors` or `named_errors` attribute of the analysis.
However, you might want to run an analysis in "fail fast" mode, so that errors are not handled.
To do this, the `fail_fast` keyword argument can be passed into `analyze`.

```python
b.analyses.CFG(fail_fast=True)
```

## Creating Analyses

An analysis can be created by subclassing the `Analysis` class.
In this section, we'll create a mock analysis to show off the various features.
Let's start with something simple:

```python
class MockAnalysis(angr.Analysis):
	def __init__(self, option):
		self.option = option
```

This is a quite simple analysis -- it takes an option, and stores it.
Of course, it's not useful, but what can you do?
Let's see how to call:

```python
# you will have to reinitialize the project to access the new class
b = angr.Project("path/to/bin")
mock = b.analyses.MockAnalysis('this is my option')
assert mock.option == 'this is my option'
```

### Naming Analyses

By default, an analysis is named the same as the class.
However, sometimes you might want a shorter name.
You can do this by defining a `__name__` attribute:

```python
class FunctionBlockAverage(angr.Analysis):
	__name__ = 'FuncSize'

	def __init__(self):
		self._cfg = self._p.analyses.CFG()
		self.avg = len(self._cfg.nodes()) / len(self._cfg.function_manager.functions)
```

After this, you can call this analysis using it's specified name. For example, `b.analyses.FuncSize()`.

### Analysis Resilience

Sometimes, your (or our) code might suck and analyses might throw exceptions.
We understand, and we also understand that oftentimes, a partial result is better than nothing.
This is specifically true when, for example, running an analysis on all of the functions of a class.
Even if some of the functions fails, we still want to know the results of the functions that do not.

To facilitate this, the `Analysis` base class provides a resilience context manager.
Here's an example:

```python
class ComplexFunctionAnalysis(angr.Analysis):
	def __init__(self):
		self._cfg = self._p.analyses.CFG()
		self.results = { }
		for addr,func in self._cfg.function_manager.functions.items():
			with self._resilience():
				if addr % 2 == 0:
					raise ValueError("can't handle functions at even addresses")
				else:
					self.results[addr] = "GOOD"
```

The context manager catches any exceptions thrown and logs them (as a tuple of the exception type, message, and traceback) to `self.errors`.
These are also saved and loaded when the analysis is saved and loaded (although the traceback is discarded, as it is not picklable).
`Analysis._resilience()` takes two optional keyword parameters.
The first is `name`, which is the name of the potential error.
If `name` is provided to `_resilience`, the error is placed in `self.named_errors[name]`.
The second argument is `exception`, which should be the type of the exception that `_resilience` should catch.
This defaults to `Exception`, which handles (and logs) almost anything that could go wrong.
You can also pass a tuple of exception types to this option, in which case all of them will be caught.

Using `_resilience` has a few advantages:

1. Your exceptions are gracefully logged and easily accessible afterwards. This is really nice for writing testcases.
2. When creating your analysis, the user can pass `fail_fast=True`, which transparently disable the resilience, which is really nice for manual testing.
3. It's prettier than having `try`/`except` everywhere.

### Analysis Functions

You might not be into the whole OO thing.
We still got your back.
You can implement analyses as functions and register them manually, like so.
Unfortunately, if you choose this route, you must handle all the options that angr pushes through the analyses (which are otherwise transparently handled).

```python
# define your function
def block_counter(project, deps, fail_fast, min_addr=0, max_addr=0xffffffff):
	return len([ irsb for irsb in deps[0].cfg.nodes if irsb.addr >= min_addr and irsb.addr < max_addr ])

# register the analysis
angr.registered_analyses['blocks'] = block_counter

# reinitialize the project
b = angr.Project("/path/to/bin")

# and run them!
b.analyses.blocks(min_addr=0x100, max_addr=0x400000)
```

However, this doesn't provide any resilience and so forth.
