
# ⏰ Kubernetes CronJob Guide with Examples

A **CronJob** in Kubernetes runs **scheduled jobs** similarly to cron tasks in Linux. It is ideal for running batch processes like backups, reports, cleanups, etc.

---

## 📘 Basic Structure of a CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/5 * * * *"  # Every 5 minutes
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
            - date; echo Hello from CronJob!
          restartPolicy: OnFailure
```

---

## 🧠 Key Fields

| Field             | Description                                |
|------------------|--------------------------------------------|
| `schedule`       | Cron format for job frequency              |
| `jobTemplate`    | The Job template that is created per run   |
| `restartPolicy`  | Must be `OnFailure` or `Never`             |
| `concurrencyPolicy` | How jobs are run if overlaps occur (`Allow`, `Forbid`, `Replace`) |
| `startingDeadlineSeconds` | Grace period for a missed schedule |
| `successfulJobsHistoryLimit` | How many successful job pods to keep |
| `failedJobsHistoryLimit`     | How many failed jobs to keep     |

---

## 🛠 Example: Run Every Minute

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: one-minute-job
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: quick-task
            image: busybox
            args:
            - /bin/sh
            - -c
            - echo Running at $(date)
          restartPolicy: OnFailure
```

---

## 📉 Example: Cleanup Job (Once a Day)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-cleanup
spec:
  schedule: "0 1 * * *"  # Every day at 1:00 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleaner
            image: alpine
            command: ["/bin/sh", "-c", "rm -rf /tmp/*"]
          restartPolicy: OnFailure
```

---

## ⚙️ Apply and Test the CronJob

```bash
kubectl apply -f cronjob.yaml
kubectl get cronjob
kubectl get jobs --watch
kubectl logs job/<job-name>
```

To trigger it manually (for testing):
```bash
kubectl create job --from=cronjob/hello-cron hello-now
```

---

## 🧹 Clean Up

```bash
kubectl delete cronjob hello-cron
```

---

✅ *CronJobs are powerful for automating periodic tasks in Kubernetes—use them wisely with resource limits and proper failure handling.*
