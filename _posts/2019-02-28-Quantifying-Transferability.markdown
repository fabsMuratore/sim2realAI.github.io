---
layout: post
title: "Quantifying the Transferability of Control Policies from Simulation to Reality"
date: 2019-02-28 12:00:00 +0100
description: On the simulation optimization bias and the optimality gap in the context of reinforcement learning # Add post description (optional)
img:  # Add image post (optional)
---

This post is about how we can quantitatively estimate the transferability of a control policies learned in a randomized simulation.

Learning continuous control policies in the real world is expensive in terms of time (e.g., gathering the data) and resources (e.g., wear and tear on the robot).
Therefore, simulation-based policy search appears to be an appealing alternative.

## Learning from Physics Simulations

In general, learning from physics simulations introduces two major challenges:
1. Physics engines (e.g., [Bullet](https://pybullet.org/wordpress/), [Vortex](https://www.cm-labs.com/vortex-studio/), or [MoJoCo](http://www.mujoco.org/)) are build on models, which are always an approximation of the real world and thus **inherently inaccurate**. For the same reason, there will always be **unmodeled effects**.

2. Sample-based optimization (e.g., reinforcement learning) is known to be **optimistically biased**. This means, that the optimizer will over-fit to the provided samples, i.e., optimize for the simulation and not for the real problem, which we actually want to solve.

The first approach to make control policies transferable from simulation to the reality (also called bridging the _reality gap_) was presented by [Jakobi et al.](http://users.sussex.ac.uk/~inmanh/jakobi95noise.pdf).
The authors showed that by adding noise to the sensors and actors while training in simulation, it is possible to yield a transferable controller. This approach has two limitations: one has to carefully select the correct magnitude of noise for every sensor and actor, and the underlying dynamics of the system remain unchanged.

## Domain Randomization

One possibility to alleviate the challenges mentioned in the previous section is by randomizing the simulations. The most prominent recent success using domain randomization is the robotic in-hand manipulation of physical objects, described in a [blog post from OpenAI](https://blog.openai.com/learning-dexterity/).

A _domain_ is one instance of the simulator, i.e., a set of domain parameters describes the current world our robot is living in. Basically, _domain parameters_ are the quantities that we use to parametrize the simulation. This can be classical physics parameters like the mass and extents of an object, as well as the simulation's time step size, or visual features like textures camera positions.

Loosely speaking, randomizing the physics parameters can be interpreted as another way of injecting noise into the simulation. In contrast to simply adding noise to the sensors and actors, this approach allows to selectively express the uncertainty on one phenomenon (e.g., rolling friction).  
The **motivation of domain randomization in the context of learning from simulations** is the idea that if the learner has seen many variations of the domain, the resulting policy will be more robust towards modeling uncertainties. Furthermore, if the learned policy is able to maintain its performance across an ensemble of domains, it is more like to transferable to the real world.

### What to Randomize?

A lot of research in the sim-2-real area has been focused on randomizing visual features (e.g., textures, camera properties, or lighting). Examples are the work of [Tobin et al.](https://arxiv.org/pdf/1703.06907.pdf), who trained an object detector for robot grasping, or the research done by [Sadeghi and Levine](https://arxiv.org/pdf/1611.04201.pdf), where a drone learned to fly from experience gathered in visually randomized environments.

In this blog post, we focus on the randomization of physics parameters (e.g., masses, center of mass, friction coefficients, or actuator delays), which change the dynamics of the system at hand.
Depending on the simulation environment, **the influence of some parameters can be crucial, while other can be neglected**.
> To illustrate this point, we consider a ball rolling downhill on a inclined plane. In this scenario, the ball's mass as well as radius do not influence how fast the ball is rolling. So, varying this parameters while learning would be a waste of time.  
Note: the ball's inertia tensor (e.g., solid or hollow sphere) does have an influence.

### How to Randomize?

After deciding on which domain parameters we want to randomize, we must decide how we want to do this. Possible approaches are:

1. Sampling domain parameters from static probability distributions.  
   This

2. Sampling domain parameters from adaptive probability distributions.  
   [Chebotar et al.](https://arxiv.org/pdf/1810.05687.pdf) presented a very promising method on how to close the sim-2-real loop by adapting the distributions from which the domain parameters are sampled  depending on results from real-world rollouts. The unique part of the proposed _SimOpt framework_ 

3. adversarial perturbations.  
   Technically, this is not domain randomization.

> Interestingly, all publications I have read so far randomize the domain parameters in a per-episode fashion, i.e., once at the beginning of every rollout. Alternatively, one could randomize the parameters every time step.
I see two reasons, why the community so far only randomizes once per rollout. First, it is harder to implement, since this could require to add and remove bodies from the physics simulation every time step. Second, the very frequent parameter changes are most likely detrimental to learning, because the resulting dynamics would become significantly nosier.

## Quantifying the Transferability

Frame reinforcement learning problem as a _stochastic program_

_Simulation Optimization Bias_ (SOB)
$$
    b[\hat{J}_n] =
    \underbrace{
        \mathbb{E}_\xi \left[ \max_{\hat{\theta} \in \Theta} \frac{1}{n}\sum_{i=1}^{n} J(\hat{\theta}, \xi_i) \right]
    }_{\text{optimal value for samples}}
    -
    \underbrace{
        \max_{\theta \in \Theta} \mathbb{E}_\xi \left[ \frac{1}{n}\sum_{i=1}^{n} J(\theta, \xi_i) \right]
    }_{\text{true optimal value}}
    \ge 0
$$

_Optimality Gap_ (OG)
$$
    G(\theta^c) =
    \underbrace{\max_{\theta \in \Theta} \mathbb{E}_\xi \left[J(\theta, \xi) \right]}_{\text{best solution's value}} -
    \underbrace{\mathbb{E}_\xi \left[J(\theta^c, \xi) \right]}_{\text{candidate solution's value}}
    \ge 0
$$

Estimated OG
$$
    G_n(\theta^c) = \max_{\theta\in\Theta} \hat{J}_n(\theta) - \hat{J}_n(\theta^c) \ge G(\theta^c)
$$

Simulation-based PolicyOptimization with Transferability Assessment (SPOTA) [Muratore et al.](https://www.ias.informatik.tu-darmstadt.de/uploads/Team/FabioMuratore/Muratore_Treede_Gienger_Peters--SPOTA_CoRL2018.pdf)


... based on the method to determine the solution quality in stochastic programs suggested by [Mak et al.](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.196.4273&rep=rep1&type=pdf).

### SPOTA &mdash; Sim-2-Sim Results


### SPOTA &mdash; Sim-2-Real Results

---

## Authors

[Fabio Muratore](https://www.ias.informatik.tu-darmstadt.de/Team/FabioMuratore)

## Acknowledgements 

We would like to thank Ankur Handa for proofreading and editing this post. 

## Credits
