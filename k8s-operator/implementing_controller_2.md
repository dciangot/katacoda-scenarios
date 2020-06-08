## Updating Job statuses

>Once we have all the jobs we own, we’ll split them into active, successful, and failed jobs, keeping track of the most recent run so that we can record it in status. Remember, status should be able to be reconstituted from the state of the world, so it’s generally not a good idea to read from the status of the root object. Instead, you should reconstruct it every run. That’s what we’ll do here.

>We can check if a job is “finished” and whether it succeeded or failed using status conditions. We’ll put that logic in a helper to make our code cleaner.

<pre><code class="go">
    // find the active list of jobs
    var activeJobs []*kbatch.Job
    var successfulJobs []*kbatch.Job
    var failedJobs []*kbatch.Job
    var mostRecentTime *time.Time // find the last run so we can update the status
    isJobFinished := func(job *kbatch.Job) (bool, kbatch.JobConditionType) {
        for _, c := range job.Status.Conditions {
            if (c.Type == kbatch.JobComplete || c.Type == kbatch.JobFailed) && c.Status == corev1.ConditionTrue {
                return true, c.Type
            }
        }

        return false, ""
    }

    getScheduledTimeForJob := func(job *kbatch.Job) (*time.Time, error) {
        timeRaw := job.Annotations[scheduledTimeAnnotation]
        if len(timeRaw) == 0 {
            return nil, nil
        }

        timeParsed, err := time.Parse(time.RFC3339, timeRaw)
        if err != nil {
            return nil, err
        }
        return &timeParsed, nil
    }

    for i, job := range childJobs.Items {
        _, finishedType := isJobFinished(&job)
        switch finishedType {
        case "": // ongoing
            activeJobs = append(activeJobs, &childJobs.Items[i])
        case kbatch.JobFailed:
            failedJobs = append(failedJobs, &childJobs.Items[i])
        case kbatch.JobComplete:
            successfulJobs = append(successfulJobs, &childJobs.Items[i])
        }

        // We'll store the launch time in an annotation, so we'll reconstitute that from
        // the active jobs themselves.
        scheduledTimeForJob, err := getScheduledTimeForJob(&job)
        if err != nil {
            log.Error(err, "unable to parse schedule time for child job", "job", &job)
            continue
        }
        if scheduledTimeForJob != nil {
            if mostRecentTime == nil {
                mostRecentTime = scheduledTimeForJob
            } else if mostRecentTime.Before(*scheduledTimeForJob) {
                mostRecentTime = scheduledTimeForJob
            }
        }
    }

    if mostRecentTime != nil {
        cronJob.Status.LastScheduleTime = &metav1.Time{Time: *mostRecentTime}
    } else {
        cronJob.Status.LastScheduleTime = nil
    }
    cronJob.Status.Active = nil
    for _, activeJob := range activeJobs {
        jobRef, err := ref.GetReference(r.Scheme, activeJob)
        if err != nil {
            log.Error(err, "unable to make reference to active job", "job", activeJob)
            continue
        }
        cronJob.Status.Active = append(cronJob.Status.Active, *jobRef)
    }

    log.V(1).Info("job count", "active jobs", len(activeJobs), "successful jobs", len(successfulJobs), "failed jobs",len(failedJobs))

    if err := r.Status().Update(ctx, &cronJob); err != nil {
        log.Error(err, "unable to update CronJob status")
        return ctrl.Result{}, err
    }
</code></pre>


## Get the next scheduled run

>We’ll calculate the next scheduled time using our helpful cron library. We’ll start calculating appropriate times from our last run, or the creation of the CronJob if we can’t find a last run.

>If there are too many missed runs and we don’t have any deadlines set, we’ll bail so that we don’t cause issues on controller restarts or wedges.

>Otherwise, we’ll just return the missed runs (of which we’ll just use the latest), and the next run, so that we can know when it’s time to reconcile again.

