---
layout: post
title: "Animation Blending"
date: 2026-01-15
---
<style>
video {
  max-width: 100%;
  height: auto;
  display: block;
  margin: 1rem auto;
}
</style>

**1501 words**

# **74 words** Introduction
[ADD IMPRESSIVE FINAL VIDEO OF PRODUCT]

This character is simultaneously running 5 different state machines with bone masking, blending between locomotion while playing distinct upper and lower body animations - all powered by a custom animation system built from scratch. Building this taught me 3 critical lessons: start simple, separate your concerns and layer your complexity. The challenge was designing BlendMotion - a pure animation mathematics library - that could integrate into any C++ game engine without forcing architectural compromises. 

# **175 words** Why this is hard (and worth doing right)
Animation systems are deceptively complex. On the surface, they're just "play animation A, blend to Animation B" - But production systems need to handle simultaneous animations across different body parts, smooth transitions between states, dynamic blending based on movement speed, and all of this while maintaining 60fps. Get the math wrong and you see pops and jitters. Get the architecture wrong and adding new features becomes a nightmare of tangled dependencies.

I approached this by splitting the problem in two: **BlendMotion** handles pure animation mathematics - skeleton hierarchies, pose blending, bone transformations - with zero engine dependencies. **GameEngine** handles integration - loading assets, managing components, rendering debug visualizations. This separation meant I could test BlendMotion independently, swap engines if needed, and reason about each system in isolation.

The result? A system running 5 concurrent state machines with per-bone masking, 2D blend spaces for locomotion, layered upper/lower body animations, and smooth transitions between arbitrary states. More importantly: a codebase where adding features doesn't require rewiring the engine, and debugging math doesn't mean wading through rendering code.

# The four lessons I wish I had known before starting

## **256 words** Lesson 1: Start Simple - Build the foundation right
I didn't start with 5 state machines and bone masking. I started with: "Can I load a skeleton and play one animation?" That's it. No blending, no transitions, no layers - just parse the data. evaluate poses, at time T, and apply transforms to bones.

This sounds obvious, but it's tempting to architect for the complex system you *want* rather than the simple system you *need first*. I've seen projects (including my own earlier attempts) collapse under the weight of premature abstraction - designing for "What if we need 10 blend layers?" before proving you blend 2 poses correctly.

The BlendMotion API reflects this philosophy. Here's the entire foundation for simple playback:
```cpp
// Animator component - just tracks current animation and playback state
Animator animator;
animator.currentAnimation = walkAnimation;
animator.animationState.Play(walkAnimation->GetAnimationClip().get(), /*loop=*/true);

// Each frame: update time and evaluate pose
animator.animationState.Update(deltaTime);

AnimationEvaluationResult result;
AnimationEvaluator::Evaluate(*skeleton, animator.animationState, result);

// Apply the final bone matrices
animator.boneMatrices = result.boneMatrices;
```
This is the entire update loop for simple animation playback. No state machines, no blend trees - just time advancement and pose evaluation. The 'Animator' component handles this path. while entities needing complex behavior add ad 'AnimationController' component that takes over with state machines.

Because this core was rock-solid and well-tested, I could layer complexity without constantly backtracking to fix fundamental issues. State machines? They just manage multiple 'AnimationState' instances. Blend spaces? They call 'Evaluate()' multiple times and blend the results. The foundation never changed.

**The takeaway:** Nail single animation playback before touching blending. Nail two-pose blending before building blend spaces. Nail blend spaces before adding state machines. Each layer depends on the one below being bulletproof. Rush the foundation and you'll spend more time debugging basics than building features.

Start simple. Prove it works. Then add one layer of complexity at a time.
<video controls>
  <source src="{{ '/assets/videos/single_animation.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>
## **281 words** Lesson 2: Separation of Concerns isn't optional
The moment I decided to split BlendMotion from the game engine, the project transformed from "my animation system" into "a reusable animation library." This wasn't just good architecture - it was a forcing function that made every design decision better.

**BlendMotion** is pure animation mathematics. No OpenGL. No file handling. No engine components. Just skeletons, poses, blending algorithms, and state machines. It takes data in, transforms it mathematically, and outputs bone matrices. That's it.

**Game Engine** handles everything else: loading GLTF files, managing 'Animator' components, rendering debug visualizations, handling input for character controllers. It uses BlendMotion as a library, just like it uses GLM for math or EnTT for the ECS.

