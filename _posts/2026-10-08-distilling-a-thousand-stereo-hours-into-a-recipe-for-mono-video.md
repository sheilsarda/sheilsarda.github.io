---
title: "Distilling a thousand stereo hours into a recipe for mono video"
date: 2026-10-08
slug: distilling-a-thousand-stereo-hours-into-a-recipe-for-mono-video
---

Suppose a data lab has two datasets:

1. 1,000 hours from a calibrated, head-mounted stereo rig with an IMU
2. 100,000 hours of monocular RGB from cameras with known intrinsics

The smaller collection gives us measurements we cannot obtain from every mono clip. The larger collection gives us more people, environments and interactions. I would use the stereo set to train and evaluate the estimators that process the mono set, then reconstruct selected scenes to produce additional supervision.

The target is task-relevant geometry within measured error bounds. A monocular model can learn useful scale priors from stereo supervision, but some scenes remain ambiguous. We need to establish which outputs transfer to new cameras and environments before treating the larger collection as robot training data.

## Label quality changes the policy

One measure of data usefulness is the real-world success rate of a policy trained on it. [HumanEgo](https://arxiv.org/abs/2605.24934), from May 2026, provides a relevant experiment.

In essence, the paper proposes:

- Aria's hand tracker, a learned model, uses the glasses' two monochrome SLAM cameras to recover 21 landmarks per hand in meters. The RGB camera plays no part in hand tracking.
- The authors compute a virtual gripper pose from those landmarks: position from the thumb-index fingertip midpoint, orientation from knuckle geometry, and opening from fingertip distance normalized per user.
- SAM2 segments the hand and arm, LaMa inpaints them out, and the pipeline renders a virtual parallel-jaw gripper and tracked object keypoints. The authors apply corresponding visual preprocessing to the robot at deployment.
- The gripper pose contributes to the render, the policy's spatial tokens, and the action label. On Water Flowers, raw human RGB gets 7.5% success, inpainting plus rendering gets 20%, and raw RGB plus spatial tokens gets 85%.

The authors train a flow matching policy from scratch on 30 minutes of Aria video per task, with no robot data. Each training example pairs a processed frame and spatial tokens with future gripper poses from the retargeted hand trajectory. The policy averages 92.5% success across four tasks, against 51.2% for ACT trained on matched-time robot teleoperation.

On Serve Bread, they hold the demonstrations, architecture and training recipe fixed and change the hand tracker. Aria-MPS gives 95% success over 40 trials, WiLoR 45%, HaMeR 32.5%, and MediaPipe 0%. The authors report systematic depth offsets of 5 to 11 cm from the monocular trackers, alongside jitter and missed detections. This establishes the cost of changing the tracker in that pipeline. It leaves open how much stereo-supervised training could recover.

## Stereo and IMU provide metric references

A known stereo baseline supplies metric depth wherever the cameras establish reliable correspondences. Visual-inertial systems such as [ORB-SLAM3](https://arxiv.org/abs/2007.11898) fuse camera observations and inertial measurements to estimate trajectory and gravity. A separate hand tracker uses the available views to estimate hand pose. Dense reconstruction and hand tracking each need their own processing and quality checks; a SLAM trajectory alone supplies neither.

Known monocular intrinsics help with projection, but leave absolute scale ambiguous without another constraint. A depth model can supply a learned scale estimate. Gravity can come from scene cues, and visual mapping can correct drift through revisits and loop closure. The stereo set gives us a way to measure the residual errors of those estimates in our domain.

I would use it as a teacher and as a source of priors, reserving independent captures to evaluate both.

## Teacher: pair the rig's RGB with measured labels

Synchronize the rig's RGB frames with its stereo and inertial observations. Use calibrated sensor extrinsics to recover the RGB camera's trajectory in a metric world frame and project depth and hand labels into its images. Keep visibility and uncertainty with each label. Occluded surfaces, failed correspondences and unreliable hand estimates should remain unlabeled.

The student receives RGB and camera intrinsics. The teacher uses the additional sensors. Fine-tune the relevant estimators on those pairs, preserving intrinsics through crops and resizes. Supply the known calibration wherever the estimator supports it; predicting field of view introduces another error when we already know the camera.

[MINT](https://arxiv.org/abs/2609.04958) provides a precedent for learning from a reconstruction pipeline. Its EgoPipeline combines GeoCalib, a MoGe-2 depth prior, visual mapping and HaWoR hand reconstruction. The authors filter 1,729 hours of egocentric video to 1,021 hours and train a model to recover camera and hand motion. Their four-GPU comparison reports a 3.67x speedup over the full labeling pipeline. They also report inherited monocular scale drift and unresolved long-horizon drift. My proposal is to test how much measured stereo and inertial supervision reduces those errors.

Hold out people and environments before making training pairs. A held-out rig recording, evaluated using RGB alone, measures transfer within the rig's capture conditions. To test the fleet cameras, collect a smaller paired set with representative fleet devices alongside the calibrated rig. Differences in shutter, blur and field of view need that test; augmentation alone cannot establish transfer.

## Prior: constrain dimensions and relationships across frames

Some quantities should remain consistent across a sequence. A person's hand dimensions and a rigid object's dimensions are examples. Camera height and head pitch change as the wearer moves, so they need distributions or time-varying estimates. A population average can regularize an ambiguous estimate, but cannot establish one person's hand size or one object's dimensions.

We already use a measured scale anchor in our ABC-130k reconstruction path. For episodes with calibrated stereo, we compute depth from disparity and compare it with the monocular depth estimate over valid pixels. We use a robust ratio to correct the scene's scale and carry that correction through export. Without a stereo pair, we record the monocular scale as unverified. This operates per captured scene; transferring the correction to unseen mono footage would require training and evaluation.

A single scale factor also leaves local shape errors untouched. A reconstructed tabletop can have the right width while a cup floats above it. The stereo collection could support more specific priors: distributions of object dimensions, relative poses, and observed support relationships. We should condition those priors on object identity and task, and retain uncertainty where the capture does not resolve the geometry.

## Reconstruction can produce additional supervision

In our current scene reconstruction experiments, we prepare camera poses, original images and geometric evidence, then use an agent to author an editable scene. The image routes use MoGe3 or MapAnything. The video route uses VidMap, with sparse tracked landmarks and per-view depth as its default evidence. The agent models separate objects, renders the scene into the observed views, and corrects discrepancies in appearance, geometry and placement. It also checks support, attachment and unintended intersections.

We use estimated geometry throughout that process. The mono pipeline's metric scale comes from learned priors, and its video world-up comes from an estimated floor plane. A calibrated stereo-plus-IMU capture could supply measured depth and an independent gravity estimate, while visual-inertial poses could constrain the rendering cameras. Those references would help distinguish a modeling error from an error in the geometry used to guide modeling.

There is a second role for the reconstructed scene. After validating the visible geometry against the capture, we could render additional viewpoints and camera configurations with corresponding depth, normals, object identities and poses. We could vary supported object dimensions or layouts and retain the labels for each variation. A monocular estimator could learn from those synthetic pairs alongside the original measured pairs.

That would extend our reconstruction work into a training pipeline. We have not established that these rendered examples improve inference on the mono fleet. The labels would describe the reconstructed scene, including its mistakes. We would need to test the synthetic data's added value and avoid treating generated backsides or hidden surfaces as observations of the real scene.

Some existing diagnostics make this distinction concrete. One geometry check runs MoGe on both the source image and the rendered image and compares their predicted depth and normals. We use it to locate discrepancies, but correlated model errors can produce agreement. Stereo-derived depth would provide an independent reference on the regions it measures. We could compare that reference with the rendered geometry itself, with the cameras and scale fixed.

The scene model could then support simulation, provided we author and validate the required collision geometry and physical properties. Visual agreement and plausible support do not establish mass, friction or contact dynamics. Recovering hand motion through an interaction also remains a separate problem from reconstructing its static surroundings.

## Spend reconstruction effort on selected scenes

I would run the mono estimators over the larger collection, use held-out calibration to identify useful clips and unresolved cases, and reconstruct a selected subset. That subset should cover recurring errors and gaps in the training distribution, including scenes the current models struggle with.

Our recent video reconstruction change makes this choice relevant. We switched the default scene input to sparse VidMap landmarks and mapped depth after a comparison with the denser route. We kept the complete-cloud pipeline as an option. That choice does not establish that dense fusion is worse in general; it means we should test which evidence helps the author produce better geometry before paying to densify everything.

Processing costs matter even before authoring. [ViPE](https://arxiv.org/abs/2508.10934) reports 3 to 5 fps at 640x480 on an RTX 5090. At an assumed 4 fps, processing all frames of 100,000 hours of 30 fps video would take 750,000 GPU-hours. Processing one frame per second would give a 25,000 GPU-hour arithmetic estimate at the same throughput, but that assumes the pipeline remains effective on the wider frame spacing. It excludes storage, transfer, scene authoring and downstream training. Faster inference and frame selection both need quality checks, especially around brief contacts and occlusions.

## Test the cheap baseline first

A team could start with existing mono estimators and discover that they already satisfy its task. ViPE estimates camera motion and near-metric depth from raw video, and MINT demonstrates training on large quantities of pipeline-generated labels. Neither result establishes accuracy on our cameras and tasks. HumanEgo's tracker errors also say nothing about ViPE's numerical error; they come from different systems and experiments.

If the mono baseline passes, the calibrated set may be most useful for evaluation. If direct stereo collection costs less per accepted demonstration than processing and filtering the mono corpus, collect more stereo. The 100-to-one ratio here describes the assumed datasets, not a permanent advantage for monocular capture. Compare cost per useful training example and downstream policy performance.

## The experiment I would run

Start with a task whose success depends on geometry that both capture systems can observe. Reserve 100 of the 1,000 stereo hours, with separate sessions for calibrating filters and for final evaluation. Compare the same mono estimator with 0, 10, 100 and 900 hours of stereo-supervised training. Fix the base model, training budget and mono training collection. Choose nested training subsets that cover people, cameras and environments. Compare direct measured supervision with an additional arm that includes rendered examples from validated reconstructions.

Measure absolute scale and pose errors without fitting them away at evaluation. Report hand-object relative error, detection coverage, temporal stability and failures by camera and environment. Test reconstructed scenes for surface placement and support, then train the same robot policy on equal amounts of accepted data from each arm. Report the rejected fraction as well as policy success, so a filter cannot appear to improve quality by discarding the useful diversity.

My prediction is that, by the end of 2027, a controlled evaluation will show task-specific stereo and inertial supervision bringing an RGB-only hand tracker within 10 percentage points of the stereo tracker on real-robot success across several manipulation tasks. It should also outperform a matched RGB-only baseline without that additional supervision. Failure to improve over that baseline would weaken the case for collecting the stereo set for training, even if it remains useful for evaluation. I would run the direct-supervision comparison before building the larger scene-rendering loop.

## Sources

Papers are linked inline: [HumanEgo](https://arxiv.org/abs/2605.24934), [ORB-SLAM3](https://arxiv.org/abs/2007.11898), [MINT](https://arxiv.org/abs/2609.04958), and [ViPE](https://arxiv.org/abs/2508.10934). The reconstruction examples describe our current implementation; the stereo-supervised student and scene-generated training comparisons are proposed experiments.
