# glossary

## tenant
an economic and technical boundary for data, permissions, and billing

## niche
a domain-specific assembly line built on the platform

## product
a packaged output or service delivered to a paying customer

## agent
a reusable thinking role such as collector, evaluator, synthesizer, compliance, or delivery

## cognitive protocol runtime
the runtime model for DAI. deterministic platform code moves a shared decision artifact through four macro cognitive protocols (Perceive, Interrogate, Discern, Decide) and a final Synthesize layer. each protocol has a fixed set of three micro-actions, typed inputs and outputs, allowed tools, forbidden behaviors, and quality rules.

## macro protocol
one of the four cognitive protocols in the runtime: Perceive, Interrogate, Discern, or Decide. a macro protocol owns one cognitive responsibility, exposes three micro-actions, and is moved through by deterministic platform code.

## micro-action
one of the twelve canonical cognitive actions inside a macro protocol. Perceive owns Detect, Frame, Aim. Interrogate owns Question, Probe, Verify. Discern owns Weigh, Contrast, Stress. Decide owns Resolve, Position, Justify. micro-actions are cognitive, not platform plumbing. retrieval is not a micro-action.

## decision artifact
the shared working object that travels through the runtime. each protocol reads from it and writes back to it. the artifact carries inputs, per-protocol outputs, and the final synthesized result. one run, one artifact.

## synthesize layer
the final layer of the runtime. integrates what survived the prior protocols into the consumable artifact. uses Integrate, Compose, and Deliver as internal operations. not counted as a fifth macro protocol and does not contribute to the twelve cognitive micro-actions.

## probe
the Interrogate micro-action that performs targeted investigation of signal gaps, evidence needs, source follow-ups, or tool-backed questions. probe may request platform retrieval through allowed tools, but it is not retrieval itself. probe replaces the earlier proposed Interrogate.Retrieve to avoid collision with the platform retrieve stage.

## position
the Decide micro-action that sets the decision posture from the niche posture vocabulary. position names a posture (in sports today: play, pass, monitor, wait, compare, avoid), not a pick or a lock. the UI label remains Read Stance. position replaces the earlier proposed Decide.Choose.
