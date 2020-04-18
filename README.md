# Probabilistic Resource Adequacy Suite

[![Build Status](https://travis-ci.org/NREL/PRAS.svg?branch=master)](https://travis-ci.org/NREL/PRAS)
[![Coverage Status](https://coveralls.io/repos/github/NREL/PRAS/badge.svg?branch=master)](https://coveralls.io/github/NREL/PRAS?branch=master)

The Probabilistic Resource Adequacy Suite (PRAS) is a collection of tools for
resource adequacy analysis of bulk power systems.

## Unleash your CPU cores

First, know that PRAS uses multi-threading, so be
sure to set the environment variable controlling the number of threads
available to Julia (36 in this Bash example, which is a good choice for
Eagle nodes - on a laptop you would probably only want 4 or so) before running
scripts or launching the REPL:

```sh
export JULIA_NUM_THREADS=36
```

## Installation

Installation requires Julia 1.3+ and the NREL package registry:

```
(v1.3) pkg> registry add https://github.com/NREL/JuliaRegistry.git
(v1.3) pkg> add PRAS
```

## Power System Data

The recommended way to store and retreive PRAS system data is in an HDF5 file
that conforms to the [PRAS system data format](SystemModel_HDF5_spec.md). Once
your system is represented in that format you can load it into PRAS with:

```julia
using PRAS
sys = SystemModel("filepath/to/systemdata.pras")
```

## Resource Adequacy Assessment

PRAS functionality is distributed across groups of
modular specifications that can be mixed, extended, or replaced to support the needs
of a particular analysis. When assessing reliability or capacity value, one can
define the specs to be used while passing along any associated parameters
or options.

The categories of specifications are:

**Simulation Specifications**: How should power system operations be simulated?
Options are `Classic` (convolution) and `Modern` (sequential Monte Carlo).

**Result Specifications**: What level of detail should be saved out during simulations?
Options are `Minimal`, `Temporal`, `SpatioTemporal`, and `Network`.

### Running an analysis

Analysis centers around the `assess` method with different arguments passed
depending on the desired analysis to run.
For example, to run a copper-plate convolution-based reliability assessment
(`Classic`) with aggregate LOLE and EUE reporting (`Minimal`),
one would run:

```julia
assess(Classic(), Minimal(), mysystemmodel)
```

If you instead want to run a simulation that considers energy-limited resources
and transmission constraints, using 100,000 Monte Carlo samples,
the method call becomes:

```julia
assess(Modern(samples=100_000), Minimal(), mysystemmodel)
```

To save results for each time period studied, change `Minimal` to `Temporal`:
```julia
assess(Modern(samples=100_000), Temporal(), mysystemmodel)
```

To save regional results for each simulation period, use the `SpatioTemporal`
result specification instead:
```julia
assess(Modern(samples=100_000), SpatioTemporal(), mysystemmodel)
```

### Querying Results

After running an analysis, metrics of interest can be obtained by calling the
appropriate metric's constructor with the result object.

For example, to obtain the system-wide LOLE over the simulation period:

```julia
result = assess(Modern(100_000), SpatioTemporal(), mysystemmodel)
lole = LOLE(result)
```

Single-period metrics such as LOLP can also be extracted if the appropriate
information was saved (i.e. if `Temporal` or `SpatioTemporal` result
specifications were used). For example, to get system-wide LOLP for April 27th,
2024 at 1pm EST:

```julia
lolp = LOLP(result, DateTime(2024, 4, 27, 13, tz"EST"))
```

Similarly, if per-region information was saved (i.e. if `Spatial` or
`SpatioTemporal` result specifications were used), region-specific metrics
can be extracted. For example, to obtain the EUE of Region A across the entire
simulation period:

```julia
eue_a = EUE(result, "Region A")
```

If the results specification supports it (i.e. `SpatioTemporal` or `Network`),
metrics can be obtained for both a specific region and time:

```julia
eue_a = EUE(result, "Region A", DateTime(2024, 4, 27, 13, tz"EST"))
```

Finally, if using the `Network` result spec, information about interface flows
and utilization factors can be obtained as well:

```julia
# Average flow from Region A to Region B during the hour of interest
flow_ab = ExpectedInterfaceFlow(
    result, "Region A", "Region B", DateTime(2024, 4, 27, 13, tz"EST"))

# Same magnitude as above, but different sign
flow_ba = ExpectedInterfaceFlow(
    result, "Region B", "Region A", DateTime(2024, 4, 27, 13, tz"EST"))

# Average utilization (average ratio of absolute value of actual flow vs maximum possible after outages)
utilization_ab = ExpectedInterfaceUtilization(
    result, "Region A", "Region B", DateTime(2024, 4, 27, 13, tz"EST"))
```

## Capacity Credit Calculations

Capacity credit calcuations build on probabilistic resource adequacy assessment
to provided capacity-based quantifications of the marginal benefit to
system resource adequacy associated with a specific resource or collection of
resources. Two capacity credit metrics (EFC and ELCC) are currently supported.

### Equivalent Firm Capacity (EFC)

`EFC` estimates the amount of idealized, 100%-available capacity that, when
added to a baseline system, reproduces the level of system adequacy associated
with the baseline system plus the study resource. The following parameters must
be specified:

 - The risk metric to be used for comparison (i.e. EUE or LOLE)
 - A known upper bound on the EFC value (usually the resource's nameplate
   capacity)
 - The regional distribution of the firm capacity to be added. This is
   typically chosen to match the regional distribution of the study resource's
   nameplate capacity.

For example, to assess the EUE-based EFC of a new resource with 1000 MW
nameplate capacity, added to the system in a single region named "A":

```julia
using ResourceAdequacy, CapacityCredit

# The base system, with power units in MW
base_system

# The base system augmented with some incremental resource of interest
augmented_system

# Get the lower and upper bounds on the EFC estimate for the resource
min_efc, max_efc = assess(
    EFC{EUE}(1000, "A"), Modern(nsamples=100_000), Minimal(),
    base_system, augmented_system)
```

If the study resource were instead split between regions "A" (600MW) and "B"
(400 MW), one could specify the firm capacity distribution as:

```julia
min_efc, max_efc = assess(
    EFC{EUE}(1000, ["A"=>0.6, "B"=>0.4]), Modern(nsamples=100_000), Minimal(),
    base_system, augmented_system)
```

### Equivalent Load Carrying Capability (ELCC)

`ELCC` estimates the amount of additional load that can be added to the system
(in every time period) in the presence of a new study resource, while
maintaining the baseline system's original adequacy level. The following
parameters must be specified:

 - The risk metric to be used for comparison (i.e. EUE or LOLE)
 - A known upper bound on the ELCC value (usually the resource's nameplate
   capacity)
 - The regional distribution of the load to be added. Note that this choice is
   somewhat ambiguous in multi-region systems, so assumptions should be clearly
   specified when reporting analysis results.

For example, to assess the EUE-based ELCC of a new resource with 1000 MW nameplate
capacity, serving new load in region "A":

```julia
using ResourceAdequacy, CapacityCredit

# The base system, with power units in MW
base_system

# The base system augmented with some incremental resource of interest
augmented_system

# Get the lower and upper bounds on the ELCC estimate for the resource
min_elcc, max_elcc = assess(
    ELCC{EUE}(1000, "A"), Modern(nsamples=100_000), Minimal(),
    base_system, augmented_system)
```

If instead the goal was to study the ability of the new resource to provide
load evenly to regions "A" and "B", one could use:

```julia
min_elcc, max_elcc = assess(
    ELCC{EUE}(1000, ["A"=>0.5, "B"=>0.5]), Modern(nsamples=100_000), Minimal(),
    base_system, augmented_system)
```

### Capacity credit calculations in the presence of sampling error

For non-deterministic assessment methods (i.e. Monte Carlo simulations),
running a resource adequacy assessment with different random number generation
seeds will result in different risk metric estimates for the same underlying
system. Capacity credit assessments can be sensitive to this uncertainty,
particularly when attempting to study the impact of a small resource on a
large system with a limited number of simulation samples.

PRAS takes steps to a) limit this uncertainty and b) warn against
potential deficiencies in statistical power resulting from this uncertainty.

First, the same random seed is used across all simulations in the capacity
credit assessment process. If the number of resources and their reliability
parameters (MTTF and MTTR) remain constant across the baseline and augmented
test systems, seed re-use ensures that unit-level outage profiles remain
identical across RA assessments, providing a fixed background against which to
measure changes in RA resulting from the addition of the study resource. Note
that satisfying this condition requires that the study resource be present in
the baseline case, but with its contributions eliminated (e.g. by setting its
capacity to zero). When implementing an assessment method that modifies the
user-provided system to add new resources (such as EFC), the programmer should
assume this invariance exists in the provided data, and not violate it in any
automated modifications.

Second, capacity credit assessments have two different stopping criteria. The
ideal case is that the upper and lower bounds on the capacity
credit metric converge to be sufficiently tight relative to a desired level
of precision. This target precision is 1 system power unit (e.g. MW) by
default, but can be relaxed to loosen the convergence bounds if desired via
the `capacity_gap` keyword argument. Once the lower and upper bounds are
tighter than this gap, their values are returned.

Additionally, at each bisection step, a hypothesis test is performed to ensure
that the theoretically-larger bounding risk metric is in fact larger than the
smaller-valued risk metric with a specified level of statistical significance.
By default, this criteria is a maximum p-value of 0.05, although this value
can be changed as desired via the `p_value` keyword argument. If at some point
the null hypothesis (the higher risk is not in fact larger than the lower
risk) cannot be rejected at the desired significance level, the assessment
will provide a warning indicating the size of the remaining capacity gap and
return the lower and upper bounds on the capacity credit estimate.
