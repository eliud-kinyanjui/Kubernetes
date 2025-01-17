![](https://gaforgithub.azurewebsites.net/api?repo=CKAD-exercises/multi_container&empty)
## Multi-container Pods (10%)
## Practice questions based on these concepts

* Understand Multi-container pod design patterns (eg: ambassador, adaptor, sidecar)
### Questions
### Create a Pod with two containers, both with image busybox and command "echo hello; sleep 3600". Connect to the second container and run 'ls'

<details><summary>show</summary>
<p>

Easiest way to do it is create a pod with a single container and save its definition in a YAML file:

```bash
kubectl run busybox --image=busybox --restart=Never -o yaml --dry-run=client -- /bin/sh -c 'echo hello;sleep 3600' > pod.yaml
vi pod.yaml
```

Copy/paste the container related values, so your final YAML should contain the following two containers (make sure those containers have a different name):

```YAML
containers:
  - args:
    - /bin/sh
    - -c
    - echo hello;sleep 3600
    image: busybox
    imagePullPolicy: IfNotPresent
    name: busybox
    resources: {}
  - args:
    - /bin/sh
    - -c
    - echo hello;sleep 3600
    image: busybox
    name: busybox2
```

```bash
kubectl create -f pod.yaml
# Connect to the busybox2 container within the pod
kubectl exec -it busybox -c busybox2 -- /bin/sh
ls
exit

# or you can do the above with just an one-liner
kubectl exec -it busybox -c busybox2 -- ls

# you can do some cleanup
kubectl delete po busybox
```

</p>
</details>

### Create pod with nginx container exposed at port 80. Add a busybox init container which downloads a page using "wget -O /work-dir/index.html http://neverssl.com/online". Make a volume of type emptyDir and mount it in both containers. For the nginx container, mount it on "/usr/share/nginx/html" and for the initcontainer, mount it on "/work-dir". When done, get the IP of the created pod and create a busybox pod and run "wget -O- IP"

<details><summary>show</summary>
<p>

Easiest way to do it is create a pod with a single container and save its definition in a YAML file:

```bash
kubectl run web --image=nginx --restart=Never --port=80 --dry-run=client -o yaml > pod-init.yaml
```

Copy/paste the container related values, so your final YAML should contain the volume and the initContainer:

Volume:

```YAML
containers:
  - image: nginx
...
    volumeMounts:
    - name: vol
      mountPath: /usr/share/nginx/html
  volumes:
  - name: vol
    emptyDir: {}
```

initContainer:

```YAML
...
initContainers:
- args:
  - /bin/sh
  - -c
  - wget -O /work-dir/index.html http://neverssl.com/online
  image: busybox
  name: box
  volumeMounts:
  - name: vol
    mountPath: /work-dir
```

In total you get:

```YAML

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: box
  name: box
spec:
  initContainers: #
  - args: #
    - /bin/sh #
    - -c #
    - wget -O /work-dir/index.html http://neverssl.com/online #
    image: busybox #
    name: box #
    volumeMounts: #
    - name: vol #
      mountPath: /work-dir #
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    volumeMounts: #
    - name: vol #
      mountPath: /usr/share/nginx/html #
  volumes: #
  - name: vol #
    emptyDir: {} #
```

```bash
# Apply pod
kubectl apply -f pod-init.yaml

# Get IP
kubectl get po -o wide

# Execute wget
kubectl run box-test --image=busybox --restart=Never -it --rm -- /bin/sh -c "wget -O- IP"

# you can do some cleanup
kubectl delete po box
```

</p>
</details>

<details><summary>Create a Pod with three busy box containers with commands “ls; sleep 3600;”, “echo Hello World; sleep 3600;” and “echo this is the third container; sleep 3600” respectively and check the status</summary>
<p>

```
// first create single container pod with dry run flag
kubectl run busybox --image=busybox --restart=Never --dry-run -o yaml -- bin/sh -c "sleep 3600; ls" > multi-container.yaml

// edit the pod like below

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox
  name: busybox
spec:
  containers:
  - args:
    - bin/sh
    - -c
    - ls; sleep 3600
    image: busybox
    name: busybox1
    resources: {}
  - args:
    - bin/sh
    - -c
    - echo Hello world; sleep 3600
    image: busybox
    name: busybox2
    resources: {}
  - args:
    - bin/sh
    - -c
    - echo this is third container; sleep 3600
    image: busybox
    name: busybox3
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

// create it
kubectl create -f multi-container.yaml

kubectl get po busybox
```
</p>
</details>
   
<details><summary>Check the logs of each container that you just created</summary>
<p>
   
```
kubectl logs busybox -c busybox1
kubectl logs busybox -c busybox2
kubectl logs busybox -c busybox3
```
</p>
</details>

<details><summary>Check the previous logs of the second container busybox2 if any</summary>
<p>
   
```
kubectl logs busybox -c busybox2 --previous
```
</p>
</details>

<details><summary>Run command ls in the third container busybox3 of the above pod</summary>
<p>
   
```
kubectl exec busybox -c busybox3 -- ls
```
</p>
</details>

<details><summary>Show metrics of the above pod containers and puts them into the file.log and verify</summary>
<p>
   
```
kubectl top pod busybox --containers

// putting them into file
kubectl top pod busybox --containers > file.log
cat file.log
```
</p>
</details>


<details><summary>Create a Pod with main container busybox and which executes this “while true; do echo ‘Hi I am from Main container’ >> /var/log/index.html; sleep 5; done” and with sidecar container with nginx image which exposes on port 80. Use emptyDir Volume and mount this volume on path /var/log for busybox and on path /usr/share/nginx/html for nginx container. Verify both containers are running.</summary>
<p>
   
```
// create an initial yaml file with this
kubectl run multi-cont-pod --image=busbox --restart=Never --dry-run -o yaml > multi-container.yaml

// edit the yml as below and create it
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: multi-cont-pod
  name: multi-cont-pod
spec:
  volumes:
  - name: var-logs
    emptyDir: {}
  containers:
  - image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo 'Hi I am from Main container' >> /var/log/index.html; sleep 5;done"]
    name: main-container
    resources: {}
    volumeMounts:
    - name: var-logs
      mountPath: /var/log
  - image: nginx
    name: sidecar-container
    resources: {}
    ports:
      - containerPort: 80
    volumeMounts:
    - name: var-logs
      mountPath: /usr/share/nginx/html
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

kubectl create -f multi-container.yaml

kubectl get po multi-cont-pod
```
</p>
</details>


<details><summary>Exec into both containers and verify that main.txt exist and query the main.txt from sidecar container with curl localhost</summary>
<p>
   
```
// exec into main container
kubectl exec -it  multi-cont-pod -c main-container -- sh
cat /var/log/main.txt

// exec into sidecar container
kubectl exec -it  multi-cont-pod -c sidecar-container -- sh
cat /usr/share/nginx/html/index.html

// install curl and get default page
kubectl exec -it  multi-cont-pod -c sidecar-container -- sh
# apt-get update && apt-get install -y curl
# curl localhost
```
</p>
</details>


## Implementing the Adapter Pattern

The adapter pattern helps with providing a simplified, homogenized view of an application running within a container. For example, we could stand up another container that unifies the log output of the application container. As a result, other monitoring tools can rely on a standardized view of the log output without having to transform it into an expected format.

1. Create a new Pod in a YAML file named `adapter.yaml`. The Pod declares two containers. The container `app` uses the image `busybox` and runs the command `while true; do echo "$(date) | $(du -sh ~)" >> /var/logs/diskspace.txt; sleep 5; done;`. The adapter container `transformer` uses the image `busybox` and runs the command `sleep 20; while true; do while read LINE; do echo "$LINE" | cut -f2 -d"|" >> $(date +%Y-%m-%d-%H-%M-%S)-transformed.txt; done < /var/logs/diskspace.txt; sleep 20; done;` to strip the log output off the date for later consumption my a monitoring tool. Be aware that the logic does not handle corner cases (e.g. automatically deleting old entries) and would look different in production systems.
2. Before creating the Pod, define an `emptyDir` volume. Mount the volume in both containers with the path `/var/logs`.
3. Create the Pod, log into the container `transformer`. The current directory should continuously write a new file every 20 seconds.

<details><summary>Show Solution</summary>
<p>

```bash
kubectl run adapter --image=busybox --restart=Never -o yaml --dry-run -- /bin/sh -c 'while true; do echo "$(date) | $(du -sh ~)" >> /var/logs/diskspace.txt; sleep 5; done;' > adapter.yaml
```
The final Pod YAML file should look something like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: adapter
spec:
  volumes:
    - name: config-volume
      emptyDir: {}
  containers:
  - args:
    - /bin/sh
    - -c
    - 'while true; do echo "$(date) | $(du -sh ~)" >> /var/logs/diskspace.txt; sleep 5; done;'
    image: busybox
    name: app
    volumeMounts:
      - name: config-volume
        mountPath: /var/logs
    resources: {}
  - image: busybox
    name: transformer
    args:
    - /bin/sh
    - -c
    - 'sleep 20; while true; do while read LINE; do echo "$LINE" | cut -f2 -d"|" >> $(date +%Y-%m-%d-%H-%M-%S)-transformed.txt; done < /var/logs/diskspace.txt; sleep 20; done;'
    volumeMounts:
      - name: config-volume
        mountPath: /var/logs
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```


```bash
$ kubectl exec adapter --container=transformer -it -- /bin/sh
/ # ls -l
-rw-r--r--    1 root     root           205 May 12 20:43 2019-05-12-20-43-32-transformed.txt
-rw-r--r--    1 root     root           369 May 12 20:43 2019-05-12-20-43-52-transformed.txt
...
/ # cat 2019-05-12-20-43-52-transformed.txt
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
/ # exit
```

</p>
</details>
