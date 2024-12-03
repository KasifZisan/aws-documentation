# Infrastructure as Code (IaC)
Infrastructure as code (IaC) is the ability to provision and support your computing infrastructure using code instead of manual processes and settings.

The most common use of IaC is in software development to build, test, and deploy applications.

#### Benefits

- **Easily duplicate environments:** The same environment can be deployed on a different system in another location using the same IaC, as long as the infrastructure resources are available.
- **Reduce Configuration error:** Manual configuration is error-prone due to human involvement. People make mistakes. Or there could be configuration drift due to changes in one setup (like a developer environment) that was missed in another setup (like a test environment).
In contrast, IaC reduces errors and streamlines error checking. If there are errors due to IaC code updates, you can quickly fix the situation by rolling the codebase to the last known stable configuration files. 
- **Iterate on best practice environments:** Source control allows software developers to easily build and branch on environments.

#### How does Infrastructure as Code work
Infrastructure as code (IaC) describes a system architecture and how it works. An infrastructure architecture contains resources such as servers, networking, operating systems, and storage. IaC controls virtualized resources by treating configuration files like source code files. You can use it to manage infrastructure in a codified, repeatable way. 

IaC configuration management tools use different language specifications. You can develop IaC similar to application code in Python or Java. You also write the IaC in an integrated development environment (IDE) with built-in error checking.

#### Approaches to IaC

- **Declarative:** Declarative IaC allows a developer to describe resources and settings that make up the end state of a desired system. The IaC solution then creates this system from the infrastructure code. 
- **Imperative:** Imperative IaC allows a developer to describe all the steps to set up the resources and get to the desired system and running state. The imperative approach becomes necessary in complex infrastructure deployments. This is especially true when the order of events is critical.

