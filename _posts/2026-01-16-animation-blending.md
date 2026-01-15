---
layout: post
title: "Animation Blending"
date: 2026-01-15
---
**1716 words**

# Introduction (150-200 words) **74 words**
[ADD IMPRESSIVE FINAL VIDEO OF PRODUCT]

This character is simultaneously running 5 different state machines with bone masking, blending between locomotion while playing distinct upper and lower body animations - all powered by a custom animation system built from scratch. Building this taught me 3 critical lessons: start simple, separate your concerns and layer your complexity. The challenge was designing BlendMotion - a pure animation mathematics library - that could integrate into any C++ game engine without forcing architectural compromises. 

# Why this is hard (and worth doing right) (150-200 words) **190 words**
Animation systems are deceptively complex. On the surface, they're just "play animation A, blend to Animation B" - But production systems need to handle simultaneous animations across different body parts, smooth transitions between states, dynamic blending based on movement speed, and all of this while maintaining 60fps. Get the math wrong and you see pops and jitters. Get the architecture wrong and adding new features becomes a nightmare of tangled dependencies.

I approached this by splitting the problem in two: **BlendMotion** handles pure animation mathematics - skeleton hierarchies, pose blending, bone transformations - with zero engine dependencies. **GameEngine** handles integration - loading assets, managing components, rendering debug visualizations. This separation meant I could test BlendMotion independently, swap engines if needed, and reason about each system in isolation.

The result? A system running 5 concurrent state machines with per-bone masking, 2D blend spaces for locomotion, layered upper/lower body animations, and smooth transitions between arbitrary states. More importantly: a codebase where adding new animation features doesn't require rewiring the engine, and debugging animation math doesn't mean wading through rendering code.

Here's how I got there, and what I learned along the way.

# The four lessons I wish I had known before starting (600-900 words)

## Lesson 1: Start Simple - Build the foundation right (200-300 words) **258 words**
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

## Lesson 2: Separation of Concerns isn't optional (200-300 words) **293 words**
The moment I decided to split BlendMotion from the game engine, the project transformed from "my animation system" into "a reusable animation library." This wasn't just good architecture - it was a forcing function that made every design decision better.

**BlendMotion** is pure animation mathematics. No OpenGL. No file handling. No engine components. Just skeletons, poses, blending algorithms, and state machines. It takes data in, transforms it mathematically, and outputs bone matrices. That's it.

**Game Engine** handles everything else: loading GLTF files, managing 'Animator' components, rendering debug visualizations, handling input for character controllers. It uses BlendMotion as a library, just like it uses GLM for math or EnTT for the ECS.

This separation paid immediate dividends:
- **Debugging was surgical.** Animation math bug? Open BlendMotion, no engine code in sight. Rendering issue? Open the engine, no animation math to wade through. Each system lived in its own world.
- **Optimization was targeted.** When profiling revealed animation evaluation as a bottleneck I could focus entirely on BlendMotion's math without touching engine code. This separation meant I could reason about performance in isolation.
- **Design decisions were clearer.** Should root motion extraction live in BlendMotion or in the engine? The boundary forced me to think: "Is this animation math or engine integration?" That clarity prevented the architectural muddle that kills projects.

The cost? An interface boundary. BlendMotion returns 'std::vector<glm::mat4>', the engine consumes it. But that boundary is where clarity lives - it forces explicit contracts instead of tangled dependencies.

**The takeaway:** Framework-agnostic design isn't about theoretical portability. It's about forcing yourself to separate "what" (animation mathematics) from "how" (engine integration). When those concerns blur, everything becomes harder to understand, debug and extend.

Separation of concerns isn't optional. It's the foundation that makes everything else possible.

## Lesson 3: Layering enabled complexity (200-300 words) **267 words**
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

# Technical Deep-Dive: The Evaluation pipeline **349 words**
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
**Why the conversion?** State machines output skinning matrices (world space * offset matrix) for efficiency, but layer blending requires local transforms. Each layer converts its output to local space, applies its bone mask, then blends into the accumulator.

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
The state machine abstraction means layers don't care about internal complexity - a layer evaluating a 2D blend space with 20 animations looks identical to one playing a single walk cycle.

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
This recursive traversal accumulates parent transforms down the hierarchy, ensuring each bone's world-space transform accounts for all ancestors. The offset matrix (inverse bind pose) converts world space to skinning space for GPU vertex skinning.

#### The Architecture in Action
This pipeline demonstrates the three lessons in practice:
1. **Start Simple** (Lesson 1): The foundation - `ComputeBoneTransform` - never changed. Everything is built on top of that.
2. **Separation of Concerns** (Lesson 2): BlendMotion handles pure math; the engine just calls `Evaluate()`
3. **Layering Enabled Complexity** (Lesson 3): Each layer evaluates independently, composition is a just a simple loop.

Adding a new layer requires zero changes to existing code. The state machine inside each layer can be arbitrarily complex, but the composition logic stays simple: evaluate, convert, blend, repeat.

# Results & What's next (150-200 words) **192 words**
The end result is a system running 5 concurrent state machines with per-bone masking, handling smooth locomotion blending, independent upper/lower body animations, and root motion extraction - all while managing more than 60fps. Performance profiling revealed that proper architecture matters more than micro-optimizations: clean separation and efficient evaluation paths gave far better gains than chasing cache-friendly data layouts.

The technical foundation supports multiple blend space evaluation methods: Linear LERP, Smoothstep, Triangulation-based blending, RBF networks, Voronoi diagrams, and bilinear interpolation for 2D spaces. State machines handle complex scenarios like conditional layer activation, smooth fade transitions and animation events triggering gameplay logic (weapon draws, footstep sounds, damage windows).

But this overview only scratches the surface. In upcoming deep-dive posts, I'll cover:
- **Blend space mathematics**: comparing triangulation vs RBF, when each approach works best
- **Root motion extraction**: coordinate space transformations and entity hierarchies
- **State machine architecture**: from simple transitions to multi-layered systems
- **Performance optimization**: what actually mattered vs what didn't

The three lessons - start simple, separate concerns, layer complexity - apply far beyond animation systems. They're architectural principles for any complex system where "just add another feature" leads to collapse.

# Conclusion (100-150 words) **93 words**
Building BlendMotion taught me that **architecture decisions compound**. Starting simple gave me a solid foundation. Separating concerns made that foundation testable and maintainable. Layering enabled complexity without chaos. Each lesson enabled the next - rush any step and the whole structure weakens.

For fellow students tackling complex systems: These principles apply whether you're building animation systems, physics engines, or rendering pipelines. The temptation to architect for the final system upfront is strong, but resist it. Build the simplest thing that works. Prove it's solid, then add one layer of complexity at a time.

# References
1. 
2. 
3. 