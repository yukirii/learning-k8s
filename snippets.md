## contexts

### list contexts

```bash
kubectl config view -o jsonpath="{.contexts[*].name}:" | tr ":" "\n"
```

### show current context

```bash
kubectl config current-context
```

### use context

```
kubectl config use-context <context name>
```
