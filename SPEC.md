# Project Specification

## Project Goal
Upskill a development team that feels insecure about Helm by building practical experience through creating an umbrella chart that abstracts their application components. The session will cover:
- Creating clean, extensible Helm umbrella charts with abstracted templating
- Best practices for values.yaml design that hides internal composition
- Investigation techniques for finding the right abstraction levels
- Essential Helm workflows (helm template, helm dry-run)
- Integration with ArgoCD, including how ArgoCD can override Helm capabilities

## Target Solution
An umbrella Helm chart that:
- Provides a clean, stable values.yaml interface that doesn't change when underlying mechanics evolve
- Abstracts multiple application components into reusable subcharts
- Demonstrates extensible architecture where new components can be added without modifying the values schema
- Includes practical examples of Helm templating patterns for abstraction
- Shows ArgoCD integration with parameter overrides

What we are trying to accomplish: "Would a developer who doesn't know Kubernetes still understand what this value does?"
If yes → it belongs in values.yaml. If no → it belongs in the template.

The solution will be built incrementally with guided lessons at each step to build team confidence.

## Key Components
- **Umbrella Chart**: Main chart that orchestrates subcomponents
- **Subcharts**: Modular components for different application parts (e.g., frontend, backend, database)
- **Values Abstraction Layer**: Templates that translate simple values into complex configurations
- **Extensibility Framework**: Chart structure that allows adding new components without values.yaml changes
- **Investigation Examples**: Sample apps to practice abstraction discovery
- **Workflow Demos**: Hands-on exercises with helm template and dry-run
- **ArgoCD Integration**: Configuration showing how ArgoCD can override Helm parameters

## Success Criteria
- Team can independently create and maintain Helm umbrella charts
- Values.yaml files are clean and hide internal complexity
- Team understands how to make charts extensible without breaking interfaces
- Team confidently uses helm template and dry-run in development workflows
- Team understands ArgoCD's role in Helm deployments and parameter overrides
- Team feels secure in their Helm knowledge and can troubleshoot issues

## Learning Areas
- Helm chart structure and templating fundamentals
- Abstraction patterns in Helm (helper templates, named templates)
- Umbrella chart design and subchart management
- Values.yaml design for clean interfaces
- Extensibility patterns (conditional includes, dynamic subcharts)
- Investigation methodology for finding right abstraction levels
- Helm CLI workflows (template, dry-run, upgrade)
- ArgoCD integration and parameter override mechanisms
