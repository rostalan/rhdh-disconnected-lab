# Disconnected RHDH 2.1 and OSL 1.39 lab

This repository tracks the reproducible inputs for a local disconnected test of
Red Hat Developer Hub 2.1 with Orchestrator and OpenShift Serverless Logic
1.39. It does not track generated archives, image mirrors, plugin exports, or
local registry data.

Read [the end-to-end guide](rhdh-2.1-osl-1.39-disconnected-e2e-guide.md).
The `disconnected-lab-assets/config` directory contains the versioned image-set and Helm
values inputs.

Do not commit pull secrets, registry credentials, or CRC login material.