<pre><code class="go">
    getNextSchedule := func(cronJob *batch.CronJob, now time.Time) (lastMissed time.Time, next time.Time, err error) {
        sched, err := cron.ParseStandard(cronJob.Spec.Schedule)
        if err != nil {
            return time.Time{}, time.Time{}, fmt.Errorf("Unparseable schedule %q: %v", cronJob.Spec.Schedule, err)
        }

        // for optimization purposes, cheat a bit and start from our last observed run time
        // we could reconstitute this here, but there's not much point, since we've
        // just updated it.
        var earliestTime time.Time
        if cronJob.Status.LastScheduleTime != nil {
            earliestTime = cronJob.Status.LastScheduleTime.Time
        } else {
            earliestTime = cronJob.ObjectMeta.CreationTimestamp.Time
        }
        if cronJob.Spec.StartingDeadlineSeconds != nil {
            // controller is not going to schedule anything below this point
            schedulingDeadline := now.Add(-time.Second * time.Duration(*cronJob.Spec.StartingDeadlineSeconds))

            if schedulingDeadline.After(earliestTime) {
                earliestTime = schedulingDeadline
            }
        }
        if earliestTime.After(now) {
            return time.Time{}, sched.Next(now), nil
        }

        starts := 0
        for t := sched.Next(earliestTime); !t.After(now); t = sched.Next(t) {
            lastMissed = t
            // An object might miss several starts. For example, if
            // controller gets wedged on Friday at 5:01pm when everyone has
            // gone home, and someone comes in on Tuesday AM and discovers
            // the problem and restarts the controller, then all the hourly
            // jobs, more than 80 of them for one hourly scheduledJob, should
            // all start running with no further intervention (if the scheduledJob
            // allows concurrency and late starts).
            //
            // However, if there is a bug somewhere, or incorrect clock
            // on controller's server or apiservers (for setting creationTimestamp)
            // then there could be so many missed start times (it could be off
            // by decades or more), that it would eat up all the CPU and memory
            // of this controller. In that case, we want to not try to list
            // all the missed start times.
            starts++
            if starts > 100 {
                // We can't get the most recent times so just return an empty slice
                return time.Time{}, time.Time{}, fmt.Errorf("Too many missed start times (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.")
            }
        }
        return lastMissed, sched.Next(now), nil
    }
    // figure out the next times that we need to create
    // jobs at (or anything we missed).
    missedRun, nextRun, err := getNextSchedule(&cronJob, r.Now())
    if err != nil {
        log.Error(err, "unable to figure out CronJob schedule")
        // we don't really care about requeuing until we get an update that
        // fixes the schedule, so don't return an error
        return ctrl.Result{}, nil
    }
</code></pre>

> We’ll prep our eventual request to requeue until the next job, and then figure out if we actually need to run.

<pre><code class="go">
    scheduledResult := ctrl.Result{RequeueAfter: nextRun.Sub(r.Now())} // save this so we can re-use it elsewhere
    log = log.WithValues("now", r.Now(), "next run", nextRun)
</code></pre>

## Run a new job if it’s on schedule, not past the deadline, and not blocked by our concurrency policy

> If we’ve missed a run, and we’re still within the deadline to start it, we’ll need to run a job.

<pre><code class="go">
    if missedRun.IsZero() {
        log.V(1).Info("no upcoming scheduled times, sleeping until next")
        return scheduledResult, nil
    }

    // make sure we're not too late to start the run
    log = log.WithValues("current run", missedRun)
    tooLate := false
    if cronJob.Spec.StartingDeadlineSeconds != nil {
        tooLate = missedRun.Add(time.Duration(*cronJob.Spec.StartingDeadlineSeconds) * time.Second).Before(r.Now())
    }
    if tooLate {
        log.V(1).Info("missed starting deadline for last run, sleeping till next")
        // TODO(directxman12): events
        return scheduledResult, nil
    }


    // figure out how to run this job -- concurrency policy might forbid us from running
    // multiple at the same time...
    if cronJob.Spec.ConcurrencyPolicy == batch.ForbidConcurrent && len(activeJobs) > 0 {
        log.V(1).Info("concurrency policy blocks concurrent runs, skipping", "num active", len(activeJobs))
        return scheduledResult, nil
    }

    // ...or instruct us to replace existing ones...
    if cronJob.Spec.ConcurrencyPolicy == batch.ReplaceConcurrent {
        for _, activeJob := range activeJobs {
            // we don't care if the job was already deleted
            if err := r.Delete(ctx, activeJob, client.PropagationPolicy(metav1.DeletePropagationBackground)); client.IgnoreNotFound(err) != nil {
                log.Error(err, "unable to delete active job", "job", activeJob)
                return ctrl.Result{}, err
            }
        }
    }