This separation paid immediate dividends:
- **Debugging was surgical.** Animation math bugs stayed in BlendMotion. Rendering issues stayed in the engine. Clean separation, clear boundaries.
- **Optimization was targeted.** When profiling revealed animation evaluation as a bottleneck I could focus entirely on BlendMotion's math without touching engine code. This separation meant I could reason about performance in isolation.
- **Design decisions were clearer.** Should root motion extraction live in BlendMotion or in the engine? The boundary forced me to think: "Is this animation math or engine integration?" That clarity prevented the architectural muddle that kills projects.

The cost? An interface boundary. BlendMotion returns 'std::vector<glm::mat4>', the engine consumes it. But that boundary is where clarity lives - it forces explicit contracts instead of tangled dependencies.

**The takeaway:** Framework-agnostic design isn't about theoretical portability. It's about forcing yourself to separate "what" (animation mathematics) from "how" (engine integration). When those concerns blur, everything becomes harder to understand, debug and extend.

Separation of concerns isn't optional. It's the foundation that makes everything else possible.

## **267 words** Lesson 3: Layering enabled complexity
Early on, I had a character running around with smooth locomotion blending. Great. Then I needed the character to aim a weapon while running. My first instinct? "I'll just add another blend on top!"

That's when I learned the hard way: **additive blending and layered blending are fundamentally different problems.**

Additive systems work when you want to add motion on top of a base - a breathing idle, a head turn, subtle adjustments. But when you need the upper body doing something completely different from the lower body? Additive falls apart. You don't want to "add aiming" to "walking" - you want to *replace* the upper body animation entirely while keeping the lower body intact.

Enter **bone masking.** Each animation layer gets a mask: "These bones only." The locomotion layer owns the legs and hips. The upper body layer owns the spine, arms and head. They evaluate independently and compose together based on execution order and blend modes.

```cpp
// Layer 0: Full-body locomotion (all bones enabled)
locomotionLayer.Evaluate() // full body walk cycle

// Layer 1: Upper body actions (masked to spine + arms + head)
upperBodyLayer.Evaluate() //aiming animation, masked to upper body

// Compose: Lower body from Layer 0, upper body from Layer 1
FinalPose = Blend(locomotionLayer, upperBodyLayer, masks)
```

This architecture scaled beautifully. Need a character to run, aim, and react to damage? Three layers - with three masks. Need weapon swapping mid-animation? Add a fourth layer that only affects the weapon hand. Each layer is independent - they don't know about each other, they just evaluate their state machines and output transforms.

**The takeaway:** Simple blending gets you far, but complex character animation needs layering with bone masks. Don't try to solve "walk while aiming" with clever blend weights - solve it with proper architectural separation. Each layer handles one concern, and the composition system brings them together.

Proper layering makes complex animation manageable instead of impossible.

# **230 words** Technical Deep-Dive: The Evaluation pipeline
Understanding layering conceptually is one thing - implementing it is another. Here's the evaluation flow that runs every frame.

#### The entry point
```cpp
controller.layerController.Update(deltaTime);
controller.layerController.Evaluate(finalPose, rootMotionDelta, wasExtracted);
```
Every frame, the animation system updates all state machines (handling transitions, checking conditions), then evaluates them to produce the final bone matrices that drive character rendering.

#### Step 1: Orchestrating Layer Composition
The `Evaluate()` function orchestrates the entire process:
```cpp
bool AnimationLayerController::Evaluate(...) {
    // Compose all layers into final transforms
    std::vector<Transform> finalTransforms;
    ComposeLayers(finalTransforms, _outRootMotionDelta, _outRootMotionWasExtracted);

    // Convert final transforms to bone matrices via hierarchy traversal
    for (size_t i = 0; i < boneCount; ++i) {
        if (skeleton->GetBone(i).parentIndex == -1) {  // Root bones only
            AnimationEvaluator::ComputeBoneTransform(*skeleton, finalTransforms, i,
                                                    glm::mat4(1.0f), _outBoneMatrices);
        }
    }
    return true;
}
```
The key insight: **composition happens in local transform space, hierarchy traversal happens once at the end.** This separation is critical for performance - `O(layers × bones + bones)` instead of `O(layers × bones²)`

