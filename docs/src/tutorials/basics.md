# The Reaction DSL
This tutorial covers some of the basic syntax for building chemical reaction
network models. Examples showing how to both construct and solve ODE, SDE, and
jump models are provided in [Basic Chemical Reaction Network Examples](@ref).

#### Basic syntax

The `@reaction_network` macro allows the (symbolic) specification of reaction
networks with a simple format. Its input is a set of chemical reactions, and
from them it generates a [`ReactionSystem`](@ref) reaction network object. The
`ReactionSystem` can be used as input to `ODEProblem`, `SteadyStateProblem`,
`SDEProblem`, `JumpProblem`, and more. `ReactionSystem`s can also be
incrementally extended as needed, allowing for programmatic construction of
networks and network composition.

The basic syntax is:

```julia
rn = @reaction_network begin
  2.0, X + Y --> XY               
  1.0, XY --> Z1 + Z2            
end
```

where each line corresponds to a chemical reaction. Each reaction consists of a
reaction rate (the expression on the left hand side of  `,`), a set of
substrates (the expression in-between `,` and `-->`), and a set of products (the
expression on the right hand side of `-->`). The substrates and the products may
contain one or more reactants, separated by `+`. The naming convention for
these are the same as for normal variables in Julia.

The chemical reaction model is generated by the `@reaction_network` macro and
stored in the `rn` variable (a normal Julia variable, which does not need to be
called `rn`). The generated `ReactionSystem` can be converted to a differential
equation model via
```julia
osys = convert(ODESystem, rn)
oprob = ODEProblem(osys, Pair.(species(rn),u0), tspan, Pair.(parameters(rn),p))
```
or more directly via
```julia
oprob = ODEProblem(rn, u0, tspan, p)
```
For more detailed examples, see the [Basic Chemical Reaction Network Examples](@ref).
The generated differential equations use the law of mass action. For the above
example, the ODEs are then

```math
\frac{d[X]}{dt} = -2 [X] [Y]\\
\frac{d[Y]}{dt} = -2 [X] [Y]\\
\frac{d[XY]}{dt} = 2 [X] [Y] - [XY]\\
\frac{d[Z1]}{dt}= [XY]\\
\frac{d[Z2]}{dt} = [XY]
```

#### Arrow variants
A variety of unicode arrows are accepted by the DSL in addition to `-->`. All of
these work:  `>`, `→` `↣`, `↦`, `⇾`, `⟶`, `⟼`, `⥟`, `⥟`, `⇀`, `⇁`. Backwards
arrows can also be used to write the reaction in the opposite direction. For example,
these three reactions are equivalent:
```julia
rn = @reaction_network begin
  1.0, X + Y --> XY               
  1.0, X + Y → XY      
  1.0, XY ← X + Y      
end
```
*On Julia 1.6 and up the plain text arrows `<--` (for backward reactions) and
`<-->` (for reversible reactions) also work. Note, these are not available on
earlier Julia versions.*

#### Using bi-directional arrows
Bi-directional unicode arrows can be used to designate a reaction that goes two
ways. These three models are equivalent:
```julia
rn = @reaction_network begin
  2.0, X + Y → XY             
  2.0, X + Y ← XY          
end
rn = @reaction_network begin
  2.0, X + Y ↔ XY               
end
```
If the reaction rates in the backward and forward directions are different, they
can be designated in the following way:
```julia
rn = @reaction_network begin
  (2.0,1.0) X + Y ↔ XY               
end
```
which is identical to
```julia
rn = @reaction_network begin
  2.0, X + Y → XY             
  1.0, X + Y ← XY          
end
```

#### Combining several reactions in one line
Several similar reactions can be combined in one line by providing a tuple of
reaction rates and/or substrates and/or products. If several tuples are provided,
they must all be of identical length. These pairs of reaction networks are all
identical:
```julia
rn1 = @reaction_network begin
  1.0, S → (P1,P2)               
end
rn2 = @reaction_network begin
  1.0, S → P1     
  1.0, S → P2
end
```
```julia
rn1 = @reaction_network begin
  (1.0,2.0), (S1,S2) → P             
end
rn2 = @reaction_network begin
  1.0, S1 → P     
  2.0, S2 → P
end
```
```julia
rn1 = @reaction_network begin
  (1.0,2.0,3.0), (S1,S2,S3) → (P1,P2,P3)        
end
rn2 = @reaction_network begin
  1.0, S1 → P1
  2.0, S2 → P2   
  3.0, S3 → P3  
end
```
This can also be combined with bi-directional arrows, in which case separate
tuples can be provided for the backward and forward reaction rates.
These reaction networks are identical
```julia
rn1 = @reaction_network begin
 (1.0,(1.0,2.0)), S ↔ (P1,P2)  
end
rn2 = @reaction_network begin
  1.0, S → P1
  1.0, S → P2
  1.0, P1 → S   
  2.0, P2 → S
end
```

