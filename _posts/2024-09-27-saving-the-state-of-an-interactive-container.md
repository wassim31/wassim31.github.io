---
layout: post
title: "Saving the state of an interactive container"
date: 2024-09-27 16:16:35 +0000
description: "Sometimes we need to start a docker container in an interactive shell for testing purposes , and..."
canonical_devto: https://dev.to/wassim31/saving-the-state-of-an-interactive-container-3pnj
image: /images/posts/saving-the-state-of-an-interactive-container/cover.jpeg
---

Sometimes we need to start a docker container in an interactive shell for testing purposes , and during the process you might wrote some application code , downloaded some big chunks of files, or configured the environment according to your use-cases , but starting the container interactively will make it destroyed once an interrupt is done ( 0 or -1 ) .

`docker run -it --rm --mount type=bind,source="$(pwd)"/work,target=/work -p 8888:8888 opencvcourses/opencv:440`

`-it` starts an interactive shell. This switch is always needed to start the container. Otherwise, it will start and stop instantly.
`--rm` specifies to kill the container after it is exited.
`--mount` creates persistent storage to save all the work. Read more about it here⁠.
`-p` is used to expose a container's port to the host.
**Note: Run this command in the parent directory of work folder**

you can save the state by commiting the changes to the pulled image.
` $ docker ps`

` $ docker commit <container_id>  repo/testimage:version3 `


While this will work, this approach is considered poor practice by the Docker community. Best way to go is using a Dockerfile.

Yeah, I think that's the right way for production for sure. But many of us are just using docker locally as a slimmed down version of virtualbox, and just want the damn state saved, exactly as it is. And don't want extra volumes on our local drives either