</code></pre>

And finally let's create our job object

<pre><code class="go">
constructJobForCronJob := func(cronJob *batch.CronJob, scheduledTime time.Time) (*kbatch.Job, error) {
        // We want job names for a given nominal start time to have a deterministic name to avoid the same job being created twice
        name := fmt.Sprintf("%s-%d", cronJob.Name, scheduledTime.Unix())

        job := &kbatch.Job{
            ObjectMeta: metav1.ObjectMeta{
                Labels:      make(map[string]string),
                Annotations: make(map[string]string),
                Name:        name,
                Namespace:   cronJob.Namespace,
            },
            Spec: *cronJob.Spec.JobTemplate.Spec.DeepCopy(),
        }
        for k, v := range cronJob.Spec.JobTemplate.Annotations {
            job.Annotations[k] = v
        }
        job.Annotations[scheduledTimeAnnotation] = scheduledTime.Format(time.RFC3339)
        for k, v := range cronJob.Spec.JobTemplate.Labels {
            job.Labels[k] = v
        }
        if err := ctrl.SetControllerReference(cronJob, job, r.Scheme); err != nil {
            return nil, err
        }

        return job, nil
    }
    // actually make the job...
    job, err := constructJobForCronJob(&cronJob, missedRun)
    if err != nil {
        log.Error(err, "unable to construct job from template")
        // don't bother requeuing until we get a change to the spec
        return scheduledResult, nil
    }

    // ...and create it on the cluster
    if err := r.Create(ctx, job); err != nil {
        log.Error(err, "unable to create Job for CronJob", "job", job)
        return ctrl.Result{}, err
    }

    log.V(1).Info("created Job for CronJob run", "job", job)
    return scheduledResult, nil
</code></pre>

## Setup

Finally, we’ll update our setup. In order to allow our reconciler to quickly look up Jobs by their owner, we’ll need an index. We declare an index key that we can later use with the client as a pseudo-field name, and then describe how to extract the indexed value from the Job object. The indexer will automatically take care of namespaces for us, so we just have to extract the owner name if the Job has a CronJob owner.

Additionally, we’ll inform the manager that this controller owns some Jobs, so that it will automatically call Reconcile on the underlying CronJob when a Job changes, is deleted, etc.

<pre><code class="go">
var (
    jobOwnerKey = ".metadata.controller"
    apiGVStr    = batch.GroupVersion.String()
)

func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // set up a real clock, since we're not in a test
    if r.Clock == nil {
        r.Clock = realClock{}
    }

    if err := mgr.GetFieldIndexer().IndexField(&kbatch.Job{}, jobOwnerKey, func(rawObj runtime.Object) []string {
        // grab the job object, extract the owner...
        job := rawObj.(*kbatch.Job)
        owner := metav1.GetControllerOf(job)
        if owner == nil {
            return nil
        }
        // ...make sure it's a CronJob...
        if owner.APIVersion != apiGVStr || owner.Kind != "CronJob" {
            return nil
        }

        // ...and if so, return it
        return []string{owner.Name}
    }); err != nil {
        return err
    }

    return ctrl.NewControllerManagedBy(mgr).
        For(&batch.CronJob{}).
        Owns(&kbatch.Job{}).
        Complete(r)
}
</code></pre>