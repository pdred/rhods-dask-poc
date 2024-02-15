
# Table of Contents

1.  [Understanding Dask and its Components](#orgad0d168)
2.  [Preparing The OpenShift Environment](#orgc92b8f3)
    1.  [Create the Dask Scheduler](#orgfa628d8)
3.  [Creating a Service for the Dask Scheduler](#org7b2ebfd)
    1.  [Exposing the Dask Dashboard](#org8fd48ef)
    2.  [Get the Route URL](#org7623beb)
4.  [Create the Dask Worker](#org295a409)
5.  [Integrating Dask with Jypyter Notebook](#org0dbd0e9)
6.  [Final Thoughts](#orgc28d8d5)

Providing a Dask cluster for use by Jupyter Notebooks in an OpenShift environment, especially with the Red Hat OpenShift Data Science (RHODS) installed, involves several steps. This process leverages OpenShift&rsquo;s capabilities for orchestrating containers, alongside the flexibility of Jupyter notebooks for data science work. This document is aims to guide you through the setup, including the deployment of Dask components and the configuration within a Jupyter Notebook.


<a id="orgad0d168"></a>

# Understanding Dask and its Components

**Dask** is a flexible library for parallel computing in Python. It&rsquo;s designed to integrate seamlessly with the PyData stack, including pandas and NumPy, making it an excellent tool for data science tasks. A Dask cluster consists of a scheduler and workers. The scheduler manages task assignments to workers, which perform the computations.


<a id="orgc92b8f3"></a>

# Preparing The OpenShift Environment

You can deploy Dask using containers, which is ideal for OpenShift. The deployment includes setting up the Dask scheduler and worker pods.


<a id="orgfa628d8"></a>

## Create the Dask Scheduler

1.  Dockerfile for the Dask Scheduler
    ```Dockerfile
        FROM daskdev/dask:latest
        RUN pip install dask distributed --upgrade
        CMD ["dask-scheduler"]
    ```        

2.  **Build and Push the Docker Image**: Use the OpenShift CLI or a CI/CD pipeline to build this Docker image and push it to your container registry.

3.  **Deploy the Scheduler to OpenShift**: Create a deployment configuration in OpenShift for the Dask scheduler. In this example, we call the file ***dask-scheduler.yaml***
    ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: dask-scheduler
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: dask-scheduler
          template:
            metadata:
              labels:
                app: dask-scheduler
            spec:
              containers:
              - name: dask-scheduler
                image: <your-registry>/dask-scheduler:latest
                ports:
                - containerPort: 8786
    ``` 

4.  **Apply the Scheduler to your OpenShift Namespace**:
    ```bash
        oc apply -f dask-scheduler.yaml
    ```


<a id="org7b2ebfd"></a>

# Creating a Service for the Dask Scheduler

The Dask scheduler service will expose the scheduler pod, allowing workers and clients (such as a Jupyter notebook) to communicate with it.

1.  Dask Scheduler Service
    ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          name: dask-scheduler
        spec:
          ports:
          - port: 8786
            targetPort: 8786
            name: dashboard
          - port: 8787
            targetPort: 8787
            name: scheduler
          selector:
            app: dask-scheduler
    ```
    
    This service exposes two ports: 8786 for the scheduler and 8787 for the Dask dashboard, which provides insights into the cluster&rsquo;s performance and task execution.


<a id="org8fd48ef"></a>

## Exposing the Dask Dashboard

Creating a service inside of the OpenShift cluster, does not make the endpoint accessible to an external entity such as a web browser. To be able to reach the Dask Dashboard from a web browser, you will need to expose the service. In OpenShift, this process is called *creating a route*.

To create a route for the Dask Dashboard, perform the following:

1.  **Identify Your Service Name**: Ensure you know the exact name of the service that exposes the Dask Dashboard. If you followed the previous instructions, it should be dask-scheduler.

2.  **Create the Route**: Use the oc command to create a route. You&rsquo;ll specify the service name, target port, and optionally, a hostname. If you don&rsquo;t specify a hostname, OpenShift will generate one based on the project and cluster domain.
    ```bash
        oc expose service dask-scheduler --port=8787 --name=dask-dashboard
    ```
    
    In this command:
    
    -   *dask-scheduler* is the name of the service that exposes the Dask scheduler and dashboard.
    
    -   *&#x2013;port=8787* specifies the port of the service that the route will target, which is the dashboard port.
    
    -   *&#x2013;name=dask-dashboard* gives a name to the route, making it easier to identify.


<a id="org7623beb"></a>

## Get the Route URL

After creating the route, you can get the URL to access the Dask Dashboard externally:
```bash
    oc get route dask-dashboard -o=jsonpath='{.spec.host}'
  ```

This command will print the hostname assigned to your route. You can access the Dask Dashboard by navigating to <http://<hostname>> in your web browser, where <hostname> is the output from the above command.


<a id="org295a409"></a>

# Create the Dask Worker

1.  Dockerfile for the Dask Worker
    ```Dockerfile
        FROM daskdev/dask:latest
        RUN pip install dask distributed --upgrade
        CMD ["dask-worker", "tcp://dask-scheduler:8786"]
    ```

2.  **Build and Push the Docker Image for Workers**.

3.  **Deploy Workers to OpenShift**. In this example, we call the file ***dask-worker.yaml***
    ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: dask-worker
        spec:
          affinity:
            podAntiAffinity:
              preferredDuringSchedulingIgnoreDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchExpressions:
                    - key: app
                      operator: In
                      values:
                      - dask-worker
          replicas: 2 # Adjust based on needed compute resources
          selector:
            matchLabels:
              app: dask-worker
          template:
            metadata:
              labels:
                app: dask-worker
            spec:
              containers:
              - name: dask-worker
                image: <your-registry>/dask-worker:latest
    ```

4.  The **Pod Anti-Affinit** stanza under the spec defines a rule which will be checked during the instantiation of the pod. In this definition, we are looking for any newly created pod which has a label of ***app=dask-worker***. If found, we will make sure that the pod is not scheduled onto an OpenShift worker node which already has a pod with this same label.

5.  **Apply the Scheduler to your OpenShift Namespace**:
    ```bash
        oc apply -f dask-worker.yaml
      ```


<a id="org0dbd0e9"></a>

# Integrating Dask with Jypyter Notebook

Assuming Jupyter notebooks are deployed in your RHODS environment, you need to ensure that your notebook has access to Dask.

1.  **Install Dask in the Jupyter Notebook Environment**: If not already included, ensure the Dask package is installed in your notebook environment.
    ```python
        !pip install dask distributed
    ```

2.  **Connect to the Dask Cluster from a Jupyter Notebook**:
    ```python
        from dask.distributed import Client
        client = Client('tcp://dask-scheduler.ns.svc:8786') 
    ```
    
    -   In the code block above, *ns* represents the namespace in OpenShift where the *Dask Scheduler* resides. The *svc* tells OpenShift that this is an internal call.

3.  **Using Dask in your Notebook**: Now, you can use Dask to parallelize your data processing tasks directly from your Jupyter notebook.
    ```python
        import dask.array as da
        
        x = da.random.random((10000, 10000), chunks=(1000, 1000))
        y = x + x.T
        z = y.mean(axis=0)
        z.compute()
      ```


<a id="orgc28d8d5"></a>

# Final Thoughts

The steps outlined provide a framework for setting up a Dask cluster and integrating it with a Jupyter notebook for data science tasks. It leverages OpenShift&rsquo;s strengths in managing containerized applications and provides a scalable, flexible environment for data science workflows.