#### Step 2: Composing Layers
`ComposeLayers()` is where the magic happens - each layer contributes its animation to the final result:
```cpp
bool AnimationLayerController::ComposeLayers(...) {
    // Start with bind pose
    _outTransforms.resize(boneCount);
    skeleton->GetBindPoseTransforms(_outTransforms);

    // Apply each layer in execution order
    for (const auto& layer : layers) {
        if (layer.currentWeight < 0.001f) continue;  // Skip inactive layers

        // Evaluate this layer's state machine
        layer.stateMachine.Evaluate(*skeleton, animationClips, cachedLayerResult);

        // Convert skinning matrices → local transforms for composition
        for (size_t i = 0; i < boneCount; ++i) {
            if (layer.useMask && !layer.boneMask.IsBoneEnabled(i)) continue;

            // Extract world transform from skinning matrix
            glm::mat4 worldMatrix = cachedLayerResult.boneMatrices[i] * 
                                   glm::inverse(bone.offsetMatrix);
            
            // Convert to local space (relative to parent)
            glm::mat4 localMatrix = glm::inverse(parentWorldMatrix) * worldMatrix;
            AnimationEvaluator::DecomposeMatrix(localMatrix, 
                                               cachedLayerTransforms[i]...);
        }

        // Apply this layer to accumulator with bone masking
        ApplyLayer(layer, cachedLayerTransforms, _outTransforms);
    }
    return true;
}
```
**Why the conversion?** State machines output skinning matrices for efficiency, but layer blending requires local transforms for proper composition.

#### Step 3: State Machine Evaluation
Each layer's state machine handles arbitrary complexity - single animations, blend spaces, or smooth transitions:

```cpp
bool StateMachine::Evaluate(...) {
    const AnimationStateData* currentState = FindState(currentStateId);

    if (!isTransitioning) {
        // Evaluate current state based on type
        switch (currentState->type) {
        case SingleAnimation: return EvaluateSingleAnimation(...);
        case BlendSpace1D: return EvaluateBlendSpace1D(...);
        case BlendSpace2D: return EvaluateBlendSpace2D(...);
        }
    }

    // Transitioning between states - blend them
    EvaluateStateToResult(currentState, currentResult);
    EvaluateStateToResult(targetState, targetResult);
    
    // Blend based on transition progress
    float blendWeight = CalculateBlendWeight();
    AnimationEvaluator::BlendTransforms(currentResult, targetResult, 
                                       blendWeight, blendedResult);
    
    return true;
}
```
The state machine abstraction means layers don't care about internal complexity - simple or complex, the interface is identical.

#### Step 4: Final Hierarchy Traversal
Once all layers are composed, we traverse the skeleton hierarchy to compute final skinning matrices:
```cpp
void AnimationEvaluator::ComputeBoneTransform(const Skeleton& _skeleton, 
                                              const std::vector<Transform>& _localTransforms,
                                              int _boneIndex, 
                                              const glm::mat4& _parentTransform,
                                              std::vector<glm::mat4>& _outBoneMatrices) {
    // Compute world transform: parent * local
    glm::mat4 localMatrix = _localTransforms[_boneIndex].ToMatrix();
    glm::mat4 worldTransform = _parentTransform * localMatrix;

    // Final skinning matrix: world * offsetMatrix
    _outBoneMatrices[_boneIndex] = worldTransform * bone.offsetMatrix;

    // Recurse to children
    for (int childIndex : children) {
        ComputeBoneTransform(_skeleton, _localTransforms, childIndex,
                           worldTransform, _outBoneMatrices);
    }
}
```
This recursive traversal accumulates parent transforms down the hierarchy. The offset matrix converts world space to skinning space for GPU rendering.

#### The Architecture in Action
This pipeline demonstrates the three lessons: foundational simplicity (`ComputeBoneTransform` never changed), clean separation (BlendMotion vs engine), and independent layering (simple composition loop). Adding new layers requires zero changes to existing code.

# **125 words** Results & What's next
The end result is a system running 5 concurrent state machines with per-bone masking, handling smooth locomotion blending, independent upper/lower body animations, and root motion extraction - all while managing more than 60fps. Performance profiling revealed that proper architecture matters more than micro-optimizations: clean separation and efficient evaluation paths gave far better gains than chasing cache-friendly data layouts.

The technical foundation supports multiple blend space evaluation methods including triangulation, RBF networks, and various 2D interpolation approaches.

Upcoming deep-dive posts will cover blend space mathematics, root motion extraction, state machine architecture, and performance optimization lessons.

The three lessons - start simple, separate concerns, layer complexity - apply far beyond animation systems. They're architectural principles for any complex system where "just add another feature" leads to collapse.

# **93 words** Conclusion
Building BlendMotion taught me that **architecture decisions compound**. Starting simple gave me a solid foundation. Separating concerns made that foundation testable and maintainable. Layering enabled complexity without chaos. Each lesson enabled the next - rush any step and the whole structure weakens.

For fellow students tackling complex systems: These principles apply whether you're building animation systems, physics engines, or rendering pipelines. The temptation to architect for the final system upfront is strong, but resist it. Build the simplest thing that works. Prove it's solid, then add one layer of complexity at a time.

# References