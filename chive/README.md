# Vald Similarity Search using chiVe Dataset

## Requirements

- [chiVe](https://github.com/WorksApplications/chiVe)
- [gRPC](https://grpc.io/)
- [Magnitude](https://github.com/plasticityai/magnitude)
- [Vald](https://github.com/vdaas/vald)
- [vald-client-python](https://github.com/vdaas/vald-client-python)

___NOTE: It is recommended to do ["Get Started"](https://vald.vdaas.org/docs/tutorial/get-started/) before running notebook.___

## Vald Installation

### Using Helm

```
helm repo add vald https://vald.vdaas.org/charts
helm install vald vald/vald --values path/to/helm/values.yaml
```

___NOTE: When using the chiVe dataset, please use sample-values.yaml or correct the following points.___

``` path/to/helm/values.yaml
agent:
  ngt:
    dimension: 300
    distance_type: cos
```

### run docker

```
docker run --user root -it -v $(pwd):/home/jovyan/work -p 8888:8888 -e UB_UID=root -e GRANT_SUDO=yes jupyter/datascience-notebook:python-3.7.6
```