#### Production and Destruction and Stoichiometry
Sometimes reactants are produced/destroyed from/to nothing. This can be
designated using either `0` or `∅`:
```julia
rn = @reaction_network begin
  2.0, 0 → X
  1.0, X → ∅
end
```
If several molecules of the same reactant are involved in a reaction, the
stoichiometry of a reactant in a reaction can be set using a number. Here, two
molecules of species `X` form the dimer `X2`:
```julia
rn = @reaction_network begin
  1.0, 2X → X2
end
```
this corresponds to the differential equation:

```math
\frac{d[X]}{dt} = -[X]^2\\
\frac{d[X2]}{dt} = \frac{1}{2!} [X]^2
```

Other numbers than 2 can be used, and parenthesis can be used to reuse the same
stoichiometry for several reactants:
```julia
rn = @reaction_network begin
  1.0, X + 2(Y + Z) → XY2Z2
end
```

#### Variable reaction rates
Reaction rates do not need to be constant, but can also depend on the current
concentration of the various reactants (when, for example, one reactant can activate the
production of another). For instance, this is a valid notation:
```julia
rn = @reaction_network begin
  X, Y → ∅
end
```
and will have `Y` degraded at rate

```math
\frac{d[Y]}{dt} = -[X][Y]
```

Note that this is actually equivalent to the reaction
```julia
rn = @reaction_network begin
  1.0, X + Y → X
end
```
*except* that the latter will be classified as [`ismassaction`](@ref) and the
former will not, which can impact optimizations used in generating
`JumpSystem`s. For this reason, it is recommended to use the latter
representation when possible.

Most expressions and functions are valid reaction rates, e.g.:
```julia
rn = @reaction_network begin
  2.0*X^2, 0 → X + Y
  gamma(Y)/5, X → ∅
  pi*X/Y, Y → ∅
end
```
but please note that user-defined functions cannot be called directly (see later
section [User defined functions in reaction rates](@ref)).

#### Defining parameters
Parameter values do not need to be set when the model is created. Components can
be designated as symbolic parameters by declaring them at the end:
```julia
rn = @reaction_network begin
  p, ∅ → X
  d, X → ∅
end p d
```
Parameters can only exist in the reaction rates (where they can be mixed with
reactants). All variables not declared after `end` will be treated as a chemical
species, and may lead to undefined behavior if unchanged by *all* reactions.

#### Naming the generated `ReactionSystem`
ModelingToolkit uses system names to allow for compositional and hierarchical
models. To specify a name for the generated `ReactionSystem` via the
`reaction_network` macro, just place the name before `begin`:
```julia
rn = @reaction_network production_degradation begin
  p, ∅ → X
  d, X → ∅
end p d
ModelingToolkit.nameof(rn) == :production_degradation
```

#### Pre-defined functions
Hill functions and a Michaelis-Menten function are pre-defined and can be used
as rate laws. Below, the pair of reactions within `rn1` are equivalent, as are
the pair of reactions within `rn2`:
```julia
rn1 = @reaction_network begin
  hill(X,v,K,n), ∅ → X
  v*X^n/(X^n+K^n), ∅ → X
end v K n
rn2 = @reaction_network begin
  mm(X,v,K), ∅ → X
  v*X/(X+K), ∅ → X
end v K
```
Repressor Hill (`hillr`) and Michaelis-Menten (`mmr`) functions are also
provided:
```julia
rn1 = @reaction_network begin
  hillr(X,v,K,n), ∅ → X
  v*K^n/(X^n+K^n), ∅ → X
end v K n
rn2 = @reaction_network begin
  mmr(X,v,K), ∅ → X
  v*K/(X+K), ∅ → X
end v K
```
