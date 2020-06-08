## Testing things

https://book.kubebuilder.io/cronjob-tutorial/running.html

> To test out the controller, we can run it locally against the cluster. Before we do so, though, we’ll need to install our CRDs, as per the quick start. This will automatically update the YAML manifests using controller-tools, if needed:

```bash
make install
make run ENABLE_WEBHOOKS=false
```

>At this point, we need a CronJob to test with. Let’s write a sample to `config/samples/batch_v1_cronjob.yaml`, and use that:

```yaml
apiVersion: batch.tutorial.kubebuilder.io/v1
kind: CronJob
metadata:
  name: cronjob-sample
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 60
  concurrencyPolicy: Allow # explicitly specify, but Allow is also default.
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

```bash
kubectl create -f config/samples/batch_v1_cronjob.yaml
```

Check you jobs with:

```bash
kubectl get cronjob.batch.tutorial.kubebuilder.io -o yaml
kubectl get job
```

