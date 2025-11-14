![Logo](https://cdn.gurobi.com/wp-content/uploads/GurobiLogo_Black.svg "Gurobi Optimization")
# Quick reference
Maintained by: [Gurobi Optimization](https://www.gurobi.com)

Where to get help: [Gurobi Support](https://www.gurobi.com/support/), [Gurobi Documentation](https://www.gurobi.com/documentation/)

Gurobi images:
- [gurobi/optimizer](https://hub.docker.com/r/gurobi/optimizer): Gurobi Optimizer (full distribution)
- [gurobi/python](https://hub.docker.com/r/gurobi/python): Gurobi Optimizer (Python API only)
- [gurobi/python-example](https://hub.docker.com/r/gurobi/python-example): Gurobi Optimizer example in Python with a WLS license
- [gurobi/modeling-examples](https://hub.docker.com/r/gurobi/modeling-examples): Optimization modeling examples (distributed as Jupyter Notebooks)
- [gurobi/compute](https://hub.docker.com/r/gurobi/compute): Gurobi Compute Server
- [gurobi/manager](https://hub.docker.com/r/gurobi/manager): Gurobi Cluster Manager

# Supported tags

* [0.0.5, latest](https://github.com/Gurobi/docker-manager-operator/blob/main/0.0.5)

# Quick reference (cont.)

Supported architectures: linux/amd64, linux/arm64

Published image artifact details: https://github.com/Gurobi/docker-manager-operator


# What is `gurobi/manager-operator`?

We introduced a Kubernetes Autoscaler Operator for Gurobi Compute Servers that provides configurable autoscaling to align the number of pods in a cluster to the current workload. The operator monitors job queues, waiting times, and node utilization, and adjusts deployments up or down as needed. Scale-down is handled gracefully to avoid interrupting running jobs.
The scaling logic can be modified through parameters, giving users control over how decisions are made. The operator is published on DockerHub and uses the existing Cluster Manager REST APIs, ensuring backwards compatibility with previous Gurobi Compute Server versions.

For detailed information, please refer to the [Gurobi Operator documentation](https://github.com/Gurobi/docker-manager-operator/blob/main/reference.md).

# License

By downloading and using this image, you agree with the
[End-User License Agreement](https://www.gurobi.com/EULA) for the Gurobi software contained in this image.

As with all Docker images, these likely also contain other software which may be under other
licenses (such as Bash, etc from the base distribution, along with any direct or indirect
dependencies of the primary software being contained).

As for any pre-built image usage, it is the image user's responsibility to ensure that any use
of this image complies with any relevant licenses for all software contained within.
