## Testing things

https://book.kubebuilder.io/cronjob-tutorial/running.html

> To test out the controller, we can run it locally against the cluster. Before we do so, though, we’ll need to install our CRDs, as per the quick start. This will automatically update the YAML manifests using controller-tools, if needed:

```bash
make install
```

>At this point, we need a SumJob to test with. Let’s write a sample to `config/samples/batch_v1_sumjob.yaml`, and use that:

```
apiVersion: batch.tutorial.kubebuilder.io/v1
kind: SumJob
metadata:
  name: sumjob-sample2
spec:
  # Add fields here
  A: 11
  B: 15
```{{copy}}

Then simply create our custom resource:

```bash
kubectl create -f config/samples/batch_v1_cronjob.yaml
```{{copy}}

Check that the resource has been created

```bash
kubectl get sumjob
```{{copy}}

Execute one reconcile cycle, then press Ctrl+C to exit:

```bash
make run ENABLE_WEBHOOKS=false
```{{copy}}

Check your job with:

```bash
kubectl describe sumjob sumjob-sample2
```{{copy}}

