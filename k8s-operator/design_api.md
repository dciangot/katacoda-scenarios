## Design CronJob API

From: https://book.kubebuilder.io/cronjob-tutorial/api-design.html

>Fundamentally a CronJob needs the following pieces:

>- A schedule (the cron in CronJob)
>- A template for the Job to run (the job in CronJob)

> We’ll also want a few extras, which will make our users’ lives easier:
>- A deadline for starting jobs (if we miss this deadline, we’ll just wait till the next scheduled time)
>- What to do if multiple jobs would run at once (do we wait? stop the old one? run both?)
>- A way to pause the running of a CronJob, in case something’s wrong with it
>- Limits on old job history
>Remember, since we never read our own status, we need to have some other way to keep track of whether a job has run. We can use at least one old job to do this.

Fill up you resource specs on the `cronjob_types.go`:
```go
// CronJobSpec defines the desired state of CronJob
type CronJobSpec struct {
    // +kubebuilder:validation:MinLength=0

    // The schedule in Cron format, see https://en.wikipedia.org/wiki/Cron.
    Schedule string `json:"schedule"`

    // +kubebuilder:validation:Minimum=0

    // Optional deadline in seconds for starting the job if it misses scheduled
    // time for any reason.  Missed jobs executions will be counted as failed ones.
    // +optional
    StartingDeadlineSeconds *int64 `json:"startingDeadlineSeconds,omitempty"`

    // Specifies how to treat concurrent executions of a Job.
    // Valid values are:
    // - "Allow" (default): allows CronJobs to run concurrently;
    // - "Forbid": forbids concurrent runs, skipping next run if previous run hasn't finished yet;
    // - "Replace": cancels currently running job and replaces it with a new one
    // +optional
    ConcurrencyPolicy ConcurrencyPolicy `json:"concurrencyPolicy,omitempty"`

    // This flag tells the controller to suspend subsequent executions, it does
    // not apply to already started executions.  Defaults to false.
    // +optional
    Suspend *bool `json:"suspend,omitempty"`

    // Specifies the job that will be created when executing a CronJob.
    JobTemplate batchv1beta1.JobTemplateSpec `json:"jobTemplate"`

    // +kubebuilder:validation:Minimum=0

    // The number of successful finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    SuccessfulJobsHistoryLimit *int32 `json:"successfulJobsHistoryLimit,omitempty"`

    // +kubebuilder:validation:Minimum=0

    // The number of failed finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    FailedJobsHistoryLimit *int32 `json:"failedJobsHistoryLimit,omitempty"`
}
```

>Next, let’s design our status, which holds observed state. It contains any information we want users or other controllers to be able to easily obtain.

>We’ll keep a list of actively running jobs, as well as the last time that we successfully ran our job. Notice that we use metav1.Time instead of time.Time to get the stable serialization, as mentioned above

```go
// CronJobStatus defines the observed state of CronJob
type CronJobStatus struct {
    // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
    // Important: Run "make" to regenerate code after modifying this file

    // A list of pointers to currently running jobs.
    // +optional
    Active []corev1.ObjectReference `json:"active,omitempty"`

    // Information when was the last time the job was successfully scheduled.
    // +optional
    LastScheduleTime *metav1.Time `json:"lastScheduleTime,omitempty"`
}
```