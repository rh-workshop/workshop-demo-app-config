# Connectivity Link applications

`bank-api` and `mock-llm` are built and signed by the existing `ci-build-image` pipeline on the hub. Their source and Containerfiles live in the application monorepo.

These overlays are not registered as Argo CD Applications. The pipeline writes each signed digest here after its existing quality and security gates pass. The initial `pending-ci` reference is deliberately not deployable. Do not sync these overlays before a successful pipeline execution.

Validation namespace: `atp-pipeline-validation`. No public Route or HTTPRoute is created by these overlays. Gateway policies and participant isolation are configured separately by the Connectivity Link workshop.
