---
title: "Paper review: TUCO, SimVLA, OCBench and Splat2Act"
date: 2026-10-13
slug: paper-review-tuco-simvla-ocbench-splat2act
thumb: /assets/posts/paper-review-tuco-simvla-ocbench-splat2act/tuco-real-tasks.jpg
---

Four papers from the first week of October 2026 ask the same question from different sides: why does a policy trained on simulated demonstrations so often fail on a real robot, and what fixes it? In our own co-training runs, adding scripted simulation episodes to real teleoperation data has so far made the real-robot policy worse, not better. So I read these papers for the specific fix each one offers. Each one changes a different property of the simulated data. TUCO changes which demonstrations you keep. SimVLA changes which states the data covers. OCBench changes how the demonstrations move. Splat2Act changes what the cameras see. Taken together, they turn "sim data doesn't transfer" into four separate questions you can test.

![Real executions of stack-the-cube, insert-the-peg, and place-the-cupcake]({{ site.baseurl }}/assets/posts/paper-review-tuco-simvla-ocbench-splat2act/tuco-real-tasks.jpg)

*Stack the cube, insert the peg, place the cupcake. These are the three real tasks in the table below, and the executions TUCO is scoring the simulated pool against. Figure 6 of Zhu et al., "TUCO: Curating Simulation Demonstrations for Sim-to-Real Robot Policy Co-Training," 2026, [arXiv:2610.05407](https://arxiv.org/abs/2610.05407).*

## TUCO: Curating Simulation Demonstrations for Sim-to-Real Robot Policy Co-Training

[TUCO](https://arxiv.org/abs/2610.05407) (Zhu et al., Stanford and collaborators, October 4) starts from a result many teams have seen without naming it. Co-training on the whole simulated pool can be roughly as bad as not co-training at all. On a Franka FR3 with an image-conditioned Diffusion Policy, 10 real demonstrations plus all 1,200 simulated ones averaged 25.8% success over three tasks. Ten real demonstrations alone averaged 16.7%. A random half of the simulated pool also gave 25.8%.

The fix is to score each simulated demonstration by its effect on the real robot. TUCO pretrains a policy in simulation, runs it on hardware, and keeps 10 successful real executions per task. Influence functions then estimate how upweighting each simulated demonstration would change the likelihood of each of those real executions. Demonstrations that point toward success stay. Among those, a log-determinant selection prefers demonstrations that support different real executions, so the chosen set covers more than one way of succeeding.

| Training data | Peg | StackCube | CupCake | Average |
|---|---|---|---|---|
| 10 real demos only | 15.0 | 35.0 | 0.0 | 16.7 |
| 10 real + all 1,200 sim | 17.5 | 40.0 | 20.0 | 25.8 |
| 10 real + random 600 sim | 7.5 | 57.5 | 12.5 | 25.8 |
| CUPID (utility ranking) | 75.0 | 65.0 | 80.0 | 73.3 |
| TUCO | 82.5 | 75.0 | 100.0 | 85.8 |

Most of that jump belongs to influence-based curation in general. [CUPID](https://arxiv.org/abs/2506.19121) (Agia et al., 2025), which ranks demonstrations by the same utility score, already reaches 73.3%. TUCO's coverage term adds 12.5 points on top, and the paper's significance tests compare TUCO with the full pool and random selection, not with CUPID. The setup also favors transfer: 3D-printed copies of the simulated objects, calibrated cameras and matched placement regions.

I find the method most useful as a diagnostic. If a curated subset recovers the gain, the pool mixes good trajectories with harmful ones. If no subset helps, the pool lacks something the real task needs, and no amount of selection can add it.

## SimVLA: Zero-Shot Sim-to-Real VLA Learning for Mobile Manipulation

That missing something is often recovery. A scripted teacher takes its typical path every time. The policy trained on it drifts off that path at test time, and the data says nothing about what to do there. Ross, Gordon and Bagnell made the cost precise in [the DAgger paper](https://arxiv.org/abs/1011.0686) (AISTATS 2011): errors compound with the square of the horizon unless the training data includes the learner's own states.

[SimVLA](https://arxiv.org/abs/2610.11248) (Baik and Lee, Yonsei and Seoul National University, October 8) builds that fix into a simulation-only pipeline at scale. It trains a mobile-manipulation VLA entirely in Isaac Lab. The data has three parts:

- **SimAction:** about 200,000 scripted trajectories across 35 tasks.
- **SimVQA:** about a million question-answer pairs written from privileged simulator state.
- **SimDeploy:** 22,000 rollouts of the policy itself in other simulated scenes.

SimDeploy exists because scripted trajectories never visit "shifted viewpoints, imperfect base poses, missed grasps." On LG's Anubis robot, across 2,120 real rollouts in a mock kitchen, a pantry and a real home, SimVLA averaged 54.4% task progress with no real training data. That beat π0.5 fine-tuned on 50 real demonstrations per task.

The authors call SimDeploy's gain task-dependent, but their own kitchen table is more decisive. Removing it lowers progress on all five tasks, from 0.50 to 0.60 down to 0.33 to 0.45, and cuts successes from 6 to 9 out of 20 down to 2 to 4. SimDeploy adds the policy's own states without teacher labels, so it is a cheaper cousin of DAgger. A scripted teacher that reads simulator state can label those states, and that is the obvious next step.

The counter-position deserves a fair hearing, and the paper supplies it. π0.5 fine-tuned on 200 real demonstrations reaches 67.8% in the kitchen it was trained in, and beats SimVLA there on every task. Real data in the deployment room still wins in that room. Simulation won the other two rooms: the same π0.5 fell to 21.3% in the pantry and 29% in the home, while SimVLA held roughly level.

## Behavioral Cloning Mystery (OCBench)

[OCBench](https://arxiv.org/abs/2610.07056) (Park and Levine, UC Berkeley, October 5) is a benchmark paper, but its data recipe matters for anyone writing scripted teachers. The authors argue that human demonstrations have three properties scripted data usually lacks:

- **Narrowly distributed:** each demonstration stays close to one way of doing the task.
- **Temporally correlated:** each action depends on several before it, not only on the current state.
- **Semantically diverse:** grasp, approach and order vary from one demonstration to the next.

Earlier scripted datasets instead get their diversity from Gaussian noise and push actions to their limits.

OCBench's scripts sample random waypoints for each phase and join them with smooth splines. Grasp angle, contact point, speed, gripper orientation, path curvature and strategy are all randomized. The scripts also make occasional mistakes and recover from them, with the mistake rate exposed as a knob. Trained on this data, behavior cloning shows the puzzles previously seen only with human data:

- A 50,000-episode dataset trained a worse policy than a 10,000-episode one.
- Closed-loop policies failed where 25-step open-loop action chunks worked.
- Validation loss rose while success improved.

The paper does not claim human-like data trains better policies, and it has no real-robot results. Its value here is the recipe. The usual complaint about planner-generated demonstrations is that they look robotic, with every first grasp following nearly the same path. Spline waypoints with randomized grasps, speeds and curvature address that directly, and the mistake-and-recover knob produces recovery data from the teacher instead of from the student's failures. Two findings are also cheap to check in any behavior-cloning stack: try chunk lengths around 25 steps, and don't pick checkpoints by validation loss.

## Splat2Act: From Captured Scenes to Personalized VLA Manipulation

[Splat2Act](https://arxiv.org/abs/2610.13945) (Ye et al., Georgia Tech, Waymo and TRI, October 9) attacks the visual gap. A 20 to 30 minute iPhone scan becomes a 3D Gaussian splat for rendering and a mesh for collision. One photo per object becomes a mesh through SAM3D. The pipeline then finds receptacles in the mesh automatically, keeps those a Stretch 3 can reach, and has a scripted oracle with privileged poses generate pickup demonstrations in Isaac Sim. Across 101 scanned campus buildings it produced 68,021 demonstrations on 852 receptacles. π0.5 is fine-tuned in two stages: first on public objects, then on the user's own objects in the deployment scene.

The headline is transfer. The best policy reached 66.7% in simulation and 63.9% on the real robot over the same 12 tasks. A second policy reached 50.0% in simulation and 58.3% on the real robot. Both stayed within 8 points of simulation and kept their order. Scene matters as much as appearance. Personalizing to the wrong scene cost 14 to 48 points, and in one case left the policy worse than no personalization.

The paper does not isolate the variable I most want isolated. No experiment renders the same scene from a plain mesh instead of the splat, so the close sim-to-real match shows this pipeline transfers, not that the splat is why. The samples are small: 12 simulated and 36 real trials per policy. The task is a single pickup judged by height, and the authors note that the real gripper's rubber fingertips forgive grasps the simulated contact model would not. In an unseen lab, both personalized policies fell to between 22% and 33%.

## Takeaways

Together, the four papers give a diagnostic order:

1. **Selection.** Score the pool by influence on real rollouts or real demonstrations. If a curated subset helps, the pool is mixed, and curation is cheap.
2. **Coverage.** If nothing in the pool helps, check whether the data ever leaves the teacher's path. Count failed grasps, empty grippers and recoveries in the dataset. If there are none, the fix is new data, from the policy's own rollouts with a teacher labeling them.
3. **Behavior.** Compare the action statistics of the scripted data with teleoperation data. Near-identical first grasps and saturated actions call for spline waypoints and a mistake knob.
4. **Appearance.** Hold the episodes fixed and change only the renderer. This is the experiment Splat2Act left out, and it is cheap for anyone with a scanned workcell.

These aren't four competing theories. Each fixes a different failure, and a real pipeline can suffer from several at once. SimVLA's real results come from combining two: broad scripted coverage plus the policy's own states. TUCO's gains are large partly because its setup had already closed the appearance gap with matched objects and calibrated cameras.

Nobody has yet published a sim-only policy that beats real-data fine-tuning in the room where the real data was collected. SimVLA came closest and still lost every kitchen task to π0.5 with 200 real demonstrations. My prediction is that the first paper to do it, within the next year, will combine at least three of these four: a scanned or matched scene, a teacher that labels the policy's own states, and demonstrations with human-like variation, curated against real outcomes. If someone gets there with appearance alone, with no on-policy data, I will have been wrong about how much the coverage gap matters.

## References

Agia, Christopher, et al. "CUPID: Curating Data Your Robot Loves with Influence Functions." *arXiv*, 23 June 2025, arxiv.org/abs/2506.19121.

Baik, Kyoungin, and Youngwoon Lee. "SimVLA: Zero-Shot Sim-to-Real VLA Learning for Mobile Manipulation." *arXiv*, 8 Oct. 2026, arxiv.org/abs/2610.11248.

Park, Seohong, and Sergey Levine. "Behavioral Cloning Mystery." *arXiv*, 5 Oct. 2026, arxiv.org/abs/2610.07056.

Ross, Stéphane, et al. "A Reduction of Imitation Learning and Structured Prediction to No-Regret Online Learning." *Proceedings of the Fourteenth International Conference on Artificial Intelligence and Statistics (AISTATS)*, 2011, arxiv.org/abs/1011.0686.

Ye, Xiaomeng, et al. "Splat2Act: From Captured Scenes to Personalized VLA Manipulation." *arXiv*, 9 Oct. 2026, arxiv.org/abs/2610.13945.

Zhu, Ning, et al. "TUCO: Curating Simulation Demonstrations for Sim-to-Real Robot Policy Co-Training." *arXiv*, 4 Oct. 2026, arxiv.org/abs/2610.05407.
