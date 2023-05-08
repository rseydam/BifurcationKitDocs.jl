# Event Handling

`BifurcationKit.jl` allows the detection of events along the branch of solutions. Its main use consists in detecting bifurcation points but they can be used and combined together by the user too.

The events are detected during a call to `br = continuation(prob, alg, contParams::ContinuationPar;kwargs...)` by turning on the following flag(s):

- `contParams.detectEvent = 1`

The event points are located by looking at the function defining the event (see below). The located event points are then returned in `br.specialpoint`.

## Precise detection of event points using Bisection

Note that the event points detected when `detectEvent = 1` are only approximate event points. Indeed, we only signal that, in between two continuation steps which can be large, a (several) event point has been detected. Hence, we only have a rough idea of where the event is located, unless your `ContinuationPar().dsmax` is very small... This can be improved as follows.

If you choose `detectEvent = 2`, a bisection algorithm is used to locate the event points more precisely. It means that we recursively track down the event. Some options in `ContinuationPar` control this behavior:

- `nInversion`: number of sign inversions in the bisection algorithm
- `maxBisectionSteps` maximum number of bisection steps
- `tolParamBisectionEvent` tolerance on parameter to locate event

## Different event types

The set of possible events `DiscreteEvent, ContinuousEvent, SetOfEvents, PairOfEvents` is detailed in the [Library](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/library/#Events-1).

## Built-in events

```@docs
BifurcationKit.SaveAtEvent
```

```@docs
BifurcationKit.FoldDetectEvent
```

```@docs
BifurcationKit.BifDetectEvent
```


## Examples

We show how to use the different events. We first set up a problem as usual.

```@example EVENT
using Revise, BifurcationKit, Setfield, ForwardDiff
using Plots
const BK = BifurcationKit
####################################################################################################
# test vector field for event detection
function Feve(X, p)
	p1, p2, k = p
	x, y = X
	out = similar(X)
	out[1] = p1 + x - y - x^k/k
	out[2] = p1 + y + x - 2y^k/k
	out
end

# parameters for the vector field
par = (p1 = -3., p2=-3., k=3)

# bifurcation problem
prob = BifurcationProblem(Feve, -2ones(2), par, (@lens _.p1);
	recordFromSolution = (x,p) -> x[1])

# parameters for the continuation
opts = ContinuationPar(dsmax = 0.1, ds = 0.001, maxSteps = 128, pMin = -3., pMax = 4.0, plotEveryStep = 10,
     newtonOptions = NewtonPar(tol = 1e-10, verbose = false, maxIter = 5),
     # parameters specific to event detection
     detectBifurcation = 0, detectEvent = 2, nInversion = 6, dsminBisection = 1e-9,
     maxBisectionSteps = 15, detectFold=false)

# arguments for continuation
args = (prob, PALC(bls = MatrixBLS()), opts)
kwargs = (plot = false, verbosity = 0,)
nothing #hide
```

### Example of continuous event

In this first example, we build an event to detect when the parameter value is `-2` or when the first component of the solution is `1`.

```@example EVENT
br = continuation(args...; kwargs...,
	event = BK.ContinuousEvent(2, 
		(iter, state) -> (getp(state)+2, getx(state)[1]-1)),)
nothing #hide		
```

gives

```@example EVENT
br
```

This shows for example that the first component of the event was detected `userC-1` first. This yields

```@example EVENT
plot(br)
```

You can also name the events as follows

```@example EVENT
 br = continuation(args...; kwargs...,
 	event = BK.ContinuousEvent(2, 
 		(iter, state) -> (getp(state)+2, getx(state)[1]-1),
 		("event1", "event2")))
nothing #hide 		
```

And get:

```@example EVENT
br
```

```@example EVENT
plot(br)
```



### Example of discrete event

You can also use discrete events to detect a change. For example, the following detect when the parameter value equals `-2`:

```@example EVENT
br = continuation(args...; kwargs...,
	event = BK.DiscreteEvent(1, 
		(iter, state) -> getp(state)>-2))
```

### Example of PairOfEvents event

Let us be a bit more creative and combine a continuous event with a discrete one:

```@example EVENT
br = continuation(args...; kwargs...,
	event = BK.PairOfEvents(
		BK.ContinuousEvent(1, (iter, state) -> getp(state)),
		BK.DiscreteEvent(1, (iter, state) -> getp(state)>-2)))
```

Here `userD-1` means that the first component of the discrete event was detected. Of course, you can name the event like done above.

### Example of set of events
We can combine more events and chain them like we want using `SetOfEvents`. In this example, we show how to do bifurcation detection and event location altogether:

```@example EVENT		
ev1 = BK.ContinuousEvent(1, (iter, state) -> getp(state)-1)
ev2 = BK.ContinuousEvent(2, (iter, state) -> (getp(state)-2, getp(state)-2.5))
# event to detect bifurcation
ev3 = BK.BifDetectEvent
# we combine the events together
eve = BK.SetOfEvents(ev1, ev2, ev3)

br = continuation(args...; kwargs...,
		event = eve)
```

Which gives

```@example EVENT
plot(br)
```
