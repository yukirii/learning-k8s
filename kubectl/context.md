## contexts

useful tool: https://github.com/ahmetb/kubectx

### list contexts

```bash
kubectl config view -o jsonpath="{.contexts[*].name}:" | sed "s/\s\|\:/\n/g"
```

### show current context

```bash
kubectl config current-context
```

### use context

```
kubectl config use-context <context name>
```
