# agree-specification-example

This repository is intended to be a complete example of using the BriefCASE OSATE plugin to cyber-harden an AADL system model.
BriefCASE is a collection of tools integrated into OSATE for *model-based engineering* of cyber-resiliency.
The tools highlighted here are: AGREE and SPLAT.
AGREE is a tool for hierarchical contract reasoning.
SPLAT is a tool to synthesize CakeML implementations from AGREE *code contracts*.

Code contracts in AGREE use variables to evaluate input and compute output for every output on the component in each step of evaluation.
Such contracts represent an implementation for the component.
SPLAT proves such implementations correct against additional guarantees that represents properties of the component behavior, and it then synthesizes the CakeML from the code contract.
The synthesized CakeML includes a proof certificate that the resulting CakeML exactly preserves the meaning of the AGREE code-contract. 

# Obtaining the Tools

BriefCASE is a fully integrated OSATE plug-in and can be installed on most systems what include a Java Runtime at version 11 or greater. The install script, depending on download speeds, takes minutes to install with most the time spent downloading Osate:

  * [Windows Install Script](https://github.com/loonwerks/CASE/blob/master/TA5/make_fmide.bat)
  * [Linux Install Script](https://github.com/loonwerks/CASE/blob/master/TA5/make_fmide.sh)
  * [OSX x86-64 Install Script](make_fmide-osx-x86.sh)

All the scripts require a network connection, `curl`, and a Java Runtime of 11 or greater. Please note that while the AGREE analysis works in all platforms, SPLAT only runs on Linux in the current distribution.

# Running AGREE and SPAT

There is an existing an existing OSATE project defined for the example (see `.project`). 
The project can be imported into the OSATE workspace as an existing project by selecting the directory in the dialogue box. 
Once the project is opened, AGREE and SPAT can be used.

AGREE is invoked from BriefCASE by first selecting an implementation, and then with either the context menu on a right-click or the actual menu.
The context menu is `AGREE`, and then `Verify Single Layer`. The actual menu is `Analysis`, `AGREE`, and then `Verify Single Layer`.
Here are the components for which AGREE was used:

  * Components in `SW.aadl`

    * `SW.Impl`: non-cyber hardened software implementation
    * `SW.cyber_Impl`: cyber hardened with filter and monitor software implementation
    * `SWSystem.cyber_Impl`: BriefCASE transformed for seL4 cyber hardened software implementation (each thread is in its own process for scheduling on the seL4 verified micro-kernel)

  * Components in `MonitorTest.aadl`

    * `should_doNothing_when_noInput.test`
    * `should_alertAndNotOutput_when_responseWithoutRequest.test`
    * `should_notAlertAndOutput_when_responseAndRequest.test`
    * `should_notAlertAndOutput_when_responseOneStepAfterRequest.test`
    * `should_alertAndNotOutput_when_noResponseOneStepAfterRequest.test`
    * `should_passAgree_when_allBehaviorsAssumedAndGuaranteed.test` (correctness example for a **code contract**)

  * Components in `AlternativeTestFrom.aadl`

    * `Monitor_Thr.Impl`: separates the implementation from the contract (alternative from of a **code contract**)
    * `CASE_Monitor_Thr.Impl`: some code contract but with a top-level contract that has weaker properties leaning aspects of alert underspecified

SPLAT is run from the context menu or top-level menu with `BriefCASE`, `CyberResiliency`, `Synthesis Tools`, and then `Run SPLAT`.
Synthesis only works on the Linux distribution. The synthesized CakeML is included here for convenience:

  * `splat/SW_CASE_Filter_Thr.cml`: synthesized CakeML for the filter
  * `splat/SW_CASE_Monitor_Thr.cml`: synthesized CakeML for the monitor

# Example Overview

The model is an AADL architectural model of a software system (SW) for route planning and automated control of a UAV.
The system receives an automation request that is forwarded to a third-party route planner (AI).
The route planner decides the flight path of the UAV based on its current position and the requested task.
The waypoint manager (WM) receives the mission command as a set of waypoints from the planner and starts the UAV flying the mission, issuing waypoints to the UAV flight controller as the UAV location changes.
The waypoint manager is a legacy component that cannot be modified.

The expected behavior of the SW system, and the components implementing the system, are modeled with AGREE contract specifications.
AGREE is the verification tool in BriefCASE for hierarchical analysis of contract specifications.
The contracts constrain input with assumptions and state properties of output with guarantees.
AGREE performs model checking on this assume-guarantee system to hierarchically prove that SW obeys all contract obligations under all possible finite input streams.
The AI is *untrusted*, and AGREE proves the system implementation does not implement its contract specification in the presence of the untrusted AI.

The system designer uses BriefCASE to cyber-harden SW by inserting high-assurance components between the AI and downstream components in the form of a filter and a monitor.
A filter enforces an invariant over each datum in the data stream by not forwarding input to its output if that input violates the filter invariant.
The auto-generated AGREE specification states that only well-formed inputs are passed to the output.
The system developer must further define the filtering policy, but it is usually based on the existing assumptions made by downstream components that consume the filter output.

A monitor captures a relation on input data over time and is thus able to reason about temporal properties of that input.
A monitor raises an alert if the specified temporal properties are ever violated.
The AGREE specification for the monitor in our example states that an automation response can only be generated in conjunction with an automation request; and further, that response must come with the request or in the next step after the request.
As with the filter, the system designer provides the policy, and that policy is typically based on existing AGREE definitions in the SW system.
AGREE proves the cyber-hardened system implements its contract specification.

High-assurance components are automatically synthesized by SPLAT, another tool in BriefCASE, from the AGREE specifications to equivalent programs in the CakeML language.
The synthesis toolchain generates a proof that equates the meaning of the AGREE specification to the behavior of the CakeML program.
In other words, for any set of input streams that meet the component's contract assumptions, the output streams produced from the synthesized CakeML code exactly match those defined by the high-assurance component's guarantees.
CakeML provides verified compilation to binaries for several different platforms, meaning that the resulting binaries exactly preserve the meaning of the original CakeML code.

Preserving the input to output relationship of streams between the AGREE contracts and CakeML lifts the AGREE contract verification results to the actual deployed system.
If the contract model verification succeeds, then the meaning of those results hold for the deployed system.
These results, however, are only valid under additional assumptions on the deployed system:

  * contracts for non-synthesized components accurately model their deployed counterparts;
  * an appropriate schedule exists to sequence component execution following the dependent data-flow in the design; and
  * the communication fabric stitching components together works in harmony with the schedule.

## Files

  * `AI.aadl`: untrusted route planner with its AGREE specification
  * `AlternativeTestForms.aadl`: an example of an alternative form, contrasting with the test form, for separating out the component code-contract from the properties of the code contract for test and verification (not for synthesis)
  * `Filter.aadl`: high-assurance filter component inserted by BriefCASE with its AGREE specification
  * `Messages.aadl`: message format definitions
  * `Monitor.aadl`: high-assurance monitor component inserted by BriefCASE with its AGREE specification
  * `MonitorTests.aadl`: defined tests for the monitor code contract that give black-box coverage and branch coverage on the code contract (preferred from of code contract testing)
  * `NoOutput.aadl`: dummy component to generate no output on the alert in the unhardened system
  * `SW.aadl`: the unhardened and hardened software implementations including an implementation appropriate for the seL4 verified micro-kernel
  * `WaypointManager.aadl`: the legacy waypoint manager with its non-mutable AGREE specification
