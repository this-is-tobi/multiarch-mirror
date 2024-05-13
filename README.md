# Multiarch image mirror :twisted_rightwards_arrows:

The purpose of this repository is to create multi-arch images for official images not compatible with ARM64. 

## Explanation

Each application receives one or more matrices to list the different architectures that need to be created and a workflow file to perform certain actions necessary to enable cross-platform compatibility and image construction.

## Automation

The workflow runs daily (03:00 am) or can be triggered manually from the Github user interface.

> *__Notes:__ Before each application build, the CI compares the last tag of the official version with the last tag of the mirror image to check whether a new build should be triggered.*


## Images

| Application | Pull command                                                |
| ----------- | ----------------------------------------------------------- |
| Mattermost  | `docker pull ghcr.io/this-is-tobi/mirror/mattermost:latest` |
| Outline     | `docker pull ghcr.io/this-is-tobi/mirror/outline:latest`    |

> *__Notes:__ All images are mirrored with the official tag version.*
