# Docker ENTRYPOINT and CMD

## Dockerfile

``` dockerfile
FROM Ubuntu

ENTRYPOINT ["sleep"]

CMD ["5"]
```

## Kubernetes Pod YAML

``` yaml
spec:
  containers:
    - name: ubuntu-sleeper
      image: ubuntu-sleeper
      command: ["sleep2.0"]
      args: ["10"]
```

## Mapping

-   Docker `ENTRYPOINT` → Kubernetes `command`
-   Docker `CMD` → Kubernetes `args`
