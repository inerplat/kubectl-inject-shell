# kubectl-debug-inject

`kubectl-debug-inject` is a powerful script designed to inject a BusyBox environment into a running Kubernetes container, even in environments that lack a shell (e.g., distroless images). The script creates a privileged container on the same node as the target pod, enabling debugging and accessing the container's filesystem. You can specify various options such as the namespace, image, container, and more for customized execution.

## Features
- Inject a debugging environment (using BusyBox) into a target container.
- Run commands in containers that lack a shell or essential binaries (e.g., distroless images).
- Copy BusyBox binaries into the target container's root filesystem.
- Create a privileged container that can access the target container's filesystem.
- Use an alternative image for debugging if necessary.
- Verbose logging for debugging purposes.

## Requirements
- A Kubernetes cluster with `kubectl` installed and configured.
- Privileged container creation permissions in the Kubernetes cluster.

## Usage

```bash
kubectl-debug-inject [options] <pod-name>
```

### Options
| Option                      | Description                                                                 |
|------------------------------|-----------------------------------------------------------------------------|
| `-n, --namespace <namespace>` | Specify the namespace of the pod (default: 'default').                      |
| `-i, --image <image>`         | Specify the image to use for the privileged container (default: 'alpine').  |
| `-c, --container <container>` | Specify the name of the target container (default: first container in pod).  |
| `-d, --daemon`                | Enable daemon mode. Only creates a privileged job without exec-ing into it. |
| `-v, --verbose`               | Enable verbose logging for detailed output.                                 |
| `--kubeconfig <file>`         | Use a specific kubeconfig file for `kubectl` commands.                      |
| `--context <context>`         | Specify the `kubectl` context to use.                                       |
| `-h, --help`                  | Display the help message.                                                   |

### Arguments
| Argument   | Description                                                |
|------------|------------------------------------------------------------|
| `<pod-name>` | The name of the pod to inject the debugging environment into. |

### Example Usage

1. Inject a debugging container into a pod:
   ```bash
   kubectl-debug-inject -n my-namespace my-pod
   ```

2. Specify a custom image for the privileged container:
   ```bash
   kubectl-debug-inject -n my-namespace -i my-image my-pod
   ```

3. Enable daemon mode:
   ```bash
   kubectl-debug-inject --daemon my-pod
   ```

4. Use a specific `kubeconfig` and context:
   ```bash
   kubectl-debug-inject --kubeconfig /path/to/kubeconfig --context my-context my-pod
   ```

## How It Works
```
+---------------------------------------------------------------+
|                            Host OS                            |
|                                                               |
|  +-------------------------------+     +-------------------+  |
|  |   Privileged Container Job    |     |  Target Container |  |
|  |         (alpine image)        |     |                   |  |
|  |  +-------------------------+  |     |                   |  |
|  |  |       /bin/busybox      |  |     |                   |  |
|  |  |   /lib/ld.musl-*.so.1   |  |     |                   |  |
|  |  +------------|------------+  |     |  +--------------+ |  |
|  |               |               |     |  |              | |  |
|  |               | (copy)        |     |  |      /       | |  |
|  |               V               |     |  |              | |  |
|  |  +-----[hostPath mount]----+  |     |  +------+-------+ |  |
|  |  | /host/run/containerd    |  |     |         |         |  |
|  |  | /io/k8s.io/<ID>/rootfs  |  |     |         |         |  |
|  |  +------------+------------+  |     |         |         |  |
|  +---------------|---------------+     +---------|---------+  |
|                  |                               |            |
|                  |                               |            |
|                  +-------------------------------+            |
|                /run/containerd/io/k8s.io/<ID>/rootfs          |
+---------------------------------------------------------------+

```
The `kubectl-debug-inject` script performs the following steps:

1. **Retrieve container information:** 
   The script fetches the container ID of the target pod's container, either specified or defaulting to the first container.

2. **Create a privileged container:** 
   A privileged container is created on the same node as the target pod with access to the host's filesystem (`/host` mount).

3. **Inject BusyBox environment:**
   The script copies BusyBox binaries and essential libraries into the target container's root filesystem. Additionally, it creates symbolic links for the BusyBox utilities in the container's `/bin/` directory.

4. **Execution modes:**
   - In non-daemon mode, the script waits for the privileged container to finish its task and then opens a shell in the target container.
   - In daemon mode, it simply creates the privileged container without opening a shell.

## Verbose Logging

Enable verbose mode using the `-v` or `--verbose` flag to get detailed output on the operations being performed.

## Notes

- The script operates based on `musl libc` for compatibility and injects all essential BusyBox utilities.
- It works seamlessly with containers that lack basic shell functionality, such as distroless images.
- Make sure you have the necessary privileges to create privileged containers in your Kubernetes cluster.

## License

This project is licensed under the MIT License.

--- 

