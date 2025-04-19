# Demoing the Tornjak Manager

1. Create Kind cluster
2. Deploy SPIRE + Tornjak on the cluster
3. Deploy the Tornjak manager
4. Access the Tornjak manager UI

We will mimic connectivity via port-forwarding. 

----------

## A step-by-step tutorial for locally demonstrating federation between Kind clusters

### Step 0: Requirements

- kubectl 
- kind (this tutorial was tested with podman and kind)
- helm

If you are using Podman, you will need to set the `KIND_EXPERIMENTAL_PROVIDER`:

```
export KIND_EXPERIMENTAL_PROVIDER=podman
```

> **TIP:** This environment variable tells KIND (Kubernetes in Docker) to use Podman as the container runtime instead of Docker. Podman is a daemonless container engine that can run without root privileges, making it more secure for some environments. KIND's Podman support is considered experimental, so you may encounter some issues not present when using Docker. If you experience problems, consider switching to Docker or check the KIND documentation for Podman-specific troubleshooting.

### Step 1: Create the Kind clusters

For the purposes of this, we can name the clusters `server` and `client`:

```
kind create cluster
```

> **TIP:** The `kind create cluster` command sets up a complete Kubernetes cluster on your local machine using containers to simulate cluster nodes. By default, this creates a single-node cluster named "kind" with both control plane and worker functionality. This command may take several minutes to complete as it downloads the necessary container images and initializes the Kubernetes components. You can verify your cluster is running with `kubectl cluster-info`.

#### Common Issues

> **ERROR:** `ERROR: failed to create cluster: node(s) already exist for a cluster with the name "kind"`
>
> **SOLUTION:** This means you already have a KIND cluster with the default name running. You can:
> 
> 1. Use the existing cluster (if it's suitable for your needs)
> 2. Delete the existing cluster with `kind delete cluster`
> 3. Create a new cluster with a different name using `kind create cluster --name tornjak-demo`

### Step 2: Deploy SPIRE + Tornjak via Helm

We will deploy SPIRE and Tornjak via the Helm charts. 

#### The Custom Helm Values

There are two things to note of the configurations of the SPIRE server:

1. **The trust domains are configured to be different.** If this is not true, then the SPIRE servers will not be able to federate. 
2. **controllerManager identities is set with a federatesWith field.** The SPIRE controller manager automatically creates workload entries when pods are created in the cluster. Setting this field causes all workload entries to automatically receive the trust bundle of the other trust domain. 

Deploy with the following commands:

```
helm upgrade --install -n spire-mgmt spire-crds spire-crds --repo https://spiffe.github.io/helm-charts-hardened/ --create-namespace
helm upgrade --install -n spire-mgmt spire spire --repo https://spiffe.github.io/helm-charts-hardened/ -f helm_values.yaml
```

#### Understanding the Helm Commands:

> 1. The first command installs the SPIRE CRDs (Custom Resource Definitions) from a remote Helm repository. CRDs are like blueprints that extend Kubernetes with new types of resources. This command must be run first because the actual SPIRE components need these definitions to exist before they can be created.
>
> 2. The second command requires the a local file named helm_vales.yaml, so ensure it is in your current working directory when running the second command, or use the full path to the file. The purpose of the command is to install the actual SPIRE and Tornjak components using your configuration values. This includes the SPIRE server, agents, and the Tornjak management interface.

### Step 2.5: Expose the Tornjak backend Endpoints

Open a new terminal window for the following command to expose the Tornjak backend endpoints:

```
kubectl port-forward  -n spire-server svc/spire-tornjak-backend 10000:10000
```

> **TIP:** The `kubectl port-forward` command creates a temporary network connection between your local machine and a service running inside your Kubernetes cluster. In this case, it connects port 10000 on your local machine to port 10000 of the Tornjak backend service running in the cluster. This allows you to access the Tornjak API at http://localhost:10000 from your local machine without needing to set up more complex networking like Ingress or LoadBalancer services.
> 
> This command must be run in a separate terminal window because it will continue running in the foreground until you press Ctrl+C to stop it. You'll need to keep this terminal open while you're using Tornjak.
> 
> **Common Issues:**
> - If you see an error like `Error from server (NotFound): services "spire-tornjak-backend" not found`, make sure you're using the correct namespace with `-n spire-server`.
> - If you see `unable to listen on port 10000: Listeners failed to create with the following errors: [unable to create listener: Error listen tcp4 127.0.0.1:10000: bind: address already in use]`, it means port 10000 is already being used by another process. You can use a different local port like `kubectl port-forward -n spire-server svc/spire-tornjak-backend 20000:10000` which would make the service available at http://localhost:20000 instead.


### Step 3: Run the Tornjak Manager

Go to the Tornjak repo and run the following command:

```
go run cmd/manager/main.go 
```

This opens a port on localhost:50000. Open the browser and go to `http://localhost:50000/manager-api/server/list` to verify. 

> **TIP:** The command `go run cmd/manager/main.go` should be run from the root directory of the Tornjak repository, not from within the `cmd/manager` directory itself.
>
> **TIP for Windows Users**: If you're running Go on Windows, it's recommended to run go run main.go from WSL (Windows Subsystem for Linux) instead. WSL has the necessary tools like gcc pre-installed, which is required for compiling certain Go dependencies (e.g., go-sqlite3). Running in WSL also avoids some of the common issues with Windows permissions and Go's native builds.
>
> **Common Issues:**
>
> **Windows Permission Errors:** On Windows, you might encounter "Access is denied" errors when trying to run the Go application. To resolve this:
>
> 1. Try running your terminal/command prompt as Administrator
> 2. Check if any antivirus software is blocking execution - for me my Spectrum Security Suite was blocking main.go from running
> 3. Ensure you have proper permissions to the directory
> 4. If using Windows Defender or another security tool, you might need to add an exception
> 5. Alternatively, you can build the binary first with `go build cmd/manager/main.go` and then run the resulting executable

#### Common Troubleshooting issues:


### Step 3.5: Make a Tornjak Manager API Call

The manager acts as a registry. You can view the registered Tornjak servers with this call:

```
curl http://localhost:50000/manager-api/server-list
```

To register the server we deployed, use the following call:

```
curl http://localhost:50000/manager-api/server/register --header "Content-Type: application/json" --data '{"name": "backend", "address": "http://localhost:10000"}'
```

Finally, to run a backend call, here's an example:

```
curl http://localhost:50000/manager-api/tornjak/serverinfo/backend
```

> **TIP:** When visiting [http://localhost:50000/manager-api/server/list](http://localhost:50000/manager-api/server/list), it should look something like this:

```json
{
  "servers": [
    {
      "name": "backend",
      "address": "http://localhost:10000",
      "tls": false,
      "mtls": false
    }
  ]
}
```

### Step 4: Run the UI

Finally, we can run the UI. From the root Tornjak directory:

```
cd frontend
REACT_APP_API_SERVER_URI=http://localhost:50000/ REACT_APP_TORNJAK_MANAGER=true npm start
```

### Cleanup

Running the manager in local mode creates a local DB file which we can remove:

```
rm -r serverlocaldb
```

```
kind delete cluster
```

