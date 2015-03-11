# Job controller

## Abstract

First basic implementation proposal for a Job controller.
Several exiting issues were already created regarding that particular subject:
- [Distributed CRON jobs in k8s #2156](https://github.com/GoogleCloudPlatform/kubernetes/issues/2156)
- [Job Controller #1624](https://github.com/GoogleCloudPlatform/kubernetes/issues/1624)

Several features also already exist that could be used by external tools to trigger batch execution on one pod within k8 cluster.
- Create a standalone pod (not linked to any existing replication controllers)
- Execute a new process within a given container from a running pod.

## Motivation

The main goal is to provide a new controller type with basic features, that is able to periodically trigger the creation of a new pod (based on its associated pod template) to be scheduled on one available node and to track its outcome.
A time-based scheduling mechanism will be implemented first. Other possible job scheduling conditions (like for instance successfully completion of other scheduled jobs) could be introduced in latter stages. 

## Job controller basic definition

The new controller json definition for a basic implementation will have the following content:

```
{
	"apiVersion": "v1beta3",
	"kind": "JobController",
	"id": "myjob-controller",
	"desiredState": {
		"schedulePolicy": {
			"timeSpec": "R5/T01:00:00/PT01",
			"execTimeout": 100,
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

New introduced part is mainly the schedulePolicy struct, that in a first version can specify the time schedule (using iso 8601 notation), the maximal number of retries for a failing job run and a maximal execution time per pod run. Reaching this execution time-out should not lead to a restart attempt of the scheduled pod (job run will be reported with a failed status)

Regarding restart policy, it could come in handy to allow failing containers within a running pod to be restarted a limited number of time in case of execution failure. The OnFailure restart policy defined at pod spec level can be extended to carry that new field, knowing that the restart count for a given container is already available and thus could be used by the kubelet to take this maximal restart field into account.

Job controller has the responsibility to advertise the pod completion status (success or failure) using events, and to delete it from the pods registry. Collecting the standard output/error of pod's containers is not covered by this design (a common solution for containers started by any controller would be needed)

## More details on job controller:

The following API objects are introduced for this new job controller:

```
type SchedulePolicy struct {
	// String containing the iso 8601 time scheduling 
	TimeSpec string `json:"timeSpec"`
	
	// Allow concurrent job to be started (case of new job schedule time reached,
	// while other previously started jobs are still running)
	AllowConcurrent bool `json:"allowConcurrent"`
	
	// Maximal execution time permitted for a scheduled job/pod (specified in seconds)
	ExecTimeout int `json:"execTimeout"`
	
	// Maximal restart number for a failing scheduled job/pod
	MaxRestart int `json:"maxRestart"`
	
	// If AllowConcurrent is false, only start the latest from a range of pending jobs 
	// waiting for the current executing one to complete (default value is true)
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

The job controller will on a repetitive basis perform the following actions:

* If a pod tracked by this controller has completed, a event is raised with its final status (success/failure) and the pod is deleted from registry.
* If a pod is still running but has reached its execution time-out (if specified), the pod is stopped and a failure event dispatched.
* If a new pod needs to be started:
	* If there are no running pods (managed by the controller) or the AllowConcurrent field is set to true, a new pod is started (added to the list of pods in registry).
	* If there are still some running pods (managed by the controller) and the AllowConcurrent is false, nothing is done.
* If the number of pods that should have been scheduled between LastScheduledTime and current time is greater than one (case of a schedule policy preventing concurrent runs), LastScheduledTime will be updated with the schedule time of the first pod in the range if SkipOutdated is set to false, or the last one otherwise.


## Possible shortcomings

The previous design would work if all containers started in a pod scheduled by this job controller have a finite execution time. However, we might have the case of started containers with unlimited lifetime (for instance, deamons providing services for the job container to perform its work). In that case, the started pod will remain in a running state (and possibly only stopped once its execution timeout is reached).
We can somehow define the notion of leading container within a pod managed by a job controller, to be able to restrict the lifetime of the whole pod to this single container. 
Once the container exits, the whole pod is (gracefully) stopped and the final status of the leading container is given to the whole pod. This leading container could be specified with a new bool attribute at container level in the pod template definition.







