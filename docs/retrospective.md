# Retrospective

An honest look at what's working and what isn't, as of 2025-04-14.

## What works well

- Having automated tests gives us more confidence in the code that we deploy.
- Using TcUnit pragmatically for the library works well.
- Using pytest with OPC-UA to call various PLC functions and check results against a VC model using a Virtual Commissioning software such as Process Simulate or iPhysics works well.
- Library management works using the TwinCAT Library Repository, although the developer needs to know which branch to pull if they are using a library version that is in development and not yet on `main`.
- We use TwinCAT Variant Management to handle variations in hardware across machine types. This works fairly well although there is always the concern about an explosion in variations as hardware changes get introduced. We have 150+ PLCs/machines with several different variants deployed.
- Originally I tried to use Automation Interface to directly Activate Configuration on target PLCs. I realized it works better to create an artifact and use file-based methods to deploy the code. This is beneficial because it is much faster for parallel deployments (you don't need to wait for TwinCAT XAE to start, create the route, and Activate Configuration).

## What could be improved

- I tried to incorporate static analysis, however it doesn't work very well with Automation Interface on 3.1.4024. I believe some issues have been fixed on 4026 but we don't plan to upgrade for a while.
- We have to use licensed IPCs to run tests, because TwinCAT doesn't offer an official way to access the usermode runtime for test purposes (you can do it but you need to manually enter the trial license key every 7 days).
- Sometimes Automation Interface has bugs. When it fails you need to login to the build agent and see what is failing, and this isn't very accessible to the PLC programmers that just want their changes to get merged.
- To build and test a PLC or library takes around 5 minutes. Since this happens fairly infrequently (only on changes to PR), it hasn't been an issue. If it becomes an issue we could look at using Automation Interface to directly deploy the code, leaving Visual Studio open all the time to reduce loading times.
