# Job controller

## Abstract

First basic implementation proposal for a Job controller.
Several exiting issues were already created regarding that particular subject:
- [Distributed CRON jobs in k8s #2156](https://github.com/GoogleCloudPlatform/kubernetes/issues/2156)
- [Job Controller #1624](https://github.com/GoogleCloudPlatform/kubernetes/issues/1624)

A need for a new controller is also mentioned in [here](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/pod-states.md#controllers-and-restartpolicy)

Several features also already exist that could be used by external tools to trigger batch execution on one pod within k8 cluster.
- Create a standalone pod (not linked to any existing replication controllers)
- Execute a new process within a given container from a running pod.

## Motivation

The main goal is to provide a new controller type with the following characteristics:
* Time initiated creation of a pod.
* Pod creation schedule (ISO8601) included in the controller definition.

Both of the above will be handled by single definition (see http://en.wikipedia.org/wiki/ISO_8601).
As with this standard you're defining the starting point and repeating intervals.

* (Future evolution) Track outcome of the pod and apply e.g. restart policy in case of failure.
* (Future evolution) Trigger jobs based on success/failure of others.

Since k8s already has 2 unused restart policies (Never and OnFailure) I've proposed to use
them here, where:
* `Never` - would allow creating run once pod's
* `OnFailure` - would allow acting upon pod's failure/success
I'd name both in that proposal, besides in the proposal I don't think we need to show this
as future evolution. The proposal should be complete, the implementation will be done
in a couple of steps IMHO.

## Job controller basic definition

The new controller json definition for a basic implementation will have the following content:

```
{
	"apiVersion": "v1beta3",
	"kind": "JobController",    <-- that's debatable but maybe we could call it just Job?
	"id": "myjob-controller",
	"desiredState": {
		"schedulePolicy": {
			"timeSpec": "R5/T01:00:00/PT01",
			"execTimeout": 100,  <-- this will be part of Quota and Resource limiting @derekwaynecarr is already working on (can't find any specific issue, but I can get back with details if you want)
			"maxRestart" : 2
		},
		"selector": { "name": "myjob"},
		"podTemplate": {
			"desiredState": {
				"manifest": {
					"version": "v1beta1",
					"id": "myapp-job",
					"containers": [{
						"name": "job-container",
						"image": "app/job"
					}]
				}
			}
			"labels": {"name": "myjob"}
		}
	}
}
```

As said before I'd add here those on failure/success reactions, though I don't known
how it should look like yet ;) And one other very important thing: wheather this job
is allowed to run in parallel, meaning two copies of this job are or not allowed.

Compared to a ReplicationController, there is no replica count since only 1 pod will be created.  JobController introduces a schedulePolicy structure which specifies:
* The schedule (using ISO8601 notation) to apply for launching the pod.
* The max number of retries for a failing job.  A job is considered failed if it exits with a return code other than zero, or if the pod is considered failed (e.g. the node fails and the pod needs to be rescheduled on another node).
* The max execution time per pod run in seconds. Reaching this limit leads to the pod being pro-actively destroyed.

Regarding restart policy, it will be necessary to allow failing containers within a pod to be restarted a limited number of times in case of failure. The OnFailure restart policy defined at pod spec level can be extended to carry this information (the restart count for a given container is already available) and would be used by the kubelet to take this maximal restart field into account.

**TODO: Spec the above in more detail**

Job controller has the responsibility to advertise pod completion status (success or failure) through events, and to delete it from the pod registry. Collecting the standard output/error of pod's containers is not covered by this design (a common solution for containers started by any controller is needed).

## More details on the job controller:

The following API objects are introduced for the JobController:

```
type SchedulePolicy struct {
	// TimeSpec contains the schedule in ISO8601 format.
	TimeSpec string `json:"timeSpec"`

	// AllowConcurrent specified whether concurrent jobs may be started
	// (covers the case where new job schedule time is reached, while
	// other previously started jobs are still running)
    AllowConcurrent	bool `json:"allowConcurrent"`

	// ExecTimeout specifies the max amount of time a job is allowed to run (in seconds).
	ExecTimeout int `json:"execTimeout"`

	// MaxRestart specifies the max number of restarts for a failing scheduled pod.
	MaxRestart int `json:"maxRestart"`

	// SkipOutdated specifies that if AllowConcurrent is false, only the latest pending job
	// that is waiting for the currently executing one to complete must be started (default: true)
	SkipOutdated bool `json:"skipOutdated"`
}

type JobControllerSpec struct {
	// Scheduling policy spec
	SchedulePolicy SchedulePolicy `json:"schedulePolicy"`

	// Pod selector for that controller
	Selector map[string]string `json:"selector"`

	// Reference to stand alone PodTemplate
	TemplateRef *ObjectReference `json:"templateRef,omitempty"`

	// Embedded pod template
 	Template *PodTemplateSpec `json:"template,omitempty"`
}

type JobControllerStatus struct {
	// Time of the latest scheduled job/pod
	LastScheduledTime string `json:"lastScheduledTime"`
}

// JobControllerController represents the configuration of a job controller.
type JobController struct {
	TypeMeta   `json:",inline"`
	ObjectMeta `json:"metadata,omitempty"`

	// Spec defines the desired behavior of this job controller.
	Spec JobControllerSpec `json:"spec,omitempty"`

	// Status is the current status of this job controller.
	Status JobControllerStatus `json:"status,omitempty"`
}

// JobControllerList is a collection of job controllers.
type JobControllerList struct {
	TypeMeta `json:",inline"`
	ListMeta `json:"metadata,omitempty"`

	Items []JobController `json:"items"`
}
```

Current ReplicationManager will be reused and extended to provide a generic management for all controller types.

Not sure about that, I'd rather propose writing one from scratch.

The job controller will on a recurring basis perform the following actions:

* If a pod tracked by this controller has completed, an event is raised with its final status (success/failure) and the pod is deleted from registry.
* If a pod is still running, but has reached its execution time-out (if specified), the pod is stopped and a failure event dispatched.
* If a new pod needs to be started:
	* If there are no running pods (managed by the controller) or the AllowConcurrent field is set to true, a new pod is started (added to the list of pods in registry).
	* If there are still some running pods (managed by the controller) and AllowConcurrent is false, nothing is done.
* If the number of pods that should have been scheduled between LastScheduledTime and current time is greater than one (case of a schedule policy preventing concurrent runs), LastScheduledTime will be updated with the schedule time of the first pod in the range if SkipOutdated is set to false, or the last one otherwise.


## Possible shortcomings

The previous design would work if all containers started in a pod scheduled by this job controller have a finite execution time. However, we might have the case of containers with unlimited lifetime (for instance, daemons providing services for the job container to perform its work). In that case, the started pod will remain in a running state (and possibly only stop once its execution timeout is reached).
We can somehow define the notion of leading containers within a pod managed by a job controller, to be able to restrict the lifetime of the whole pod to this single container.
Once the container exits, the whole pod is (gracefully) stopped and the final status of the leading container is given to the whole pod. This leading container could be specified with a new bool attribute at container level in the pod template definition.
