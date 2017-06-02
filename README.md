## Getting Into Kubernetes RBAC For User Authorization

**Note -- this uses a method contrary to the kubernetes docs, namely using a serviceccount for a user, but it gets me what I want -- a kubectl user with limited access...**


This assumes:

	- you have a kubernetes cluster > 1.6 with RBAC enabled
	- you're using the admin user role to run `kubectl`.  (This would be the default created by your script.)
	- you want to add more kubectl users but want to limit their access to the k8s API

### Before or After, some reading

Take a spin through [the official docs](https://kubernetes.io/docs/admin/authorization/rbac/) to get a high level perspective on the options.  We're going to try to give some concrete and directed examples here.

### Intro to existing RBAC roles:

As silly as it sounds, running this command will help you grok what you're looking at with RBAC roles:

```
kubectl get clusterroles
```

Those are a list of clusterroles that are already present on your cluster.  You can use them for authorization directly by referencing them (which we do below) or you can use them as a basis to create your own clusterroles (or roles).

### What sort of resources, verbs, groups are in a role

Take a look at any of the bootstrapped RBAC clusterroles (which are any of them listed from the `get clusterroles` command from the previous section  without system: prefixed)

```
kubectl get clusterrole admin -o=yaml
```

Do the same for the other basic ones: `admin, edit, view`.  That should give you an idea of what you'll want to do to corral your users.

If you don't understand what you're looking at, you may want to peruse [the official docs](https://kubernetes.io/docs/admin/authorization/rbac/).

### Wow.  So?

Heh, aren't we spunky?

Here's what we want to do -- we want to be able to create multiple users with varying access to cluster resources.  We're going to do that using the RBAC stuff above plus a bit of kubectl commands googled and swiped from a gist.

#### Task: create an edit user that is constrained to a single namespace 

Like it says -- create a user that can change things in a single namespace.  We want to be able to use kubectl to do this stuff (it's no mere serviceaccount, my friends).  We also don't have any of those fancy authN plugins so we'll have to otherwise make due.

  - namespace exists or created: nginx-app
  - user exists or created: nginx-edit
  - rbac clusterrole to use: edit

We're going to use an existing RBAC (bootstrapped) role.  As you gain more experience with RBAC and your cluster and user needs, you may want to dump out the role and whittle it down to as few grants as possible.

Run this command to take care of it for you:

```
./gen-user-kubeconfig.sh nginx-edit nginx-app
```

That creates the namespace, the user, and then generates the kubeconfig file as `k8s-<namespace>-<user>-conf`.  You can't do anything yet as you haven't granted your new sa any roles.  Now we must RBAC!

The role already exists (as a clusterrole) so we do NOT need to create and fire off a yaml to create that on the cluster.  What's left?  We have to bind the role to our user (and in this case to a particular namespace).  We need a [rolebinding](https://kubernetes.io/docs/admin/authorization/rbac/#rolebinding-and-clusterrolebinding).  Keep using your admin user to create this.  

We'll switch over to the `nginx-edit` user soon enough.


Create file `rolebinding-nginx-edit.yaml` with the following contents:

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: nginx-edit-namespace
  namespace: nginx-app
subjects:
- kind: ServiceAccount
  name: nginx-edit
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

You can see what we did there.  We just glued some stuff together:

```
sa:nginx-app:nginx-edit <-connected-> role:edit
```


Then apply it to the cluster:

```
kubectl create -f rolebinding-nginx-edit.yaml
```

Now you can properly `kubectl` as your new user.

#### That's it.  Now use it.

Lets spin up a nginx deployment in our new nginx-app namespace.  Create `nginx-deploy.yaml` with the following contents:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
        status: live
    spec:
      containers:
      - name: nginx
        image: nginx:1.13.1
        ports:
        - containerPort: 80

```

We can use our kubeconfig to do this by running the following command:

```
KUBECONFIG=k8s-nginx-app-nginx-edit-conf kubectl create -f nginx-deploy.yaml
```

You should see `deployment "nginx-deployment" created`.  We can confirm it by querying the cluster with:

```
KUBECONFIG=k8s-nginx-app-nginx-edit-conf kubectl get po, deploy
```

which should show you something like this:

```
NAME                                  READY     STATUS    RESTARTS   AGE
po/nginx-deployment-107352269-1rh9r   1/1       Running   0          1m
po/nginx-deployment-107352269-p54t5   1/1       Running   0          1m
po/nginx-deployment-107352269-vmfpw   1/1       Running   0          1m

NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/nginx-deployment   3         3         3            3           1m
```

That's it.  

#### But you didn't check if it was locked down

That's true.  So lets just run the same command in the `default` namespace:

```
$ KUBECONFIG=k8s-nginx-app-nginx-edit-conf kubectl get po,deploy --namespace=default
Error from server (Forbidden): User "system:serviceaccount:nginx-app:nginx-edit" cannot list pods in the namespace "default". (get pods)
Error from server (Forbidden): User "system:serviceaccount:nginx-app:nginx-edit" cannot list deployments.extensions inthe namespace "default". (get deployments.extensions)
```

Boom!


## Other Notes

  - [K8s Docs Listing](https://kubernetes.io/docs/admin/service-accounts-admin/) difference between user and service accounts

  - thanks to so0k for [this gist](https://gist.github.com/so0k/8fad3b1639b3d70cd841703fda67f16b) that was the basis of my own script.

