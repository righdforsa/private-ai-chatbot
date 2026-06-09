Purpose of this repo is to practice using Kubernetes in deployment as well as AI services in a pseudo production environment.

ARCHITECTURE.md - high and mid-level description of the environment and what is in-scope vs. out-of-scope.

TODOS:
- set up CD with secrets management so a push to the repo is automatically applied. I want to run it by hand for a while first.

ERRATA:
- if this were a real environment, I would want to control my own Kubernetes control-plane server, because I don't like giving up control of the critical path to managing the environment.
