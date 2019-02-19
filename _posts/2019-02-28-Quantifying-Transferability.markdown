---
layout: post
title: "Quantifying the Transferability of Control Policies Learned Using Domain Randomization"
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

2. Sample-based optimization (e.g., reinforcement learning) is known to be **optimistically biased**. This means, that the optimizer will over-fit to the provided samples, i.e., optimize for the simulation and not for the real problem, which the one we actually want to solve.

The first approach to make control policies transferable from simulation to the reality, also called bridging the _reality gap_, was presented by [Jakobi et al.](http://users.sussex.ac.uk/~inmanh/jakobi95noise.pdf).
The authors showed that by adding noise to the sensors and actors while training in simulation, it is possible to yield a transferable controller. This approach has two limitations: first, the researcher has to carefully select the correct magnitude of noise for every sensor and actor, second the underlying dynamics of the system remain unchanged.

## Domain Randomization

One possibility to tackle the challenges mentioned in the previous section is by randomizing the simulations. The most prominent recent success using domain randomization is the robotic in-hand manipulation of physical objects, described in a [blog post from OpenAI](https://blog.openai.com/learning-dexterity/).

A _domain_ is one instance of the simulator, i.e., a set of _domain parameters_ describes the current world our robot is living in. Basically, domain parameters are the quantities that we use to parametrize the simulation. This can be physics parameters like the mass and extents of an object, as well as the simulation's time step size, or visual features like textures camera positions.

Loosely speaking, randomizing the physics parameters can be interpreted as another way of injecting noise into the simulation while learning. In contrast to simply adding noise to the sensors and actors, this approach allows to selectively express the uncertainty on one phenomenon (e.g., rolling friction).  
**The motivation of domain randomization in the context of learning from simulations** is the idea that if the learner has seen many variations of the domain, then the resulting policy will be more robust towards modeling uncertainties and errors. Furthermore, if the learned policy is able to maintain its performance across an ensemble of domains, it is more like to transferable to the real world.

### What to Randomize?

A lot of research in the sim-2-real field has been focused on randomizing visual features (e.g., textures, camera properties, or lighting). Examples are the work of [Tobin et al.](https://arxiv.org/pdf/1703.06907.pdf), who trained an object detector for robot grasping (see figure), or the research done by [Sadeghi and Levine](https://arxiv.org/pdf/1611.04201.pdf), where a drone learned to fly from experience gathered in visually randomized environments.
<img align="right" src="/assets/img/2019-02-28/Tobin_etal_2018_Fig1.jpg" width="30%">

In this blog post, we focus on the randomization of physics parameters (e.g., masses, centers of mass, friction coefficients, or actuator delays), which change the dynamics of the system at hand.
Depending on the simulation environment, **the influence of some parameters can be crucial, while other can be neglected**.
> To illustrate this point, we consider a ball rolling downhill on a inclined plane. In this scenario, the ball's mass as well as radius do not influence how fast the ball is rolling. So, varying this parameters while learning would be a waste of computation time.  
Note: the ball's inertia tensor (e.g., solid or hollow sphere) does have an influence.

### How to Randomize?

After deciding on which domain parameters we want to randomize, we must decide how to do this. Possible approaches are:

1. Sampling domain parameters from static probability distributions.  
   This approach is the most widely used of the listed. The common element of ... is that every domain parameter is randomized according to a specified distribution, e.g. a uniform distribution around the nominal value. It is also possible to randomize
   Examples of this randomization strategy are the work by [OpenAI], , and [Muratore et al.](https://www.ias.informatik.tu-darmstadt.de/uploads/Team/FabioMuratore/Muratore_Treede_Gienger_Peters--SPOTA_CoRL2018.pdf)

2. Sampling domain parameters from adaptive probability distributions.  
   [Chebotar et al.](https://arxiv.org/pdf/1810.05687.pdf) presented a very promising method on how to close the sim-2-real loop by adapting the distributions from which the domain parameters are sampled depending on results from real-world rollouts.
   The main advantage is, that this approach alleviates the need for hand-tuning the distributions of the domain parameters, which is currently a significant part of the hyper-parameter search. On the other side, the adaptation requires 
   For this reason, we will only focus on methods that sample from static probability distributions.

3. Applying adversarial perturbations.  
   One could argue that technically these approaches do not fit the domain randomization category, since the perturbations are not necessarily random. However, I think this concept is an interesting compliment to the previously mentioned sampling methods. In particular, I want to highlight the following two ideas.
   [Mandlekar et al.](http://vision.stanford.edu/pdf/mandlekar2017iros.pdf) proposed physically plausible perturbations of the domain parameters by randomly deciding (Bernoulli experiment) when to add a rescaled gradient of the expected return w.r.t. the domain parameters. Therefore, the proposed algorithm requires a differentiable physics simulator.
   [Pinto et al.](https://arxiv.org/pdf/1703.02702.pdf) suggested to add a antagonist agent whose goal is to hinder the protagonist agent (the policy to be trained) from fulfilling its task. Both agents are trained simultaneously and make up a zero-sum game.  
   In general, adversarial approaches may provide a particularly robust policy.  However, without any further restrictions,it is always possible create scenarios in which the protagonist agent can never win, i.e., the policy will not learn the task.

> Interestingly, all publications I have read so far randomize the _domain parameters_ in a per-episode fashion, i.e., once at the beginning of every rollout (excluding the adversarial approaches mentioned in the list above). Alternatively, one could randomize the parameters every time step.
I see two reasons, why the community so far only randomizes once per rollout. First, it is harder to implement from the physics engine point of view. Second, the very frequent parameter changes are most likely detrimental to learning, because the resulting dynamics would become significantly nosier.

## Quantifying the Transferability

Frame reinforcement learning problem as a _stochastic program_ (SP).
$$
    \xi \sim p(\xi; \psi)\\
    J(\theta^{\star}) = \max_{\theta \in \Theta} \mathbb{E}_\xi \left[J(\theta, \xi) \right]
$$

approximated
$$
    \hat{J}_n(\theta^{\star}_n) = \max_{\theta \in \Theta} \frac{1}{n}\sum_{i=1}^{n} J(\theta, \xi_i)
$$

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

_Optimality Gap_ (OG) at the candidate solution $\theta^c$
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


### SPOTA &mdash; Sim-2-Sim Results 
Preliminary results on transferring policies trained with SPOTA from one simulation to another have been reported in [Muratore et al.](https://www.ias.informatik.tu-darmstadt.de/uploads/Team/FabioMuratore/Muratore_Treede_Gienger_Peters--SPOTA_CoRL2018.pdf).

<center>
<iframe width="50%" src="https://www.youtube.com/watch?v=RQ7zq_bcv_k" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
<iframe width="50%" src="https://www.youtube.com/watch?v=ORi9sjhs_tw" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

### SPOTA &mdash; Sim-2-Real Results 

... TDA ...

---

## Authors

[Fabio Muratore](https://www.ias.informatik.tu-darmstadt.de/Team/FabioMuratore) &mdash; 	Intelligent Autonomous Systems, TU Darmstadt, Germany

## Acknowledgements 

I want to thank Ankur Handa for proofreading and editing this post. 

## Credits

Figure Josh Tobin