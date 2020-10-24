# Optimizers

ADCME provides a rich class of optimizers and acceleration techniques for conducting inverse modeling. The programming model also allows for easily extending ADCME with customer optimizers. In this section, we show how to take advantage of the built-in optimization library by showing how to solve an inverse problem--estimating the diffusivity coefficient of a Poisson equation from sparse observations--using different kinds of optimization techniques.    


## Solving an Inverse Problem using L-BFGS optimizer

Consider the Poisson equation in 2D:

$$\begin{aligned}
\nabla  \cdot (\kappa (x,y)\nabla u(x,y)) &= f(x) & (x,y) \in \Omega\\ 
u(x,y) &=0 & (x,y)\in \partial \Omega
\end{aligned}\tag{1}$$

Here $\Omega$ is a L-shaped domain, which can be loaded using `meshread`. 

$$f(x,y) = -\sin\left(20\pi y+\frac{\pi}{8}\right)$$

and the diffusivity coefficient $\kappa(x,y)$ is given by 


$$\kappa(x,y) = 2+e^{10x} - (10y)^2$$

We can solve Equation 1 using a standard finite element method. Here, we use [AdFem.jl](https://github.com/kailaix/AdFem.jl) to solve the PDE.

```julia
using AdFem
using ADCME
using PyPlot 

function kappa(x, y)
    return 2 + exp(10x) - (10y)^2
end

function f(x, y)
    return sin(2π*10y+π/8)
end

mmesh = Mesh(joinpath(PDATA, "twoholes_large.stl"))

Kappa = eval_f_on_gauss_pts(kappa, mmesh)
F = eval_f_on_gauss_pts(f, mmesh)
L = compute_fem_laplace_matrix1(Kappa, mmesh)
RHS = compute_fem_source_term1(F, mmesh)

bd = bcnode(mmesh)
L, RHS = impose_Dirichlet_boundary_conditions(L, RHS, bd, zeros(length(bd)))

SOL = L\RHS 
close("all")
figure(figsize = (10, 4))
subplot(121)
visualize_scalar_on_gauss_points(Kappa, mmesh)
title("\$\\kappa\$")
subplot(122)
visualize_scalar_on_fem_points(SOL, mmesh)
title("Solution")
savefig("optimizers_poisson.png")
```

![](https://github.com/ADCMEMarket/ADCMEImages/blob/master/ADCME/Optimizers/optimizers_poisson.png?raw=true)


Now we approximate $\kappa(x,y)$ using a deep neural network ([`fc`](@ref) in ADCME). The script is nearly the same as the forward computation

```julia
using AdFem
using ADCME
using PyPlot 

function f(x, y)
    return sin(2π*10y+π/8)
end

mmesh = Mesh(joinpath(PDATA, "twoholes_large.stl"))

xy = gauss_nodes(mmesh)
Kappa = squeeze(fc(xy, [20, 20, 20, 1])) + 1.0
F = eval_f_on_gauss_pts(f, mmesh)
L = compute_fem_laplace_matrix1(Kappa, mmesh)
RHS = compute_fem_source_term1(F, mmesh)

bd = bcnode(mmesh)
L, RHS = impose_Dirichlet_boundary_conditions(L, RHS, bd, zeros(length(bd)))

sol = L\RHS 
```

We want to find a deep neural network such that `sol` and `SOL` match. We can train the neural network by minimizing a loss function. 

```julia
loss = sum((sol - SOL)^2)*1e10
```

Here we multiply the loss function by `1e10` because the scale of `SOL` is $10^{-5}$. We want the initial value of `loss` to have a scale of $O(1)$. 

We can minimize the loss function by 
```julia
sess = Session(); init(sess)
losses = BFGS!(sess, loss)
```

We can visualize the result:

```julia
figure(figsize = (10, 4))
subplot(121)
semilogy(losses)
xlabel("Iterations"); ylabel("Loss")
subplot(122)
visualize_scalar_on_gauss_points(run(sess, Kappa), mmesh)
savefig("optimizer_bfgs.png")
```

![](https://github.com/ADCMEMarket/ADCMEImages/blob/master/ADCME/Optimizers/optimizer_bfgs.png?raw=true)

We see that the estimated $\kappa(x,y)$ is quite similar to the reference one. 

## Use the optimizer from Optim.jl 

We have used the built-in optimizer L-BFGS. What if we want to try out other options? [`Optimize!`](@ref) is an API that allows you to try out custom optimizers. For convenience, it also supports optimizers from the [Optim.jl](https://github.com/JuliaNLSolvers/Optim.jl) package. 

Let's consider using BFGS to solve the above problem:

```julia
import Optim
sess = Session(); init(sess)
losses = Optimize!(sess, loss, optimizer = Optim.BFGS())

figure(figsize = (10, 4))
subplot(121)
semilogy(losses)
xlabel("Iterations"); ylabel("Loss")
subplot(122)
visualize_scalar_on_gauss_points(run(sess, Kappa), mmesh)
savefig("optimizer_bfgs2.png")
```

![](https://github.com/ADCMEMarket/ADCMEImages/blob/master/ADCME/Optimizers/optimizer_bfgs2.png?raw=true)


Unfortunately, it got stuck after several iterations. 

## Build your own optimizers

What if we want to design our own optimizers. To do this, we can construct an [`Optimizer`](@ref) object:

```julia
function ADCME.:optimize(opt::DefaultOptimizer)
    x = opt.x0
    g = zeros(length(x))
    losses = zeros(1000)
    @info opt.options
    for i = 1:opt.options[:niter]
        losses[i] = opt.f(x)
        opt.g!(g, x)
        x = x - 0.001 * g
    end
    return losses
end

opt = DefaultOptimizer()
sess = Session(); init(sess)
losses = Optimize!(sess, loss, optimizer = opt, niter = 1000)
```

We can visualize the result:
```julia
figure(figsize = (10, 4))
subplot(121)
semilogy(losses)
xlabel("Iterations"); ylabel("Loss")
subplot(122)
visualize_scalar_on_gauss_points(run(sess, Kappa), mmesh)
savefig("optimizer_gd.png")
```


![](https://github.com/ADCMEMarket/ADCMEImages/blob/master/ADCME/Optimizers/optimizer_gd.png?raw=true)

Not good enough, but at least we have a way to interact with optimizers!


## The `AbstractOptimizer` Interface

To create your own optimizer, you will be deal with `AbstractOptimizer` most of the time. The actual implementation is in (you need to `import ADCME:optimize` first)
```julia
optimize(opt::YourOwnOptimizer)
```
Here `YourOwnOptimizer` is a derived class of `AbstractOptimizer`, and ADCME provides `DefaultOptimizer` for convenience. You can assume that `opt` has the following fields:

- `f`: returns the loss function at `x`
- `g!`: an in-place function `g!(G, x)` that modifies the gradient vector `G` at `x`
- `x0`: the initial guess 
- `options`: a Symbol to Any mapping

Once `optimize` is implemented and `opt` is an instance of `YourOwnOptimizer`, you can run `optimize(opt)`. Or, if you want to use with the computational graph:

```julia
Optimize!(sess, loss; optimizer = YourOwnOptimizer(...))
```

Note ADCME will override `f`, `g!`, `x0`, and add additional options to `options`. See [`Optimizer`](@ref) for concrete examples. 