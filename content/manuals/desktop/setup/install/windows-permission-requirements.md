---
description: Understand permission requirements for Docker Desktop for Windows
keywords: Docker Desktop, Windows, security, install
title: Understand permission requirements for Windows
linkTitle: Windows permission requirements
aliases:
- /desktop/windows/privileged-helper/
- /desktop/windows/permission-requirements/
- /desktop/install/windows-permission-requirements/
weight: 40
---

This page contains information about the permission requirements for running and installing Docker Desktop on Windows, the functionality of the privileged helper process `com.docker.service`, and the reasoning behind this approach.

It also provides clarity on running containers as `root` as opposed to having `Administrator` access on the host and the privileges of the Windows Docker engine and Windows containers.

Docker Desktop on Windows is designed with security in mind. Administrative rights are only required when absolutely necessary.

## Permission requirements

While Docker Desktop on Windows can be run without having `Administrator` privileges, it does require them during installation. On installation you receive a UAC prompt which allows a privileged helper service to be installed. After that, Docker Desktop can be run without administrator privileges. 

Running Docker Desktop on Windows without the privileged helper does not require users to have `docker-users` group membership. However,
some features that require privileged operations will have this requirement. 

If you performed the installation, you are automatically added to this group, but other users must be added manually. This allows the administrator to control who has access to features that require higher privileges, such as creating and managing the Hyper-V VM, or using Windows containers.

When Docker Desktop launches, all non-privileged named pipes are created so that only the following users can access them:
- The user that launched Docker Desktop.
- Members of the local `Administrators` group.
- The `LOCALSYSTEM` account.

## Privileged helper

Docker Desktop needs to perform a limited set of privileged operations which are conducted by the privileged helper process `com.docker.service`. This approach allows, following the principle of least privilege, `Administrator` access to be used only for the operations for which it is absolutely necessary, while still being able to use Docker Desktop as an unprivileged user.


The privileged helper `com.docker.service` is a Windows service which runs in the background with `SYSTEM` privileges. It listens on the named pipe `//./pipe/dockerBackendV2`. The developer runs the Docker Desktop application, which connects to the named pipe and sends commands to the service. This named pipe is protected, and only users that are part of the `docker-users` group can have access to it.

The service performs the following functionalities:
- Ensuring that `kubernetes.docker.internal` is defined in the Win32 hosts file. Defining the DNS name `kubernetes.docker.internal` allows Docker to share Kubernetes contexts with containers.
- Ensuring that `host.docker.internal` and `gateway.docker.internal` are defined in the Win32 hosts file. They point to the host local IP address and allow an application to resolve the host IP using the same name from either the host itself or a container.
- Securely caching the Registry Access Management policy which is read-only for the developer.
- Creating the Hyper-V VM `"DockerDesktopVM"` and managing its lifecycle - starting, stopping, and destroying it. The VM name is hard coded in the service code so the service cannot be used for creating or manipulating any other VMs.
- Moving the VHDX file or folder.
- Starting and stopping the Windows Docker engine and querying whether it's running.
- Deleting all Windows containers data files.
- Checking if Hyper-V is enabled.
- Checking if the bootloader activates Hyper-V.
- Checking if required Windows features are both installed and enabled.
- Conducting healthchecks and retrieving the version of the service itself.

The service start mode depends on which container engine is selected, and, for WSL, on whether it is needed to maintain `host.docker.internal` and `gateway.docker.internal` in the Win32 hosts file. This is controlled by a setting under `Use the WSL 2 based engine` in the settings page. When this is set, WSL engine behaves the same as Hyper-V. So:
- With Windows containers, or Hyper-v Linux containers, the service is started when the system boots and runs all the time, even when Docker Desktop isn't running. This is required so you can launch Docker Desktop without admin privileges.
- With WSL2 Linux containers, the service isn't necessary and therefore doesn't run automatically when the system boots. When you switch to Windows containers or Hyper-V Linux containers, or choose to maintain `host.docker.internal` and `gateway.docker.internal` in the Win32 hosts file, a UAC prompt appears asking you to accept the privileged operation to start the service. If accepted, the service is started and set to start automatically upon the next Windows boot.

## Containers running as root within the Linux VM

The Linux Docker daemon and containers run in a minimal, special-purpose Linux
VM managed by Docker. It is immutable so you can’t extend it or change the
installed software.  This means that although containers run by default as
`root`, this doesn't allow altering the VM and doesn't grant `Administrator`
access to the Windows host machine. The Linux VM serves as a security boundary
and limits what resources from the host can be accessed. File sharing uses a
user-space crafted file server and any directories from the host bind mounted
into Docker containers still retain their original permissions.  Containers don't have access to any host files beyond those explicitly shared.

## Enhanced Container Isolation

In addition, Docker Desktop supports [Enhanced Container Isolation
mode](/manuals/enterprise/security/hardened-desktop/enhanced-container-isolation/_index.md) (ECI),
available to Business customers only, which further secures containers without
impacting developer workflows.

ECI automatically runs all containers within a Linux user-namespace, such that
root in the container is mapped to an unprivileged user inside the Docker
Desktop VM. ECI uses this and other advanced techniques to further secure
containers within the Docker Desktop Linux VM, such that they are further
isolated from the Docker daemon and other services running inside the VM.

## Windows containers

> [!WARNING]
>
> Enabling Windows containers has important security implications.

Unlike the Linux Docker Engine and containers which run in a VM, Windows containers are implemented using operating system features, and run directly on the Windows host. If you enable Windows containers during installation, the `ContainerAdministrator` user used for administration inside the container is a local administrator on the host machine. Enabling Windows containers during installation makes it so that members of the `docker-users` group are able to elevate to administrators on the host. For organizations who don't want their developers to run Windows containers, a `-–no-windows-containers` installer flag is available to disable their use.

## Networking

For network connectivity, Docker Desktop uses a user-space process (`vpnkit`), which inherits constraints like firewall rules, VPN, HTTP proxy properties etc. from the user that launched it.
