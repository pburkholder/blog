[An email I sent today, expanded into a veritable rant...]

Too many proprietary monitoring systems are thrust upon operations teams based upon a cursory feature set evaluation, with little attention paid to how to deploy, manage, update and otherwise tend to the long-term operational needs within a technical infrastructure.

A typical example is the AppDynamics APM which doesn't version their software agent, or make it available as a system-level package, so there's no OOB way for a DevOps engineer using a configuration management system to upgrade the agent across systems. (I should make a blog post on this point...). 

Such systems then go underprovisioned, underutilized, and eventually the contract is dropped. Admittedly this is based on only a few datapoints, but I wouldn't be surprised if it's a typical pattern.

## External Monitoring

External monitoring systems have similar issues. I like synthetic monitors, especially for newer applications that may not have enough natural users for real-user-monitoring systems. But _versioning_ monitors matters to me as much as version applications: Was it the monitor that crashed the app? Did we introduce a change that resulted in false positives? Etc. etc.  Versioning matters.

Last year I could not find any external distributed monitoring services that let you specify a transactional monitor as version-able code except NeuStar, which was insanely expensive, or CA's NimSoft, which locked one into 1990s JUnit code. In the end I used PingDom until I could write my own #casperjs and #sensuapp system to bring it all in-house. 

I have lots of #monitoringlove now thanks to the open-source world. Will the commercial world ever get there?