# Vald Similarity Search using chiVe Dataset

This example shows the text similarity search example with [chiVe](https://github.com/WorksApplications/chiVe) dataset.

It uses the Vald cluster for the search engine and Jupyter Notebook for running an example.

## Requirements

**_NOTE: It is recommended to do ["Get Started"](https://vald.vdaas.org/docs/tutorial/get-started/) before running Notebook._**

To execute this example, it requires the Vald cluster.

- [Vald](https://github.com/vdaas/vald)

And the following requirements will be installed when executing the example.

- [chiVe](https://github.com/WorksApplications/chiVe)
- [gRPC](https://grpc.io/)
- [Magnitude](https://github.com/plasticityai/magnitude)
- [vald-client-python](https://github.com/vdaas/vald-client-python)


## How it works

### Vald Installation

1. Prepare Kubernetes Cluster

   ```bash
   # verify
   kubectl get cluster-info
   ```

1. Add Vald charts to Helm repo

   ```bash
   helm repo add vald https://vald.vdaas.org/charts
   ```

1. Deploy the Vald cluster

   ```bash
   helm install vald vald/vald --values path/to/helm/values.yaml
   ```

   **NOTE: When using the chiVe dataset, please use [sample-values.yaml](./sample-values.yaml) or correct the following points.**

   ```yaml
   # edit path/to/helm/values.yaml
   agent:
     ngt:
       dimension: 300
       distance_type: cos
   ```

1. Verify

   ```bash
   kubectl get pods
   ```

### Run Jupyter Notebook on Docker

1. Download the dataset

   Before running the Docker image, please download the chiVe dataset applied for Magnitude.

   ```bash
   curl "https://sudachi.s3-ap-northeast-1.amazonaws.com/chive/chive-1.2-mc90.magnitude" -o "chive-1.2-mc90.magnitude"
   ```

1. Verify the endpoint of Vald cluster

   This example requires the Vald cluster endpoint to send requests from Jupyter Notebook.
   Please verify your cluster endpoint.

   - If Kubernetes ingress is enabled, you can use ingress host and port.

     ```bash
     kubectl get ingress
     ```

   - If disabled, you can use the endpoint by executing `kubectl port-forward`.
     ```bash
     # port-forward (the endpoint will be {host ip}:8081)
     kubectl port-forward svc/vald-lb-gateway 8081:8081
     ```

1. Run Jupyter Notebook on Docker

   ```bash
   # use python-3.7.6 image because Magnitude DOES NOT apply new python version. (2022-06)
   docker run --user root -it -v $(pwd):/home/jovyan/work -p 8888:8888 -e UB_UID=root -e GRANT_SUDO=yes jupyter/datascience-notebook:python-3.7.6
   ```

### Execute example

1. Access via browser

1. Select Notebook and execute the example

<div align="center">
    <img src="./chive-demo.gif" width="100%" />
</div>

### Cleanup

1. Stop the Docker container by `Ctrl-C`.

1. Delete Vald cluster by `helm uninstall vald` or according to the method you deployed.
