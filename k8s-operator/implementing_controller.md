## Implementing cronjob controller

From https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html:

> We’ll start out with some imports. You’ll see below that we’ll need a few more imports than those scaffolded for us. We’ll talk about each one when we use it.

<pre><code class="go">
package controllers

import (
    "context"
    "fmt"
    "sort"
    "time"

    "github.com/go-logr/logr"
    "github.com/robfig/cron"
    kbatch "k8s.io/api/batch/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ref "k8s.io/client-go/tools/reference"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    batch "tutorial.kubebuilder.io/project/api/v1"
)
</code></pre>

> Next, we’ll need a Clock, which will allow us to fake timing in our tests.

<pre><code class="go">
type realClock struct{}

func (_ realClock) Now() time.Time { return time.Now() }

// clock knows how to get the current time.
// It can be used to fake out timing for testing.
type Clock interface {
    Now() time.Time
}

type CronJobReconciler struct {
    client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
    Clock
}
</code></pre>


> Notice that we need a few more RBAC permissions -- since we’re creating and managing jobs now, we’ll need permissions for those, which means adding a couple more markers.

<pre><code class="go">
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=batch,resources=jobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch,resources=jobs/status,verbs=get
</code></pre>


> Now, we get to the heart of the controller -- the reconciler logic.

<pre><code class="go">
var (
    scheduledTimeAnnotation = "batch.tutorial.kubebuilder.io/scheduled-at"
)

func (r *CronJobReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
    ctx := context.Background()
    log := r.Log.WithValues("cronjob", req.NamespacedName)
</code></pre>

>We’ll fetch the CronJob using our client. All client methods take a context (to allow for cancellation) as their first argument, and the object in question as their last. Get is a bit special, in that it takes a NamespacedName as the middle argument (most don’t have a middle argument, as we’ll see below).

>Many client methods also take variadic options at the end.

<pre><code class="go">
    var cronJob batch.CronJob
    if err := r.Get(ctx, req.NamespacedName, &cronJob); err != nil {
        log.Error(err, "unable to fetch CronJob")
        // we'll ignore not-found errors, since they can't be fixed by an immediate
        // requeue (we'll need to wait for a new notification), and we can get them
        // on deleted requests.
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
</code></pre>

> To fully update our status, we’ll need to list all child jobs in this namespace that belong to this CronJob. Similarly to Get, we can use the List method to list the child jobs. Notice that we use variadic options to set the namespace and field match (which is actually an index lookup that we set up below).

<pre><code class="go">
    var childJobs kbatch.JobList
    if err := r.List(ctx, &childJobs, client.InNamespace(req.Namespace), client.MatchingFields{jobOwnerKey: req.Name}); err != nil {
        log.Error(err, "unable to list child Jobs")
        return ctrl.Result{}, err
    }
</code></pre>

> The reconciler fetches all jobs owned by the cronjob for the status. As our number of cronjobs increases, looking these up can become quite slow as we have to filter through all of them. For a more efficient lookup, these jobs will be indexed locally on the controller's name. A jobOwnerKey field is added to the cached job objects. This key references the owning controller and functions as the index. Later in this document we will configure the manager to actually index this field.
